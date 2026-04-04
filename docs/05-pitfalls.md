# 05 — Armadilhas de produção

Problemas que aparecem em produção e que documentação de tutorial não menciona.
Organizados por frequência e impacto — os primeiros aparecem mais, os últimos
causam mais dano quando aparecem.

---

## 1. Container não é seguro por natureza — ilusão de isolamento

O namespace de rede e PID isola o container do host em condições normais.
O que a maioria dos times não configura:

- Container rodando como root dentro do namespace — em caso de container escape
  via vulnerabilidade no runtime, o processo tem privilégios de root no host
- Capabilities não restringidas — `docker run` concede ~14 capabilities por padrão,
  incluindo `CAP_NET_ADMIN` e `CAP_SYS_CHROOT`
- `seccomp=unconfined` — desabilitado explicitamente "para debug" e nunca reabilitado
- `privileged: true` — dá acesso total ao host; equivale a rodar sem container

```bash
# Verificar o que está rodando com configuração insegura
docker inspect $(docker ps -q) --format \
  '{{.Name}} privileged={{.HostConfig.Privileged}} user={{.Config.User}}' | \
  awk '/privileged=true|user=$/'
```

**O risco prático:** exploração de aplicação vulnerável (RCE em app web) + container
mal configurado = comprometimento do host. Sem as configurações de segurança,
você tem isolamento de conveniência, não de segurança.

Configuração mínima para produção: `user` não-root, `cap_drop: ALL`,
`no-new-privileges: true`. Ver checklist completo em [04-production-checklist.md](04-production-checklist.md).

---

## 2. Container sem limites de recursos derruba o host inteiro

Um container sem `memory limit` pode consumir toda a RAM disponível no host.
O kernel vai matar processos — não necessariamente o do container causador.
Em hosts com vários containers, o mais provável é que o kernel mate os menores
antes de chegar ao que causou o problema.

```bash
# Ver containers sem limite de memória
docker inspect $(docker ps -q) \
  --format '{{.Name}} mem_limit={{.HostConfig.Memory}}' | \
  awk -F= '$2==0'
# Memory=0 significa sem limite
```

**Caso real:** worker de processamento de imagens sem limite de memória.
Upload de arquivo corrompido faz o processo tentar alocar 4GB em um host com 8GB.
O kernel mata o processo do banco de dados — que está no mesmo host — por ser
o maior consumidor de memória não-crítica no momento. Resultado: downtime total
que aparentemente não tem relação com o deploy do worker.

**OOM kill silencioso:**

```bash
# Ver histórico de OOM kills no host
dmesg -T | grep -i "oom\|killed process"
journalctl -k | grep -i "oom\|out of memory"
```

Sem monitoramento de OOM, você vai debugar um serviço reiniciando sem saber o motivo.

---

## 3. Container como servidor de 2005 — tudo num container só

PHP + nginx + MySQL + cron + Redis no mesmo container.
Tecnicamente funciona. Operacionalmente é um problema:

- Você não consegue escalar só o PHP sem escalar o banco
- Restart do container reinicia tudo — incluindo o banco no meio de uma transação
- Logs de quatro serviços diferentes misturados em stdout
- Atualização de qualquer componente exige rebuild da imagem inteira
- Healthcheck não distingue qual componente está com problema

Isso acontece quando o time containerizou um sistema legado sem refatorar a
arquitetura — empacotou o servidor físico dentro de um container.

**Quando é aceitável:** ambiente de desenvolvimento local onde conveniência
importa mais que operabilidade. Produção: cada serviço tem seu container.

**A refatoração que você vai fazer de qualquer jeito:** quando precisar escalar
o PHP porque o tráfego aumentou, você vai separar o banco do app. É melhor
fazer isso antes de ir para produção do que durante um incidente de capacidade.

---

## 4. Premature orchestration — Kubernetes antes da maturidade operacional

Times que containerizam tudo e, na mesma semana, decidem "vamos para Kubernetes
porque é o padrão da indústria".

**O que você ganha com Kubernetes:**
- Scheduling automático entre múltiplos nós
- Rolling updates e rollback declarativo
- Self-healing (restart automático, rescheduling)
- Service discovery e load balancing interno
- Abstrações para secret management, config, storage

**O que você paga:**
- etcd para manter o estado do cluster (precisa de backup, quorum, manutenção)
- Control plane: kube-apiserver, kube-scheduler, kube-controller-manager — cada um
  com seus próprios requisitos de disponibilidade e manutenção
- Curva de aprendizado de kubectl, RBAC, namespaces, NetworkPolicy, PodSecurityPolicy
- Debugging completamente diferente de Docker — `kubectl exec`, `kubectl logs`,
  `kubectl describe pod`, eventos de scheduling
- Se usando serviço gerenciado (EKS, GKE, AKS): custo do control plane +
  custo dos nós + custo de storage para volumes persistentes

