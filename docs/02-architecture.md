# 02 — Decisões de arquitetura

As decisões desta fase determinam a qualidade operacional da migração.
Refatorar arquitetura de container em produção é caro — vale a pena acertar agora.

---

## Single process por container

A regra existe por razões práticas, não filosóficas:

- O daemon do container monitora a saúde pelo estado do PID 1 — se você tem dois
  processos no container e o secundário morre silenciosamente, o container aparece
  como saudável
- Restart policy age no container inteiro — você não consegue reiniciar só o nginx
  sem reiniciar o app que está junto
- Escala horizontal replica o container inteiro — se nginx e app estão juntos,
  escalar o app escala o nginx também, que provavelmente não é o que você quer

**Quando a regra quebra na prática:**

Processos auxiliares com ciclo de vida idêntico ao principal podem coexistir
se gerenciados por um supervisor de processo (`s6-overlay`, `supervisord`).
Exemplos legítimos:
- Agent de APM que precisa estar no mesmo processo space do app
- Sidecar de exportação de métricas com dependência forte de memória compartilhada

Nesses casos, use `s6-overlay` — ele trata sinais corretamente e respeita a semântica
de PID 1 em containers.

```dockerfile
FROM debian:12-slim

RUN apt-get update && apt-get install -y s6-overlay && \
    rm -rf /var/lib/apt/lists/*

COPY rootfs/ /
ENTRYPOINT ["/init"]
```

`supervisord` funciona mas tem overhead de configuração maior e o tratamento de sinais
é menos confiável em containers do que `s6`.

---

## Modelagem de redes

Cada rede no Compose cria uma bridge no host. Containers na mesma rede se comunicam
pelo nome do serviço — DNS interno resolvido pelo daemon Docker.

**Padrão por responsabilidade:**

```yaml
networks:
  frontend:      # nginx ↔ app
  backend:       # app ↔ db, app ↔ cache
  monitoring:    # app ↔ prometheus exporter (opcional, isola tráfego de métricas)
```

O banco de dados não deve estar na rede `frontend`. Não porque vai dar problema
imediatamente — vai funcionar — mas porque quando houver uma misconfiguration no nginx,
o banco não será exposto.

**Isolamento vs. conectividade:**

Redes marcadas como `internal: true` não têm rota para a internet pelo gateway da bridge.
Use isso para serviços que nunca precisam de acesso externo: bancos de dados, caches,
registries privados.

```yaml
networks:
  backend:
    internal: true
```

Se um container nessa rede precisar fazer pull de imagem, vai falhar. Certifique-se
de que a imagem já está no host ou use um registry interno também na mesma rede.

**Overlay networks (Swarm):**

Em ambientes com múltiplos hosts Docker (Swarm), use overlay networks.
O tráfego é encriptado por padrão com `--opt encrypted`. A porta UDP 4789 (VXLAN)
precisa estar liberada entre os nós.

```bash
docker network create \
  --driver overlay \
  --opt encrypted \
  --subnet 10.20.0.0/24 \
  app-backend
```

---

## Volumes nomeados vs bind mounts

A decisão não é estética.

**Volumes nomeados:**

```yaml
volumes:
  postgres-data:
    name: ${COMPOSE_PROJECT_NAME:-app}-postgres-data
```

- O daemon Docker gerencia permissões — o diretório é criado com UID/GID corretos
- Portável entre hosts sem depender de estrutura de diretórios
- `docker volume ls` dá visibilidade; `docker volume inspect` mostra metadados
- Backup via `docker run --rm -v volume:/data busybox tar czf - /data`
- Não há colisão de nomes entre projetos se você usa `name:` explícito com
  `${COMPOSE_PROJECT_NAME}` — sem isso, dois projetos com `postgres-data` compartilham o mesmo volume

**Bind mounts:**

```yaml
volumes:
  - /etc/app/config.yaml:/app/config.yaml:ro
  - /var/log/app:/app/logs
```

Use bind mount quando:
- O arquivo é gerenciado por um sistema externo (Ansible, Chef, Puppet) que precisa
  escrever diretamente no filesystem do host
