# 01 — Avaliação do sistema antes de containerizar

Antes de escrever qualquer Dockerfile, você precisa entender o que está migrando.
A maioria dos problemas de containerização não surge na migração — surge porque algo
não foi mapeado na fase de avaliação.

Este documento cobre o que investigar e onde as surpresas costumam aparecer.

---

## Dependências de sistema

Liste tudo que o sistema consome do SO além de CPU e memória.

**O que mapear:**

```bash
# Pacotes instalados que o processo usa diretamente
dpkg -l | grep -E '^ii' | awk '{print $2, $3}'

# Bibliotecas dinâmicas do binário principal
ldd /usr/bin/meu-servico

# Arquivos abertos pelo processo em runtime
lsof -p $(pgrep meu-servico) | awk '{print $9}' | sort -u

# Módulos de kernel carregados (cgroups, namespaces, etc.)
lsmod | grep -E 'nf_|ip_|br_|veth'
```

Atenção especial para:
- Dependências de módulos de kernel (`ip_tables`, `nf_conntrack`) — containers herdám
  o kernel do host, mas o módulo precisa estar carregado antes
- Binários que fazem syscalls raras (`ptrace`, `mknod`, `mount`) — o seccomp default
  do Docker bloqueia ~44 syscalls
- Processos que leem de `/proc` ou `/sys` com expectativas específicas de namespace

---

## Dados persistentes

Containers são efêmeros por design. Qualquer dado que precisa sobreviver a um restart
precisa ser identificado agora.

**Mapeamento:**

```bash
# Onde o processo escreve
strace -e trace=openat,open -p $(pgrep meu-servico) 2>&1 | grep O_WRONLY | head -50

# Diretórios com escrita recente
find /opt/meu-servico /var/lib/meu-servico -newer /tmp -type f 2>/dev/null

# Tamanho dos dados para estimar volume
du -sh /var/lib/meu-servico/*
```

**Categorias e decisão de volume:**

| Tipo de dado | Volume nomeado | Bind mount | Nenhum |
|---|---|---|---|
| Banco de dados | Sim | Não recomendado | Nunca |
| Uploads de usuário | Sim | Se backup externo gerencia | Nunca |
| Config dinâmica | Depende | Se gerenciada por CM externo | Nunca |
| Cache regenerável | Não | Não | Sim (tmpfs ou ephemeral) |
| Logs | Não | Se driver externo coleta | Sim (stdout/stderr) |

Logs para stdout/stderr não é apenas convenção — é o que permite que o daemon de logs
do host (journald, fluentd, loki) colete sem instrumentação adicional no container.

---

## Variáveis de ambiente e configuração

Sistemas legados têm configuração espalhada em múltiplos lugares.

**Onde procurar:**

```bash
# Variáveis que o processo lê no init
strings /proc/$(pgrep meu-servico)/environ | sort

# Arquivos de config referenciados
lsof -p $(pgrep meu-servico) | grep REG | grep -v 'mem\|txt'

# Crontabs e systemd units que definem env
systemctl show meu-servico | grep -i env
cat /etc/default/meu-servico 2>/dev/null
```

**Problemas comuns:**

- Configuração hardcoded no código que nunca foi externalizada — você vai descobrir isso
  quando o container subir e não conseguir conectar no banco
- Valores que dependem do hostname do servidor (`$(hostname)` em scripts de init)
- Caminhos absolutos que assumem estrutura de diretórios do SO host (`/opt/app/conf`)
- Segredos em arquivos com permissões restritivas (`chmod 600`) — no container você
  precisa de outra estratégia (secrets do Compose/Swarm, variável de ambiente, vault)

---

## Portas e interfaces de rede

```bash
# Portas em uso pelo processo
ss -tlnp | grep $(pgrep meu-servico)

# Ou com netstat em sistemas mais antigos
netstat -tlnp | grep $(pgrep meu-servico)

# Conexões estabelecidas (mapeia dependências de rede)
ss -tnp | grep $(pgrep meu-servico)
```