**Quando Kubernetes faz sentido:**
- Múltiplos serviços independentes que precisam escalar de forma diferente
- Time com pelo menos uma pessoa que conhece Kubernetes em profundidade
- Workload que realmente se beneficia de scheduling multi-nó
- Orçamento para os custos adicionais de operação

**Quando não faz sentido:**
- 3-5 containers no mesmo host
- Time sem experiência prévia em Kubernetes
- Aplicação monolítica que não vai escalar horizontalmente

Para a maioria dos cenários de containerização inicial, Docker Compose (ou Swarm
para multi-nó simples) é a escolha correta. Kubernetes é a evolução quando você
tem os problemas que ele resolve.

---

## 5. Ausência de cache entre containers — carga desnecessária em CPU

Múltiplas instâncias do mesmo serviço, cada uma com cache em memória local.
Cada container tem seu próprio cache que vai sendo invalidado e reconstruído
independentemente. Em escala, isso significa:

- Queries ao banco sendo replicadas N vezes (uma por container)
- Processamento de dados sendo feito N vezes em paralelo para a mesma entrada
- Pico de carga no banco exatamente quando você escala horizontalmente — o inverso
  do que você queria

**O problema:**

```
request → container-1 (cache miss) → banco
request → container-2 (cache miss) → banco   # mesma query, 5ms depois
request → container-3 (cache miss) → banco   # mesma query, 10ms depois
```

**A solução:** cache externo compartilhado (Redis, Memcached) é dependência
do serviço, não do container individual.

```yaml
services:
  api:
    environment:
      CACHE_URL: redis://redis:6379/0
    deploy:
      replicas: 4   # todas as 4 instâncias usam o mesmo Redis

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
```

Isso também se aplica a sessões de usuário — se a sessão está em memória local,
o usuário perde a sessão quando o request cai num container diferente.

---

## 6. Redes mal modeladas — isolamento excessivo ou insuficiente

**Isolamento insuficiente:** todos os containers na rede padrão do Docker (`bridge`).
Qualquer container pode falar com qualquer outro container no host sem restrição.
Um container comprometido tem acesso a todos os serviços internos.

**Isolamento excessivo:** cada container numa rede própria sem possibilidade de
comunicação. Você acaba usando `network_mode: host` para "resolver" o problema,
eliminando todo o isolamento de rede.

**O modelo correto:** redes por responsabilidade funcional com comunicação explícita.

```yaml
networks:
  frontend:    # nginx ↔ app
  backend:     # app ↔ db, app ↔ redis
  internal:    # componentes que nunca falam com internet
    internal: true

services:
  nginx:
    networks: [frontend]

  app:
    networks: [frontend, backend]

  postgres:
    networks: [backend]

  redis:
    networks: [backend]
```

O banco não está na rede `frontend` — nginx não pode falar com o banco por design.

**Problema frequente com `extra_hosts` e resolução DNS:**

```yaml
services:
  app:
    extra_hosts:
      - "api-parceiro.example.br:10.0.1.50"   # resolve via /etc/hosts no container
```

Isso funciona mas é frágil — quebra se o IP mudar e não há fallback. Prefira
resolução DNS real via servidor interno ou `network_mode: bridge` com configuração
de DNS no daemon.

---

## 7. Storage driver inconsistente após atualização do Docker

Docker 25+ (e especialmente 27+) mudou o comportamento padrão de detecção do
storage driver. Em algumas distribuições, após atualização do Docker, o daemon
pode iniciar com `overlay2` mas com configurações diferentes de `native-overlay-diff`
que alteram como as layers são compostas.

**O sintoma:** containers que funcionavam param de iniciar, ou iniciam com filesystem
corrompido. `docker info | grep "Storage Driver"` ainda mostra `overlay2` mas
o comportamento de escrita mudou.

```bash
# Verificar configuração completa do storage driver
docker info | grep -A5 "Storage Driver"

# Verificar se native-overlay-diff está ativo
docker info | grep "Native Overlay Diff"

# Ver opções do daemon
cat /etc/docker/daemon.json
```

**A causa:** sem `storage-driver` explícito no `daemon.json`, o Docker detecta
automaticamente. Em hosts com kernels mais novos, a detecção pode mudar após
atualização do Docker, resultando em comportamento diferente entre hosts com
versões diferentes do Docker ou kernel.

**A mitigação:** fixar explicitamente no `daemon.json`:

```json
{
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=false"
  ]
}
```

