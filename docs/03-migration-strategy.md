# 03 — Estratégia de migração

Migração sem downtime não é um truque — é um processo com etapas bem definidas.
O risco está nos atalhos: pular validação, não testar rollback, subestimar diferenças
de comportamento entre o ambiente legado e o containerizado.

---

## Premissas

Antes de iniciar:

- [ ] Assessment completo (doc 01) — sem surpresas conhecidas em aberto
- [ ] Arquitetura definida (doc 02) — volumes, redes, registry
- [ ] Ambiente de homologação equivalente ao de produção em comportamento de rede e storage
- [ ] Rollback documentado e testado em homologação
- [ ] Janela de cutover alinhada com stakeholders

---

## Ambiente paralelo

O padrão é construir o ambiente containerizado em paralelo ao sistema legado,
validar paridade de comportamento, e só então fazer o cutover.

```
legado:       [nginx]──[app]──[postgres]   (portas originais: 80, 443)
paralelo:  [nginx]──[app]──[postgres]      (portas diferentes: 8080, 8443)
```

Em produção, o ambiente paralelo roda no mesmo host ou em host separado atrás
do mesmo load balancer, mas sem receber tráfego real ainda.

**Construção do ambiente paralelo:**

```bash
# 1. Build da imagem a partir do código atual
docker build -t registry.infra.example.br/app/api:1.0.0 .
docker push registry.infra.example.br/app/api:1.0.0

# 2. Subir stack com portas alternativas para não colidir com legado
COMPOSE_PROJECT_NAME=app-parallel \
APP_PORT=8080 \
docker compose up -d

# 3. Verificar logs imediatamente
docker compose logs -f --tail=100
```

**Promoção gradual de tráfego (se possível):**

Se o load balancer suporta weighted routing (nginx upstream com weight, HAProxy,
AWS ALB weighted target groups), promova gradualmente:

```nginx
upstream app_backend {
    server 10.0.1.10:8080 weight=1;   # containerizado — 10% do tráfego
    server 10.0.1.10:3000 weight=9;   # legado — 90% do tráfego
}
```

Ajuste os pesos à medida que a confiança aumenta. Se algo der errado, a reversão
é uma mudança de configuração no load balancer — sem deploy, sem restart de serviço.

Nem todo ambiente tem essa capacidade. Se não tiver, vá direto para o cutover
janelado descrito abaixo.

---

## Migração de dados

Se o banco vai para dentro de um container (com volume nomeado), o procedimento
de migração inicial precisa ser documentado e testado.

**PostgreSQL:**

```bash
# No servidor legado: dump consistente
pg_dump -h 127.0.0.1 -U app_user -d app_db \
  --no-owner --no-acl -F c -f /tmp/app_db_$(date +%Y%m%d_%H%M).dump

# Transferir para o host de destino
scp /tmp/app_db_*.dump operador@10.0.1.20:/tmp/

# No host de destino: restaurar no volume do container
# O postgres precisa estar rodando para aceitar a restore
docker compose up -d postgres
docker exec -i app-postgres-1 pg_restore \
  -U app_user -d app_db -F c < /tmp/app_db_*.dump

# Verificar integridade básica
docker exec app-postgres-1 psql -U app_user -d app_db \
  -c "SELECT schemaname, tablename, n_live_tup FROM pg_stat_user_tables ORDER BY n_live_tup DESC;"
```

**Dados não-relacionais (uploads, arquivos):**

```bash
rsync -avz --progress /var/lib/app/uploads/ \
  operador@10.0.1.20:/mnt/app-uploads/

# Verificar contagem de arquivos
find /var/lib/app/uploads -type f | wc -l
# vs
find /mnt/app-uploads -type f | wc -l
```

---

## Cutover

O cutover é a janela em que o tráfego muda do legado para o containerizado.
A janela deve ser curta — o trabalho pesado foi feito na fase de validação.

**Sequência típica:**

```
T-5m    Confirmar que ambiente containerizado está saudável (healthchecks, logs)
T-5m    Confirmar que backup recente do banco está disponível
T-0     Parar aplicação legada (ou bloquear escrita, dependendo do sistema)
T+1m    Migração de dados delta (se aplicável)
T+2m    Atualizar load balancer / DNS para apontar para containers
T+5m    Validação de smoke tests (ver seção abaixo)
T+15m   Decisão: confirmar ou rollback
```

**Migração delta (banco de dados):**

Se o banco é compartilhado entre legado e container durante a fase de validação,
não há delta. Se são bancos separados e o legado continuou recebendo escritas
durante a fase de validação, você precisa sincronizar o delta:

```bash
# Gerar dump apenas das mudanças desde o snapshot inicial
# (Requer WAL archiving ou lógica de aplicação — não é trivial)
# Se não tiver mecanismo de delta, o cutover precisa incluir parada do legado
# antes de qualquer migração de dados
```

Sem mecanismo de delta, o modelo é: para o legado, migra os dados, sobe o container.
A janela de downtime é o tempo de migração.

**Atualização de DNS:**