- O conteúdo precisa ser editado diretamente no host sem intermediários
- Integração com ferramentas de backup que operam no filesystem do host

Evite bind mounts para dados de banco de dados em produção — permissões de diretório
são fonte frequente de problemas quando o container roda com UID diferente do owner
do diretório no host.

**tmpfs:**

Para dados temporários que não precisam persistir e onde performance de escrita importa:

```yaml
services:
  nginx:
    tmpfs:
      - /var/run:size=10m,mode=755
      - /var/cache/nginx:size=100m,mode=755
```

Tamanho explícito evita que um processo desgovernado consuma memória RAM ilimitadamente
via tmpfs.

---

## Estado em produção

Containers stateless escalam e fazem rollback trivialmente. Containers com estado
(bancos, filas, objetos) têm operações mais complexas.

**Banco de dados em container:**

Viável para ambientes de desenvolvimento, homologação, e produção com volume bem
configurado. A decisão de colocar banco em container em produção depende de:

- Você tem backup e restore testados? (Não "configurados" — **testados**)
- O storage do volume tem IOPS adequados para o workload?
- Tem processo de upgrade do banco que não depende de acesso ao filesystem do host?

Se qualquer resposta for não, o banco fica fora de container por ora.

**Sessions e estado de aplicação:**

Aplicações que guardam estado em memória local não escalam horizontalmente.
Antes de containerizar, o estado precisa ser externalizado para Redis, Memcached,
ou banco de dados. Isso não é um problema de containerização — é um problema
de arquitetura que a containerização expõe.

---

## Registry privado

Produção não puxa imagens de registry público na hora do deploy. Isso falha por:
- Rate limit do Docker Hub
- Indisponibilidade temporária do registry externo
- Audit trail comprometido de qual imagem está rodando

Opções por custo/complexidade crescente:
1. Docker Registry v2 (self-hosted, sem UI, adequado para times pequenos)
2. Harbor (self-hosted, com UI, RBAC, scan de vulnerabilidades)
3. AWS ECR / GCP Artifact Registry / Azure ACR (gerenciado, paga por storage e transferência)

Para Registry v2 self-hosted, veja o template `private-registry` em
[docker-compose-templates](https://github.com/diegobianchetti/docker-compose-templates).

**Convenção de tags:**

```
registry.infra.example.br/app/api:1.4.2
registry.infra.example.br/app/api:1.4.2-20260115
registry.infra.example.br/app/api:stable   # alias mutável — atualizado no deploy
```

- `latest` não vai para registry de produção
- Tag semântica ou com data para rastreabilidade
- Alias mutável (`stable`, `production`) para facilitar rollback sem mudar config

**Credenciais no host:**

```bash
docker login registry.infra.example.br
# Grava em ~/.docker/config.json ou /root/.docker/config.json para o daemon
```

Em produção, prefira credential helpers ou secrets do orquestrador em vez de
arquivos de credencial no filesystem.

---

## Imagem base

A escolha da imagem base tem impacto direto em segurança, tamanho e manutenibilidade.

| Base | Uso adequado | Quando evitar |
|---|---|---|
| `debian:12-slim` | Binários que precisam de libs do sistema | Imagens muito grandes |
| `ubuntu:24.04` | Times acostumados com Ubuntu, compatibilidade | Quando tamanho importa |
| `alpine:3.x` | Imagens mínimas, tamanho crítico | Aplicações com libs que assumem glibc |
| `distroless/base` | Máxima redução de superfície de ataque | Debug — sem shell |
| `scratch` | Binários Go/Rust completamente estáticos | Qualquer dependência de runtime |

Alpine usa musl libc, não glibc. Binários compilados para glibc não rodam em Alpine
sem ajuste. CGO em Go com Alpine é fonte frequente de bugs difíceis de diagnosticar.

**Multi-stage build** é padrão para linguagens compiladas:

```dockerfile
FROM golang:1.22 AS builder
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /bin/api ./cmd/api

FROM gcr.io/distroless/static-debian12
COPY --from=builder /bin/api /api
ENTRYPOINT ["/api"]
```

A imagem final não tem compilador, package manager, ou shell — reduz superfície
de ataque e tamanho.