Ver mais detalhes em [ansible-docker-setup](https://github.com/diegobianchetti/ansible-docker-setup)
que trata especificamente deste problema na padronização de hosts.

---

## 8. Logs sem rotação enchendo disco silenciosamente

O storage driver padrão do Docker é `json-file`. Sem configuração de rotação,
cada linha de log é appendada em arquivos em `/var/lib/docker/containers/[ID]/[ID]-json.log`.

Em sistemas com alto volume de log (APIs HTTP, serviços de streaming), esses arquivos
crescem sem limite até o disco encher.

```bash
# Ver tamanho dos logs por container
find /var/lib/docker/containers/ -name '*-json.log' \
  -exec du -sh {} \; | sort -rh | head -10

# Ou via docker system
docker system df -v | grep -A5 "Images\|Containers\|Volumes"
```

**O que acontece quando o disco enche:**

- `docker logs` passa a falhar silenciosamente
- O daemon Docker pode travar ao tentar escrever novos logs
- Outros processos no host que precisam de disco passam a falhar

**Configuração mínima (via daemon.json, aplica a todos os containers):**

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "5"
  }
}
```

Total máximo por container: 250MB (5 arquivos × 50MB). Ajuste para seu caso.

**Atenção:** `max-file` define o número de arquivos rotativos, não o número de
arquivos a manter. Com `max-file: 5`, você tem 4 arquivos comprimidos + 1 ativo.

---

## 9. Bots e crawlers derrubando serviços por falta de bloqueio na borda

Containers com recursos limitados não absorvem tráfego de bots da mesma forma
que um servidor dedicado. Um crawler agressivo ou scanner de vulnerabilidades
pode saturar o container muito antes de chegar perto da capacidade do hardware.

**O problema:** a aplicação dentro do container processa cada request de bot —
consulta banco, gera resposta, escreve log. Com `memory: 512M` e 200 requests/s
de scanner, o container atinge o limite e começa a matar requests legítimos.

**O bloqueio precisa acontecer antes do container de aplicação:**

```nginx
# nginx na borda — bloqueia antes de fazer proxy para o app
geo $block_ip {
  default         0;
  10.100.0.0/16   0;   # rede interna — nunca bloquear

  # IPs de scanners conhecidos
  185.220.101.0/24  1;
  192.241.200.0/24  1;
}

map $http_user_agent $block_ua {
  default                     0;
  "~*masscan"                 1;
  "~*zgrab"                   1;
  "~*nuclei"                  1;
  "~*python-requests/2\.[0-9]" 1;  # scrapers não-identificados
}

server {
  if ($block_ip)  { return 444; }
  if ($block_ua)  { return 444; }
}
```

Ver configurações mais detalhadas de bloqueio em
[nginx-container-hardening](https://github.com/diegobianchetti/nginx-container-hardening).

**Rate limiting como segunda linha:**

```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=20r/s;

location /api/ {
  limit_req zone=api burst=50 nodelay;
  limit_req_status 429;
  proxy_pass http://app:8080;
}
```

`nodelay` processa o burst imediatamente (sem delay artificial) mas rejeita
o que ultrapassar. Sem `nodelay`, o nginx coloca requests em fila e atrasa,
o que em alguns casos piora o problema.

---

## 10. Docker vs Podman — a decisão não é binária

Times migram de Docker para Podman esperando resolução de todos os problemas
de segurança, ou ficam com Docker por inércia sem avaliar os casos de uso onde
Podman é genuinamente superior.

**Diferenças que importam operacionalmente:**

| Aspecto | Docker | Podman |
|---|---|---|
| Daemon | Sim (dockerd rodando como root) | Não (daemonless) |
| Rootless out-of-the-box | Requer configuração | Padrão |
| Compose | `docker compose` (plugin nativo) | `podman-compose` (compatibilidade parcial) |
| Systemd integration | Precisa de systemd unit separada | `podman generate systemd` nativo |
| Kubernetes YAML | Não nativo | `podman generate kube` |
| Socket compatibility | API padrão | Socket compatível com Docker API |
| Redes rootless | Bridge simples | `slirp4netns` — performance menor, menos features |

**Quando Podman é a escolha certa:**

- Ambientes com política de segurança que proíbe daemon rodando como root
- Sistemas que usam systemd como orquestrador (servidor de aplicações sem Compose)
- Máquinas de desenvolvimento onde o usuário não tem sudo
- Migração para Kubernetes — o YAML gerado por Podman é mais limpo

**Quando Docker é a escolha certa:**

- Stack baseada em Docker Compose com features avançadas (healthchecks em depends_on,
  extension fields, profiles)
- Integração com ferramentas que assumem Docker socket (Portainer, Watchtower, CI/CD pipelines)
- Time com experiência em Docker e sem motivação técnica para migrar
- Workloads que precisam de redes overlay complexas (Swarm)

**O problema real:** adotar Podman esperando resolver problemas de segurança sem
ajustar o resto da configuração. Podman rootless com `slirp4netns` em produção
com alto throughput de rede tem performance significativamente menor que Docker
com bridge network. Isso raramente aparece em benchmarks de lab.

A decisão deve ser baseada no workload, não em tendência ou recomendação genérica.
