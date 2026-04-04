# container-migration-notes

Guia de referência para containerização de sistemas em produção.

Voltado para SREs e Infrastructure Engineers com experiência em Linux que estão
migrando workloads reais — não para quem está aprendendo Docker pela primeira vez.

O material assume que você já sabe o que é um container. O foco é nos problemas
que aparecem quando você leva isso para produção: o que avaliar antes, as decisões
de arquitetura que importam, como fazer o cutover sem derrubar nada, o que checar
antes de considerar a migração concluída, e os erros que você vai querer evitar.

---

## Índice

| Documento | Conteúdo |
|---|---|
| [01-assessment.md](docs/01-assessment.md) | Avaliação do sistema antes de começar |
| [02-architecture.md](docs/02-architecture.md) | Decisões de design que definem a qualidade da migração |
| [03-migration-strategy.md](docs/03-migration-strategy.md) | Cutover sem downtime, rollback, validação de paridade |
| [04-production-checklist.md](docs/04-production-checklist.md) | O que precisa estar em ordem antes de declarar conclusão |
| [05-pitfalls.md](docs/05-pitfalls.md) | 10 armadilhas reais de produção |

---

## Público-alvo

- Já administra Linux em produção
- Tem alguma experiência com Docker/Podman, mesmo que limitada
- Está encarregado de migrar um ou mais sistemas para containers
- Quer evitar os erros que só aparecem depois que o ambiente está em produção

Se você está começando com containers do zero, leia a documentação oficial primeiro.
Este material pressupõe que você já fez isso.

---

## O que este guia não é

- Tutorial de instalação do Docker
- Introdução a Kubernetes
- Lista de comandos básicos

---

## Contexto de uso

Os exemplos usam dados fictícios: IPs na faixa RFC 1918, domínios `.example.br`,
nomes de serviços genéricos. Qualquer semelhança com sistemas reais é coincidência.

A maioria dos exemplos assume Docker + Docker Compose. Onde Podman tem comportamento
relevantemente diferente, isso é indicado explicitamente.

---

## Outros repositórios do portfólio

- [ansible-docker-setup](https://github.com/diegobianchetti/ansible-docker-setup) — instalação e padronização de hosts Docker
- [nginx-container-hardening](https://github.com/diegobianchetti/nginx-container-hardening) — hardening de nginx como container
- [docker-compose-templates](https://github.com/diegobianchetti/docker-compose-templates) — templates comentados de ambientes multi-serviço
