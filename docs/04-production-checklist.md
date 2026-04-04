# 04 — Checklist de produção

Usar como gate antes de declarar a migração concluída.
Cada item tem uma razão de existir — não é burocracia.

---

## Healthchecks

Containers sem healthcheck são monitorados apenas pelo estado do processo (running/stopped).
Isso não detecta: aplicação que subiu mas não responde, banco que aceitou conexão mas
não executa queries, worker que parou de processar sem travar.

```yaml
services:
  api:
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/health"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 30s   # tempo para a aplicação inicializar antes de começar a checar

  postgres:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 20s

  redis:
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
```

- `start_period` é ignorado pelo restart policy — o container não é reiniciado
  durante o período inicial mesmo que healthcheck falhe. Use para aplicações
  com boot lento.
- `depends_on condition: service_healthy` na cadeia de dependências só funciona
  se o serviço dependido tem healthcheck configurado.
- O endpoint `/health` deve checar dependências críticas (banco acessível, cache acessível),
  não apenas retornar 200 incondicionalmente.

---

## Restart policies

Sem restart policy, containers derrubados por OOM ou crash ficam parados até
intervenção manual.

```yaml
services:
  api:
    restart: unless-stopped

  worker:
    restart: on-failure:5   # máximo 5 tentativas — evita loop infinito de crash

  postgres:
    restart: unless-stopped
```

| Policy | Comportamento | Quando usar |
|---|---|---|
| `no` | Nunca reinicia | Containers de debug |
| `on-failure[:N]` | Reinicia em erro, limite opcional | Workers, jobs |
| `unless-stopped` | Reinicia sempre, exceto se parado manualmente | Serviços de produção |
| `always` | Reinicia sempre, inclusive após `docker stop` | Quando `unless-stopped` não for suficiente |

`always` pode criar situações onde você faz `docker stop` para manutenção e o container
sobe sozinho em seguida. `unless-stopped` é mais seguro para operação manual.

---

## Limites de recursos

Containers sem limites competem por recursos do host sem controle. Um leak de memória
em um container pode derrubar todos os outros no mesmo host.

```yaml
services:
  api:
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 512M
        reservations:
          cpus: "0.5"
          memory: 256M

  postgres:
    deploy:
      resources:
        limits:
          cpus: "4.0"
          memory: 2G
        reservations:
          cpus: "1.0"
          memory: 1G
```

- `limits` definem o teto — o container não ultrapassa. Se ultrapassar memória,
  o kernel mata o processo (OOM kill).
- `reservations` garantem recursos mínimos — útil para scheduling em Swarm.
- Em Compose sem Swarm, `deploy.resources` é respeitado desde Docker Compose v2
  com o plugin. Verifique com `docker compose version`.

**Dimensionamento:**

Não chute. Colete métricas no legado por pelo menos uma semana em horário de pico:

```bash
# CPU e memória do processo legado
pidstat -u -r -p $(pgrep app_service) 1 60

# No container (após subir em homologação com carga real ou simulada)
docker stats --no-stream app-api-1
```

Defina o limite em ~150% do pico observado. Margem menor aumenta risco de OOM
em spike; margem maior desperdiça recursos.

---

## Logging

Containers devem escrever em stdout/stderr. O daemon Docker captura e rotaciona.

```yaml
services:
  api:
    logging:
      driver: "json-file"
      options:
        max-size: "50m"
        max-file: "5"
```

Sem `max-size` e `max-file`, os logs crescem sem limite até encher o disco.
O padrão do Docker é `json-file` sem rotação — isso é um problema em produção.
Ver pitfall #8 em [05-pitfalls.md](05-pitfalls.md).

**Log drivers alternativos:**

| Driver | Quando usar |
|---|---|
| `json-file` | Padrão, adequado para a maioria dos casos com rotação configurada |
| `journald` | Integração com systemd/journalctl no host |
| `syslog` | Envio para syslog centralizado |
| `fluentd` | Stack de logging centralizado com Fluentd/Fluentbit |
| `awslogs` | Containers no EC2/ECS enviando para CloudWatch |

Driver diferente de `json-file` e `journald` desabilita `docker logs` — você
não vai conseguir ver os logs via CLI diretamente no container.

**Configuração global no daemon (recomendado):**

```json
// /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "5"
  }
}
```

Isso aplica a todos os containers do host sem precisar configurar em cada Compose file.
Containers com configuração explícita sobrescrevem o padrão do daemon.

---

## Monitoramento

O que precisa estar em vigor antes de ir para produção:

**Métricas do container:**

```bash
# Verificar se o host exporta métricas via node_exporter + cAdvisor
docker run -d \
  --name cadvisor \
  --restart unless-stopped \
  -v /:/rootfs:ro \
  -v /var/run:/var/run:ro \
  -v /sys:/sys:ro \
  -v /var/lib/docker/:/var/lib/docker:ro \
  -p 127.0.0.1:8081:8080 \
  gcr.io/cadvisor/cadvisor:v0.47.2
```

cAdvisor expõe métricas de CPU, memória, rede e I/O por container no formato
Prometheus. Fundamental para detectar vazamentos de memória e container prestes
a atingir limite de recursos.

**Alertas mínimos que precisam existir:**

| Métrica | Threshold | Severidade |
|---|---|---|
| Container CPU > 90% por 5min | Alerta | Warning |
| Container memória > 85% do limite | Alerta imediato | Critical |
| Container em estado unhealthy | Alerta imediato | Critical |
| Container restartado N vezes em 1h | Investigar | Warning |
| Disco do host > 80% | Alerta | Warning |

**Checagem de container restartado:**

```bash
# Containers com restart count > 0
docker ps --format "table {{.Names}}\t{{.Status}}" | grep -v "Up [0-9]* [a-z]* " 
# Ou mais direto:
docker inspect $(docker ps -q) --format '{{.Name}} restarts={{.RestartCount}}' | \
  awk -F= '$2>0'
```

---

## Segurança mínima

Não é o escopo completo de segurança de containers — mas sem esses itens
a migração não está pronta para produção.

```yaml
services:
  api:
    user: "1000:1000"              # nunca root
    read_only: true                # filesystem imutável
    tmpfs:
      - /tmp:size=50m
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE           # somente se precisar de porta < 1024
    security_opt:
      - no-new-privileges:true
```

- `no-new-privileges` impede que processos dentro do container escalem privilégios
  via `setuid` binaries — mesmo que o container vaze para o host, o dano é limitado
- `read_only: true` exige que você mapeie todos os diretórios que precisam de escrita
  — isso força o inventário de paths de escrita que o assessment deveria ter feito
- Rodar como root dentro do container não é necessário na esmagadora maioria dos casos
  e é uma das causas mais comuns de container escape

---

## Sumário do checklist

```
Healthchecks
[ ] Todos os serviços têm healthcheck configurado
[ ] start_period adequado para tempo de boot
[ ] Endpoint /health verifica dependências críticas

Restart policies
[ ] Todos os serviços têm restart policy explícita
[ ] Workers com on-failure:N para evitar loop infinito

Limites de recursos
[ ] CPU e memória com limits configurados
[ ] Dimensionamento baseado em métricas reais, não chute

Logging
[ ] max-size e max-file configurados (ou via daemon.json)
[ ] Log driver adequado ao stack de observabilidade

Monitoramento
[ ] cAdvisor ou equivalente coletando métricas
[ ] Alertas para container unhealthy e OOM configurados

Segurança
[ ] Nenhum container rodando como root
[ ] cap_drop: ALL com cap_add seletivo
[ ] no-new-privileges habilitado
[ ] read_only onde aplicável
```