Registre:
- Porta de escuta e em qual interface (0.0.0.0 vs 127.0.0.1 vs IP específico)
- Serviços que o processo consome (banco, cache, APIs externas, LDAP)
- Protocolos não-HTTP: se o sistema usa UDP, SCTP, ou raw sockets, o mapeamento
  de portas do Docker tem comportamento diferente

**Atenção com `host` network mode:** resolver binding em interface específica
através de `network_mode: host` é tecnicamente viável mas desabilita o isolamento
de rede do container — qualquer processo no container enxerga todas as interfaces
do host. Isso quase nunca é a solução certa.

---

## Processos em background

Sistemas legacy frequentemente são mais complexos do que parecem no PID principal.

```bash
# Processos filhos
pstree -p $(pgrep meu-servico)

# Processos que se comunicam via IPC
lsof -p $(pgrep meu-servico) | grep -E 'FIFO|unix|PIPE'

# Crons do usuário do serviço
crontab -u meu-servico -l 2>/dev/null

# Systemd timers associados
systemctl list-timers | grep meu-servico
```

**Implicação direta:** containers rodam um processo principal (PID 1). Processos
filhos rodam, mas a semântica de sinais muda — `SIGTERM` vai para o PID 1, que precisa
propagar para os filhos. Se o processo principal é um shell script que faz `exec` em
outro binário, isso funciona. Se é um script que faz fork sem `wait`, você vai ter
processos zumbi e shutdown problemático.

Crons não pertencem dentro de containers. Extraia jobs periódicos para:
- `docker run --rm` disparado por cron no host
- `CronJob` se estiver em Kubernetes
- Serviço dedicado com loop interno se a frequência for alta

---

## Integrações externas

```bash
# DNS que o processo resolve
tcpdump -i any port 53 -nn &
sleep 30 && kill %1

# Certificados TLS que o processo valida
openssl s_client -connect api.parceiro.example.br:443 -showcerts 2>/dev/null | \
  openssl x509 -noout -subject -issuer

# Chaves SSH, tokens, certificados usados para autenticação
find /home /opt /etc -name '*.pem' -o -name '*.key' -o -name 'id_*' 2>/dev/null | \
  xargs -I{} ls -la {}
```

**O que vai dar problema:**

- Certificados CA privados instalados no SO — no container você precisa adicionar
  à image (`COPY ca-bundle.crt /usr/local/share/ca-certificates/` + `update-ca-certificates`)
  ou montar como volume
- Autenticação Kerberos/SSSD/LDAP que depende do hostname do servidor
- Integrações que whitelist por IP de origem — o IP do container não é o IP do host
  sem configuração explícita de NAT/SNAT
- NFS mounts — containers podem usar NFS via volume driver, mas não via `/etc/fstab`

---

## O que vai dar problema — resumo direto

| Situação | Impacto | Mitigação |
|---|---|---|
| Processo assume `/tmp` persistente entre restarts | Dados perdidos no restart | Identificar e usar volume ou tmpfs explícito |
| Config hardcoded no código | Container não consegue ser configurado externamente | Refatorar antes de containerizar |
| Processo escuta em IP específico do host | Container não responde | Garantir que bind seja em `0.0.0.0` |
| Syscalls bloqueadas pelo seccomp | Container crasha silenciosamente | Testar com `--security-opt seccomp=unconfined` e depois criar perfil customizado |
| Cron interno ao processo | Comportamento imprevisível após scale horizontal | Externalizar para job dedicado |
| Processo faz `setuid`/`setgid` em runtime | Falha com `--user` ou capabilities restritivas | Ajustar Dockerfile para rodar com UID correto desde o início |
| Dependência de `systemd` ou `dbus` | Não funciona em container padrão | Re-avaliar se containerização faz sentido para este componente |

O último item é mais comum do que parece: sistemas de monitoramento, agentes de MDM,
e algumas ferramentas de backup assumem `systemd` como init. Nesses casos,
containerização pode não ser a resposta certa.