```bash
# Se usando DNS interno com TTL baixo
# Reduzir TTL 24h antes do cutover
dig app.infra.example.br | grep TTL

# No cutover
nsupdate -k /etc/bind/Kapp.example.br.private <<EOF
server 10.0.1.1
update delete app.infra.example.br A
update add    app.infra.example.br 60 A 10.0.1.20
send
EOF
```

---

## Smoke tests pós-cutover

Não confie apenas em "parece estar funcionando". Tenha uma lista de validações
executáveis que cobrem o caminho crítico da aplicação.

```bash
#!/usr/bin/env bash
# smoke-test.sh — executar imediatamente após cutover

BASE_URL="https://app.infra.example.br"
FAIL=0

check() {
  local desc="$1"
  local url="$2"
  local expected="$3"

  status=$(curl -sk -o /dev/null -w "%{http_code}" "$url")
  if [ "$status" = "$expected" ]; then
    echo "OK  $desc ($status)"
  else
    echo "FAIL $desc — esperado $expected, obtido $status"
    FAIL=$((FAIL + 1))
  fi
}

check "Homepage"               "$BASE_URL/"                     "200"
check "Health endpoint"        "$BASE_URL/health"               "200"
check "API status"             "$BASE_URL/api/v1/status"        "200"
check "Auth redirect"          "$BASE_URL/admin"                "302"
check "Static assets"          "$BASE_URL/assets/app.js"       "200"
check "404 handling"           "$BASE_URL/rota-que-nao-existe"  "404"

# Verificar que não está servindo do cache stale
cache_header=$(curl -sk -I "$BASE_URL/" | grep -i "x-cache\|age:" | head -2)
echo "Cache headers: $cache_header"

if [ $FAIL -gt 0 ]; then
  echo ""
  echo "ATENÇÃO: $FAIL verificação(ões) falharam — avaliar rollback"
  exit 1
fi

echo ""
echo "Todos os checks passaram."
```

---

## Rollback

O plano de rollback precisa existir **antes** do cutover, não durante o incidente.

**Rollback de tráfego (DNS/LB):**

A reversão mais rápida — o legado continua rodando durante toda a janela de cutover.

```bash
# Reverter DNS para o IP do legado
nsupdate -k /etc/bind/Kapp.example.br.private <<EOF
server 10.0.1.1
update delete app.infra.example.br A
update add    app.infra.example.br 60 A 10.0.1.10
send
EOF

# Religar a aplicação legada se foi pausada
systemctl start app
```

**Rollback de dados (se banco foi migrado):**

Se escritas aconteceram no banco containerizado durante a janela:

```bash
# Dump do banco containerizado
docker exec app-postgres-1 pg_dump -U app_user app_db \
  -F c -f /tmp/rollback_$(date +%Y%m%d_%H%M).dump

# Copiar para o servidor legado
scp /tmp/rollback_*.dump operador@10.0.1.10:/tmp/

# Restaurar no banco legado
# (assumindo que banco legado foi preservado durante a janela)
pg_restore -U app_user -d app_db -F c /tmp/rollback_*.dump
```

**Definir critério de rollback antes do cutover:**

> "Se qualquer smoke test falhar, ou se houver erro 5xx acima de X% em 5 minutos,
> rollback imediato sem discussão."

Sem critério definido, decisões de rollback viram debate durante o incidente.

---

## Validação de paridade

Antes de descomissionar o legado, valide que o comportamento é equivalente.

**Comparação de logs:**

```bash
# Legado — sample de requests
grep -E '(GET|POST|PUT|DELETE)' /var/log/nginx/app_access.log | \
  awk '{print $7, $9}' | sort | uniq -c | sort -rn | head -20

# Container — mesmo período
docker logs app-nginx-1 --since 1h | \
  awk '{print $7, $9}' | sort | uniq -c | sort -rn | head -20
```

**Tempo de resposta:**

```bash
# Antes: medido no legado
ab -n 1000 -c 10 https://app-legacy.infra.example.br/api/v1/status

# Depois: medido no container
ab -n 1000 -c 10 https://app.infra.example.br/api/v1/status
```

**Erros de aplicação:**

Compare taxa de erro (5xx) entre legado e container no mesmo período de tráfego.
Um aumento de erro acima de 0.1% merece investigação antes de prosseguir.

---

## Descomissionamento do legado

Só após:

- [ ] Ambiente containerizado rodando em produção por pelo menos 72 horas sem incidente
- [ ] Smoke tests passando continuamente
- [ ] Métricas de erro equivalentes ao legado
- [ ] Backup do estado final do legado arquivado (banco, configs, logs recentes)
- [ ] Aprovação explícita do responsável técnico

```bash
# Arquivar config e estado antes de desligar
tar czf /backups/legacy-app-final-$(date +%Y%m%d).tar.gz \
  /etc/app /var/lib/app /var/log/app

# Parar e desabilitar serviço legado
systemctl stop app
systemctl disable app

# Manter o host por 30 dias antes de descomissionar
# (tempo suficiente para detectar qualquer comportamento não validado)
```
