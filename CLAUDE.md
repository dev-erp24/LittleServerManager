# Contexto — erp24

Documento para qualquer agente/dev que peque no projeto a meio.
Cobre **porquê**, **o quê**, **como está estruturado**, **convenções**,
**o que está fora de scope** e **como adicionar coisas**.
Lê isto **antes** de mexer em código. Continua a ler o código antes de
aplicar mudanças (este doc pode estar desatualizado).

---

## 1. Objetivo

CLI Go (`erp24`) que provisiona servidores **Debian** (12+). Substitui scripts
bash dispersos por:

- Subcomandos idempotentes por módulo (firewall, ssh, docker, ...).
- Menu interativo quando corrido sem args (`sudo erp24`).
- Wizard automático no primeiro arranque (sem config).
- Estado em ficheiro YAML, sincronizado com UFW.
- Distribuído como `.deb` (nfpm) publicado por MyPublisher em GitHub Releases.

**Foco:** servidor Debian para correr containers (Docker engine rootful) +
acesso controlado por whitelist de IPs.

---

## 2. História / origem

1. Começou em `/Users/pedro/Desktop/Debian/setup-server.sh` — bash monolítico
   (UFW + SSH + Docker + MySQL).
2. Refactorizado para `/Users/pedro/Desktop/Debian/` — modular bash com
   `setup.sh` orquestrador + `modules/01-firewall.sh`, `02-ssh.sh`, etc.
3. MySQL removido (app-specific, não base server).
4. Convertido para Go — projeto atual (`/Users/pedro/Projects/LittleServerManager`).
5. Público em https://github.com/santospedro1993/LittleServerManager.

---

## 3. Decisões arquiteturais (com porquê)

| Decisão | Porquê |
|---|---|
| Go em vez de bash | Idempotência mais limpa, testável, binário único, wrappers tipados sobre system tools. Bash continua a ser quem faz o trabalho real (apt, ufw, systemctl) — Go orquestra. |
| Cobra para CLI | Standard de facto, dá help/version/completion grátis. |
| YAML direto (`gopkg.in/yaml.v3`), **sem viper** | Viper era overkill (precisávamos só de Load/Save). Menos uma dep. |
| Sem TUI lib (huh/bubbletea) | User pediu "só CLI". Prompts via `bufio` stdlib chegam. |
| `internal/runner` wrappa `os/exec` | Centraliza logging, dry-run, root check. |
| Binário em `/usr/sbin/erp24` | Convenção Debian para tools que precisam de root. Em PATH do sudo. |
| Config em `/etc/erp24/config.yaml` | Convenção `/etc/<projeto>/`. Path default no flag `--config`. |
| State separado (`state.yaml`) | Mantém `config.yaml` editável-à-mão; state é gerado. |
| Whitelist única `network.allowed_ips` (não por-porta) | Simplicidade. Vazia = portas abrem a todos. |
| Política `auto_open_ports: true\|false\|ask` | User pode controlar quando módulos abrem firewall. `ask` default. |
| Wizard só no primeiro run; menu nas restantes | Match com o que user pediu: "Setup Inicial (se for), senão Validar setup". |
| Modules devem registar portas em `state.yaml` quando abrem em UFW | Permite `add-ip`/`remove-ip` sincronizar todas as portas geridas sem hardcode. |

---

## 4. Fora de scope (decisão explícita do user)

- **MySQL / Postgres / Redis / compose stacks** — app-specific, fora do `erp24`.
- **SSH keys** (deploy `authorized_keys` + desligar `PasswordAuth`) — user dispensou nesta fase.
- **Swap** — user dispensou.
- **WireGuard** — adiar; user vai pedir como módulo no futuro.
- **Reverse proxy / TLS / certbot / backups** — não pedido.
- **RHEL / Alpine / Arch / não-systemd** — só Debian, só systemd.

Quando user pedir wireguard ou outro: segue o **padrão de módulo** (secção 8).

---

## 5. File layout

```
erp24/
├── main.go                    # entrypoint; version injetado via -ldflags
├── go.mod / go.sum
├── README.md                  # docs públicas (PT)
├── CLAUDE.md                  # este ficheiro
├── config.example.yaml        # template, vai para /etc/erp24/ no .deb
├── nfpm.yaml                  # empacotamento .deb (amd64/arm64)
├── packaging/postinstall.sh   # postinst do .deb (LF obrigatório)
├── .gitattributes             # *.sh / *.service eol=lf (MyPublisher corre em Windows)
├── .mypublisher/workflows/
│   └── producao.yml           # build + nfpm + tag + GitHub Release
├── cmd/                       # cobra subcommands
│   ├── root.go                # rootCmd + flags globais + shouldOpen()
│   ├── menu.go                # menu interativo + runAllModules + helpers
│   ├── init.go                # wizard
│   ├── validate.go            # checks (cresce a cada módulo novo)
│   ├── all.go                 # `erp24 all` → runAllModules()
│   ├── ip.go                  # add-ip / remove-ip + sync UFW
│   ├── firewall.go            # UFW bootstrap (idempotente)
│   ├── ssh.go                 # user + harden + open port + state register
│   ├── docker.go              # install engine + enable daemon
│   ├── fail2ban.go            # jail.local
│   ├── upgrades.go            # unattended-upgrades
│   ├── timesync.go            # parent + sub-cmds: sync, status
│   ├── sysctl.go              # /etc/sysctl.d/99-erp24.conf
│   ├── hostname.go            # hostnamectl + /etc/hosts
│   ├── sentinel.go            # instala + regista agente Sentinel (opt-in)
│   ├── port.go                # port add/remove/allow/revoke/list
│   ├── status.go              # métricas do host (--live)
│   └── system.go              # system update / reboot (+ alias update-server)
└── internal/
    ├── config/config.go       # Config struct + Load/Save + IP add/remove
    ├── state/state.go         # State struct + ManagedPort + Add/Remove
    ├── prompt/prompt.go       # Ask, AskInt, AskIPOrCIDR, Confirm, Choose
    ├── runner/runner.go       # Run, Capture, Stdin, TryRun, AptGet/TryAptGet, RunEnv, DryRun, RequireRoot
    ├── ufw/ufw.go             # Install, Allow*, Delete*, OpenWhitelisted
    ├── ssh/ssh.go             # CreateUser, Harden, OpenFirewall
    ├── docker/docker.go       # RemoveConflicts, InstallRepo, InstallEngine, EnableDaemon
    ├── fail2ban/fail2ban.go   # Install, WriteJailConfig, Enable
    ├── upgrades/upgrades.go   # Install, EnablePeriodic
    ├── timesync/timesync.go   # SetTimezone, Enable, ForceSync, Synced, NTPEnabled
    ├── sysctl/sysctl.go       # Body (const), Write, Apply, Get, Expected
    ├── hostname/hostname.go   # Set, UpdateHosts, Apply, Current
    ├── sentinel/sentinel.go   # Installed, Registered, NormalizeCentral, Run
    ├── sysstat/sysstat.go     # CPU/RAM/disk/net a partir de /proc e /sys
    └── sysupdate/sysupdate.go # apt update/upgrade, needrestart, reboot pendente/agendado
```

---

## 6. Schema do `config.yaml`

`config.yaml` representa **intenção** — sem segredos, sem dados derivados.
Whitelist de IPs **NÃO** está aqui (lê-se direto do UFW). Password SSH **NÃO**
está aqui (pedida em runtime quando o user é criado).

```yaml
schema_version: 1                # ausente = legacy v0 (ver abaixo)
timezone: Etc/UTC                # default se omitido
hostname: srv01                  # opcional; vazio → módulo hostname não corre
fqdn: srv01.exemplo.com          # opcional

ssh:
  port: 2210                     # required
  user: dev24                    # required (password pedida em runtime)

network:
  auto_open_ports: ask           # true | false | ask

sentinel:
  central_url: https://cc.erp24.pt   # só usado se modules.sentinel: true

# Quais módulos opcionais correr em `erp24 all` e na pós-wizard auto-run.
# firewall + ssh são sempre true (baseline de segurança); chamadas
# diretas a `erp24 <módulo>` correm sempre, ignorando estas flags.
modules:
  firewall: true                 # forçado a true em validate()
  ssh: true                      # forçado a true em validate()
  sysctl: true
  timesync: true
  hostname: true                 # ignorado em runtime se cfg.Hostname == ""
  docker: true                   # único opt-in pedido no wizard
  fail2ban: true
  upgrades: true
  sentinel: false                # opt-in (como docker)
```

Campos com default em `internal/config/config.go::validate()`. Required ones
falham com erro explícito. Configs sem `schema_version` (v0, legacy) têm as
flags baseline + docker forçadas a `true` para não perder módulos
silenciosamente — **sentinel fica `false`** (não se regista um agente à
socapa; precisa de install key). `CurrentSchemaVersion` sobe quando o layout
muda.

---

## 7. Schema do `state.yaml`

Gerido pelo erp24. NÃO editar à mão. Vive em `filepath.Dir(cfgPath) + /state.yaml`
(default `/etc/erp24/state.yaml`). Ausência do ficheiro não é erro — `state.Load`
devolve `State{}` vazio e o primeiro `Save()` cria-o.

```yaml
managed_ports:
  # kind: "host" (sshd-like, INPUT chain) or "docker" (container-published,
  # FORWARD chain). Empty kind in legacy files is treated as host.
  - {port: 2210, proto: tcp, label: SSH, kind: host}
  - {port: 5432, proto: tcp, label: postgres, kind: docker}

# Módulos que correram com sucesso ao menos uma vez.
# Usado por `validate` para decidir o que avaliar e pelo menu para mostrar
# estado. Helpers: `state.MarkInstalled(name)` / `state.IsInstalled(name)`.
# `cmd/root.go::markInstalled()` é chamado no fim de cada cmd/<X>.go após
# sucesso (best-effort: falhas a carregar o state não abortam o módulo).
installed_modules:
  - firewall
  - ssh
  - sysctl
  - timesync
  - hostname
  - docker
  - fail2ban
  - upgrades
  - sentinel
```

Cada módulo que abre porta whitelisted **deve** registar via
`st.AddPort(state.ManagedPort{...})` + `st.Save()`. Cada módulo que corre com
sucesso **deve** chamar `markInstalled("<nome>")` (helper em `cmd/root.go`).

---

## 8. Como adicionar módulo novo X

Padrão obrigatório (segue ssh.go ou fail2ban.go como template):

1. **`internal/X/X.go`** — funções puras:
   - `Installed() bool`
   - `Install() error`
   - `Configure(...) error` (toma valores do config, não loaders)
   - `Enable() error`
   - `Status()` ou similar (read-only, p/ validate)
   - **Idempotente**. Usa `runner.Run/Capture/Stdin`. Devolve `error`.
   - **apt sempre via `runner.AptGet(...)` / `TryAptGet(...)`** (só os
     subargs, sem `"apt-get"`): espera pelo lock e corre não-interativo.
     Nunca `runner.Run("apt-get", ...)` — num Debian acabado de arrancar o
     apt-daily prende o lock e dá exit 100.

2. **`cmd/X.go`** — cobra subcommand:
   ```go
   var XCmd = &cobra.Command{
       Use:   "X",
       Short: "...",
       RunE: func(cmd *cobra.Command, args []string) error {
           if err := RequireAdmin(); err != nil { return err }
           cfg, err := config.Load(cfgFile)
           if err != nil { return err }
           runner.Section("X: ...")
           // chama internal/X/* em ordem
           return nil
       },
   }
   func init() { rootCmd.AddCommand(XCmd) }
   ```

3. **`cmd/menu.go`** — dois sítios: `runAllModules()` (slice `mods`, respeita
   `cfg.Modules.<X>`) e `modulesMenu()` (lista + switch; ao inserir antes de
   "all" corrige os números).

4. **`cmd/validate.go`** — `if st.IsInstalled("X") { check(...) } else { skip(...) }`
   + entrada em `moduleRunners()` (para o Check & Fix).

5. Se introduzir campo novo no config:
   - `internal/config/config.go::Config` struct (yaml tag).
   - Default em `validate()`.
   - Prompt em `cmd/init.go::runWizard`.
   - `config.example.yaml`.

6. Atualiza `README.md` (secção Comandos + Como funcionam os módulos).

7. Atualiza este `CLAUDE.md` (secção 9 — lista de módulos).

8. **Não** mudes a ordem em `runAllModules` sem perceber dependências
   (firewall sempre primeiro; ssh depois de firewall; docker antes de
   fail2ban e sentinel).

Armadilhas: dentro de um `*Cmd` nunca referir `rootCmd` (init cycle — usar
`appVersion`); segredos nunca no config (pedir em runtime com
`prompt.AskPassword`); em `--dry-run` `runner.Capture` devolve `""` e chamadas
HTTP têm de ser saltadas à mão; módulo opt-in não pode ficar `true` na
migração legacy v0.

---

## 8b. First-run bootstrap (sem config)

Sequência quando `sudo erp24` corre e `/etc/erp24/config.yaml` não existe
(invocação tem de ser admin):

1. **System update**: `apt update + upgrade + autoremove + autoclean`,
   com `needrestart` em modo list-only (drop-in em
   `/etc/needrestart/conf.d/99-erp24.conf` impede o dialog interativo
   "Which services should be restarted?"). Detecção via
   `needrestart -b -p` + `/var/run/reboot-required`. Reinício de
   serviços individuais NUNCA é feito inline (risco de matar a sessão
   sshd).
2. Se há reboot pendente → prompt 3-way (now / 04:00 / defer). Em
   qualquer escolha, erp24 sai. Re-correr depois retoma daqui.
3. Wizard (timezone — default `Etc/UTC` —, hostname, ssh user/port,
   política `auto_open_ports`, **docker e sentinel como opt-in** — sentinel
   pede também o `central_url`).
   firewall + ssh + sysctl + timesync + fail2ban + upgrades são baseline
   incondicional (`true`). hostname é baseline **condicional**: a flag é
   `c.Hostname != ""` — se o user deixou o hostname vazio no wizard, o
   módulo fica `false` e não corre (`cmd/init.go:111`).
4. `runAllModules()` respeitando `cfg.Modules.<X>` flags.
5. Prompt: ativar auto-launch do menu para o ssh.user (escreve bloco
   marcado em `~/.profile`, NÃO em `.bash_profile` — ver secção 14).

`runMenu()` só lá vai parar quando config existe — caso contrário entra
em bootstrap.

## 9. Módulos atuais (ordem em `erp24 all`)

| # | Cmd | O que faz | Lê do config |
|---|---|---|---|
| 1 | `firewall` | Install UFW, defaults deny/allow, abre 22 (bootstrap), enable, escreve bloco DOCKER-USER em `/etc/ufw/after.rules` + `ufw reload` | — |
| 2 | `timesync` | `set-timezone` + `set-ntp true` + enable timesyncd | `timezone` |
| 3 | `sysctl` | Escreve `/etc/sysctl.d/99-erp24.conf` + `sysctl --system` | — (hardcoded) |
| 4 | `hostname` | `hostnamectl` + atualiza `/etc/hosts` | `hostname`, `fqdn` (skip se vazio) |
| 5 | `ssh` | Cria user **fora do grupo sudo** (operator-only), grava `/etc/sudoers.d/erp24` com NOPASSWD em `/usr/sbin/erp24` (gate é em-app via `RequireAdmin`), hardening via drop-in `/etc/ssh/sshd_config.d/99-erp24.conf` validado com `sshd -t`, abre porta UFW (1ª vez = a todos), regista state, **fecha porta 22 (bootstrap) após abrir a nova** (só se port≠22 e nova porta já permitida; confirm interativo, auto com `-y`; sessão atual sobrevive) | `ssh.user`, `ssh.port`, `network.auto_open_ports` |
| 6 | `docker` | Remove conflitos, install Docker CE (engine + cli + containerd + buildx + compose plugin), `systemctl enable --now docker.service` | — |
| 7 | `fail2ban` | Install, escreve `jail.local` (port=ssh.port, ignoreip = `ufw.SpecificSources` da porta SSH) | `ssh.port` |
| 8 | `upgrades` | Install + escreve `20auto-upgrades` + enable service | — |
| 9 | `sentinel` | Opt-in. Descarrega `<central>/downloads/sentinel-agent[-arm64].deb`, pede *install key* (`ik_`, single-use) em runtime, `POST <central>/api/agents/register` → token → `/etc/sentinel/agent.toml` (0600), `systemctl enable --now sentinel-agent`. Pull-only HTTPS, não abre portas. Idempotente: se `agent.toml` existe não re-regista | `sentinel.central_url`, `modules.sentinel` |

Cada módulo regista-se em `state.yaml::installed_modules` após sucesso.
`validate` só faz checks contra módulos com flag — outros aparecem como `[--]`.

Comandos não-modulares:
- `erp24 system update` — `apt update + upgrade + autoremove + autoclean`.
  needrestart fica em **list-only** (drop-in `/etc/needrestart/conf.d/99-erp24.conf`):
  nunca reinicia serviços inline. Após upgrade detecta serviços/kernel/microcode
  pendentes via `needrestart -b -p` + `/var/run/reboot-required`. Se algo
  precisa atenção, oferece: reboot agora / agendar 04:00 (`systemd-run
  --on-calendar`) / adiar.
- `erp24 system reboot` — atalho directo p/ prompt 3-way de reboot.
- `erp24 update-server` — alias hidden p/ `system update` (compat).
- `erp24 status [--live]` — host metrics (CPU% + clock avg/max/aggregate via
  `/sys/devices/system/cpu/*/cpufreq` ou `/proc/cpuinfo` fallback, RAM, disk
  per-fs via `df`, network rate per-NIC via `/proc/net/dev`). `--live` faz
  refresh 2s; sair com `q`/`x`/`Esc`/`Enter`/`Ctrl+C` (stty raw poll, sem
  goroutines órfãs). Operator-class.

Sub-cmds especiais:
- `erp24 timesync sync` — força re-sync (restart timesyncd)
- `erp24 timesync status` — `timedatectl status`

---

## 10. Subcomandos top-level (todos)

```
erp24                       # menu interativo (English UI; 'b'=back, 'x'=exit)
erp24 init                  # wizard
erp24 validate              # audita estado vs config (read-only)
erp24 all                   # corre módulos selecionados em config.modules.*
erp24 system update         # apt update + upgrade + autoremove + auto-restart
erp24 system reboot         # reboot now / schedule 04:00 / defer
erp24 update-server         # alias hidden p/ system update (compat)
erp24 add-ip [IP]           # atalho: ALLOW from IP em TODAS as portas geridas
erp24 remove-ip [IP]        # atalho: DELETE allow from IP (com fallback p/ Anywhere)
erp24 port add <P>/<PROTO> [LABEL]    # registar + abrir (--restrict abre fechado)
erp24 port remove <P>/<PROTO>          # fechar + remover de state
erp24 port allow <P>/<PROTO> <IP>      # ALLOW from IP só nessa porta
erp24 port revoke <P>/<PROTO> <IP>     # DELETE allow from IP só nessa porta
erp24 port list                        # tabela: portas geridas + sources UFW
erp24 firewall|ssh|docker|fail2ban|upgrades|timesync|sysctl|hostname|sentinel
erp24 timesync sync|status
erp24 status [--live]                  # métricas do host
```

Flags globais:
- `--config <path>` (default `/etc/erp24/config.yaml`)
- `--dry-run` (loga, não executa)
- `-y/--yes` (auto-confirma prompts)
- `--version`

---

## 10b. Modelo de permissões (admin vs operator)

Dois papéis. Decididos em runtime, não persistidos.

- **admin** = real root. `uid==0 && ($SUDO_USER == "" || $SUDO_USER == "root")`.
  Caminho típico: cloud console / IPMI / `su -` com password do root /
  root invocando `sudo erp24` (idempotente, sudo seta SUDO_USER=root).
  Pode tudo (init, ssh, docker, firewall, ip, port, all, ...).
- **operator** = `sudo erp24` invocado de user não-root (ex: dev24).
  `$SUDO_USER` é o user não-root. Só pode: `validate`, `system update`,
  `system reboot`, `timesync status`, `status`, `port list`, `network`
  list/overview, e qualquer subcmd não destrutivo. Tudo o resto é
  rejeitado por `RequireAdmin()` em `cmd/root.go`.

`RequireAdmin()` está aplicado em: firewall, ssh, docker, fail2ban,
upgrades, sysctl, hostname, sentinel, init, add-ip, remove-ip, port
add/remove/allow/revoke. Operator-class cmds usam só `runner.RequireRoot()`.

dev24 (ou seja qual for `ssh.user`) **NÃO** está no grupo sudo. Único
caminho de privilégio é via `/etc/sudoers.d/erp24` (NOPASSWD blanket em
`/usr/sbin/erp24`) e a verificação em-app filtra por subcmd. Para
operações destrutivas: real root login (não há atalho via sudo).

Menu Network é **secure-by-default** (assume `kind=docker`):
- "Abrir porta a 1+ IP (bloqueia o resto)" — o caminho normal: regista a
  porta fechada (`restrict`) e abre-a só aos IP/CIDR indicados; nunca passa
  por Anywhere. Sem IPs a porta fica registada mas FECHADA.
- "Abrir porta a TODOS (Anywhere)" — só com confirmação explícita
  (default "não").
- Fechar porta, adicionar/remover IP a uma porta ou a TODAS.
`--host` (serviço host, INPUT) fica só via CLI `erp24 port add`.

Menu: navegação unificada com `x` (recua em submenus, sai no top-level —
nunca há overlap p.q. submenu não tem ponto de saída e top-level não tem
parent). `b`/`back`/`q`/`quit`/`exit` são aliases. Validate fundiu-se com
"Check & Fix": uma só entrada que mostra o report e, se houver FAILs e
caller for admin, oferece re-correr os módulos com falha.

## 10c. Como correr containers (docker rootful)

Daemon docker corre como root no socket default `/var/run/docker.sock`.
Apenas root tem acesso (não adicionamos users ao grupo `docker` — equivale
a root, melhor não dar isso a operadores).

Caminho único: login como root (ou `sudo -i`) e correr `docker` directo.

```bash
sudo -i                           # ou login como root direto
docker run hello-world
docker compose -f /root/stacks/app/compose.yaml up -d
```

dev24 (operator) **não** corre containers. Único caminho de privilégio
para containers é root login (cloud console / IPMI / `su -`). Mantém
admin/operator separados de workloads.

Notas:
- Compose files vivem em `/root/stacks/<app>/compose.yaml` (ou
  `/opt/stacks/<app>/`, sem preferência forte).
- **Portas docker são FECHADAS por defeito.** O bloco DOCKER-USER que
  `erp24 firewall` mete em `/etc/ufw/after.rules` faz com que `docker run -p`
  não exponha nada ao mundo sem regra UFW correspondente.
- Para expor porta de container: `erp24 port add <PORT>/tcp <LABEL>` (default
  kind=docker → `ufw route allow`). Whitelist por IP via
  `erp24 port allow <PORT>/tcp <IP>`. Remover regra route é
  `ufw route delete allow ...` (não `ufw delete route ...`, que é inválido).
- Traefik com httpchallenge precisa de 80/443 abertos a TODOS.
- **Não** dar acesso ao grupo `docker` a users não-root — é equivalente a
  root sem audit trail.

## 11. Semântica da whitelist de IPs

**Source of truth = UFW**. Config NÃO armazena IPs. Operações lêem `ufw status`
e modificam regras direto.

### Kinds de porta gerida

Cada `ManagedPort` em `state.yaml` tem um `kind` que decide em que chain
iptables a regra entra:

| Kind | Chain UFW | Cobre | Comando UFW |
|---|---|---|---|
| `host` | INPUT (`ufw-user-input`) | Serviços a escutar no host (sshd) | `ufw allow ...` |
| `docker` | FORWARD (`ufw-user-forward` via `DOCKER-USER`) | Portas publicadas por containers (`docker run -p`) | `ufw route allow ...` |

`erp24 port add P/T LABEL` default = `docker` (caso comum: expor container).
`--host` flag para serviços host. Módulos (ssh) registam com kind=host explícito.

Estado de uma porta gerida pode ser:
- **Aberta a todos** → `ALLOW Anywhere` (host) ou `ALLOW FWD Anywhere` (docker).
- **Restrita** → uma ou mais regras `from <ip> to any port P proto T` (no
  chain certo), SEM regra Anywhere em paralelo.

### DOCKER-USER patch

Docker rootful escreve regras em `DOCKER-USER` (filter) + `DOCKER` (nat)
quando publica porta. Tráfego docker passa por FORWARD, não INPUT.
**Sem intervenção, UFW INPUT-only não tem efeito sobre containers.**

`erp24 firewall` escreve bloco marcado em `/etc/ufw/after.rules`:

```
*filter
:ufw-user-forward - [0:0]
:DOCKER-USER - [0:0]
-A DOCKER-USER -m conntrack --ctstate ESTABLISHED,RELATED -j RETURN
-A DOCKER-USER -j ufw-user-forward
-A DOCKER-USER -j RETURN -s 10.0.0.0/8
-A DOCKER-USER -j RETURN -s 172.16.0.0/12
-A DOCKER-USER -j RETURN -s 192.168.0.0/16
-A DOCKER-USER -j DROP
COMMIT
```

A 1ª regra (ESTABLISHED,RELATED) é **obrigatória**: sem ela as respostas
a ligações que os próprios containers abrem (SYN-ACK de um IP público)
caem no DROP e todo o outbound TCP dos containers dá timeout (ACME do
Traefik, S3, ...). Corrigido na 0.6.9 — **nunca** correr `erp24 firewall`
com um binário anterior (reescreve o bloco sem a regra). Desbloqueio
efémero num servidor antigo:
`iptables -I DOCKER-USER 1 -m conntrack --ctstate ESTABLISHED,RELATED -j RETURN`.

Só existe para IPv4 (`after.rules`); falta `after6.rules` (pendente).

Efeito: DOCKER-USER salta para `ufw-user-forward` (onde `ufw route allow`
mete regras). Tráfego docker→docker (private LAN) faz RETURN cedo, externo
sem regra cai em DROP. `ufw reload` aplica.

Resultado: container que faz `docker run -p 5432:5432` está **fechado** ao
mundo até `erp24 port allow 5432/tcp <IP>`.

### Comandos

`erp24 add-ip X`:
1. Para cada porta em `managed_ports`: emite `ufw [route] allow from X to
   any port P proto T` consoante `kind`.
2. Se a porta tinha regra Anywhere (mesmo chain), remove-a → passa a restrita.

`erp24 remove-ip X`:
1. Para cada porta gerida: `ufw delete [route] allow from X ...` consoante kind.
2. Não re-abre a todos automaticamente (footgun antigo; portas que ficam
   sem regras ficam fechadas mesmo).

`fail2ban` lê `ignoreip` de `ufw.SpecificSources(sshPort, "tcp")` (INPUT).
SSH é kind=host, por isso a leitura INPUT está correcta.

Visualização (`Network → List ports`) chama `kindAllowedSources(kind, ...)`
por porta — host olha INPUT, docker olha FORWARD.

---

## 12. Convenções de código

- **Erros**: `fmt.Errorf("contexto: %w", err)` com wrap.
- **Logs**: PT amigável ("UFW já presente.", "User 'X' criado.").
- **Sections**: `runner.Section("X: o que vou fazer")` antes de cada bloco.
- **Status**: `runner.Log(...)` para linhas de progresso.
- **Escrita de ficheiros system**: `os.WriteFile(path, []byte(body), 0644)` — preferir sobre `tee` via stdin (mais limpo, sem output duplicado).
- **Comandos shell**: `runner.Run("apt-get", "install", "-y", "-qq", "pkg")` — args separados, sem string interpolation.
- **Comandos que podem falhar**: `runner.TryRun(...)` ignora erro (ex: remover pkg que pode não existir).
- **Captura de output**: `runner.Capture(...)` devolve `(string, error)`. Erro discardable se semântica é "vazio = não existe".
- **Stdin**: `runner.Stdin(body, "cmd", "arg")` para piping.
- **Cobra**: cada subcommand em ficheiro próprio (`cmd/X.go`), regista via `init()`.
- **`SilenceUsage: true`** já no rootCmd — erros não imprimem usage spam.
- **`validate`** deve devolver erro se algum check FAIL → exit code != 0 → CI-friendly.
- **Commits/PRs**: NÃO incluir `Co-Authored-By: Claude ...` nem `🤖 Generated with [Claude Code]` em commits ou bodies de PRs. Autoria é só do user.

---

## 13. Build & release

### Local

```bash
go build -o erp24 .
go build -ldflags "-X main.version=0.1.0" -o erp24 .   # com versão
GOOS=linux GOARCH=amd64 go build -o erp24-linux .      # cross-compile
```

### Release (MyPublisher — `.mypublisher/workflows/producao.yml`)

Já não é GitHub Actions (`.github/` foi apagada em ago-2026). O MyPublisher
corre localmente (Windows/Git Bash ou Mac) no branch `master`:

1. `version/resolve` — versão pedida na UI (sugere o próximo patch).
2. Build — para amd64 + arm64: `CGO_ENABLED=0 go build -trimpath
   -ldflags "-s -w -X main.version=$VERSION" -o bin/erp24 .` e
   `nfpm pkg --packager deb` (config em `nfpm.yaml`; nfpm instalado via
   `go install` se faltar). Gera `dist/erp24_<version>_<arch>.deb` +
   alias `dist/erp24_<arch>.deb` (para `releases/latest/download/`).
3. `git/tag` — tag **sem prefixo `v`** (desde 0.4.1; as `v0.x` antigas ficam).
4. `github/release` — notes auto + os 4 `.deb`.

Conteúdo do pacote (`nfpm.yaml`): binário → `/usr/sbin/erp24`,
`config.example.yaml` → `/etc/erp24/` (não é conffile — atualiza sempre),
README → `/usr/share/doc/erp24/`, postinst `packaging/postinstall.sh`.

**CRLF**: como o MyPublisher pode correr em Windows, o `postinstall.sh` com
CRLF partia o postinst (`#!/bin/bash\r`). Proteções: `.gitattributes`
(`*.sh`/`*.service eol=lf`) + `tr -d '\r'` no `producao.yml` antes do nfpm.
Não remover.

Nome do pacote = `erp24` (renomeado de `lsm`: existe um pacote Debian
oficial `lsm` e `apt install lsm` substituía-o).

Ao re-descarregar o `.deb` com `wget`, apagar primeiro (`rm -f erp24_*.deb`)
— senão fica `.deb.1` e o apt instala o antigo.

### Postinst behavior

`packaging/postinstall.sh` (versão via `dpkg-query`). Distingue install
fresco (`$2` vazio) vs upgrade (`$2 = OLD_VERSION`):

- **Fresh**: imprime paths + `sudo erp24` next step.
- **Upgrade**: imprime "X → Y" + sugere `diff config vs example` + `sudo erp24 validate`.

`config.yaml` + `state.yaml` **NÃO** estão no manifest do package → preservados sempre.
`config.example.yaml` é atualizado a cada install.

---

## 14. Locations no servidor

| Path | Conteúdo | Gerido por |
|---|---|---|
| `/usr/sbin/erp24` | binário real | dpkg |
| `/usr/local/bin/erp24` | wrapper `exec sudo /usr/sbin/erp24 "$@"` (`/usr/sbin` não está no PATH default de users não-root) | erp24 `PersistentPreRunE` (idempotente) |
| `/etc/erp24/config.yaml` | source of truth | user / wizard |
| `/etc/erp24/config.example.yaml` | template do schema | dpkg (atualizado em upgrades) |
| `/etc/erp24/state.yaml` | portas geridas | erp24 (read/write) |
| `/etc/sysctl.d/99-erp24.conf` | sysctl tuning | erp24 sysctl |
| `/etc/ufw/after.rules` (bloco marcado) | DOCKER-USER hook → ufw-user-forward + private LAN RETURN + DROP default. Faz `ufw route allow` cobrir portas docker. | erp24 firewall |
| `/etc/fail2ban/jail.local` | jail config | erp24 fail2ban |
| `/etc/apt/apt.conf.d/20auto-upgrades` | unattended-upgrades periodic | erp24 upgrades |
| `/etc/sudoers.d/erp24` | NOPASSWD blanket em `/usr/sbin/erp24` ao SSH user | erp24 ssh |
| `/etc/ssh/sshd_config.d/99-erp24.conf` | sshd Port + PermitRootLogin no + PasswordAuthentication yes | erp24 ssh |
| `/etc/sentinel/agent.toml` | config + token do agente Sentinel (0600) | erp24 sentinel |
| `/etc/needrestart/conf.d/99-erp24.conf` | needrestart em modo list-only (sem dialog interativo no apt) | erp24 system update |
| `/home/<ssh.user>/.profile` (bloco marcado) | auto-launch do menu no login interativo | erp24 bootstrap |
| `/usr/share/doc/erp24/README.md` | docs | dpkg |

---

## 15. User profile

- **Email**: pedro.santos@erp24.pt
- **Idioma**: PT (mensagens, prompts, README, este doc).
- **Plataforma dev**: macOS (darwin/arm64). Server target: Debian linux.
- **GitHub**: santospedro1993/LittleServerManager (público).
- **Estilo de comunicação**: resposta seca e direta. Sem fluff/preâmbulos.
- **Go fora do PATH** em shells não-interativos: `export PATH="$PATH:/usr/local/go/bin"`.
  Validar com `go build ./... && go vet ./... && go test ./...`. Não há Debian
  local — lógica de servidor só se prova no servidor real.
- **Fluxo**: faz perguntas curtas, prefere ver opções com trade-offs antes de decidir, depois manda implementar. Aprecia validate/dry-run.

---

## 16. Coisas a saber antes de mudar código

- **Não introduzir deps novas sem pensar duas vezes**. Atualmente: `cobra`, `yaml.v3` + indireto.
- **Manter mensagens em PT**.
- **Idempotência sempre**. Cada módulo verifica estado antes de agir. Re-correr 10× tem o mesmo efeito que correr 1×.
- **Validate cresce com módulos**. Se adicionas módulo, adiciona checks ao `cmd/validate.go`.
- **Wizard cresce com config**. Campo novo → prompt no wizard.
- **`config.example.yaml`** tem de espelhar o schema atual (é o template que vai para `/etc/erp24/`).
- **CLI deve continuar a funcionar sem args** → menu. `erp24 all -y` deve funcionar sem prompts.
- **Não partir o postinst / `nfpm.yaml`** sem testar (`nfpm pkg` + `dpkg-deb --info`/`--contents` do resultado).
- **Versão é injetada via ldflags** — `main.version` é `"dev"` se não for set.

---

## 17. Próximos módulos planeados (por user)

- **wireguard** — VPN. Config provável: peer config inline ou path para conf. Server ou client peer.
- **container monitoring** — par do `erp24 status` para containers. Ler cgroup v2 + `docker stats`. CPU/RAM/disk-IO/net por container.
- **ssh-keys** — deploy de `authorized_keys` + opção para deprecar PasswordAuth (`PasswordAuthentication no` no drop-in atual). Mais seguro que password.
- **backup** — snapshot tar de `/etc/erp24/{config,state}.yaml` + UFW rules + sshd drop-in para recuperar config rapidamente.

(Lista atualiza-se à medida que user pede.)

---

## 17b. TODO — bugs/melhorias notados em revisão (2026-05-07, revisto 2026-10-16)

Lista de coisas detetadas em full review pós-refactor. Não bloqueiam; correr quando houver tempo.

### Bugs com risco real

- [x] ~~**`portRevoke` reabre a TODOS automaticamente**~~ — resolvido: agora só remove e avisa que a porta fica fechada.
- [ ] **IPv6 cego em `allowedSourcesForDirection`** (`internal/ufw`): salta linhas `(v6)` → `IsOpenToAll`/`IsRouteOpenToAll` não veem um Anywhere só-IPv6 se v4/v6 dessincronizarem.
- [ ] **Bloco DOCKER-USER só em `after.rules`** — falta o equivalente em `after6.rules`.
- [ ] **Validate de managed ports não associa o FAIL ao módulo dono.** `currentModule = ""` antes do loop de portas. "Check & Fix" não pode reaplicar. Fix: associar SSH port → ssh module; portas custom → não auto-fix.
- [ ] **Wizard não rejeita usernames perigosos** (`root`, users existentes em uso). Pode mexer em accounts não-erp24. Fix: validar contra `getent passwd` + lista de reserved.
- [ ] **`fail2ban` validate só checka active**, não confirma jail SSH carregado. Pode estar "active" sem proteger nada. Fix: parse `fail2ban-client status sshd`.
- [ ] **Auto-launch só funciona p/ bash**. Se shell do user for zsh/fish, `.profile` não é lido. Fix: sniff shell, escrever em `~/.zprofile` / `~/.config/fish/config.fish` consoante.
- [ ] **`hostname.Apply` falha em containers** sem privilégio (LXC restrito, Docker host PID=1). `hostnamectl` retorna erro. Fix: try-fallback para `echo "$name" > /etc/hostname; hostname "$name"` se hostnamectl falhar.

### Inconsistências / menores

- [ ] **Logs PT misturados com EN** em `internal/{ufw,timesync,upgrades,fail2ban}` e `cmd/{ip,port,system,init}`. Translate.
- [ ] **`runner.Capture` retorna combined output** (stdout+stderr). Algumas chamadas (ex: `grep`) podem ter stderr que polui o que esperamos como valor. Considerar `runner.CaptureStdout`.
- [ ] **`runner.Run` log line não escapa args** com espaços (cosmético; comando real é seguro porque exec.Command separa args).
- [ ] **Sysctl drop-in** `99-erp24.conf` não distingue IPv4-only hosts. `net.ipv6.conf.all.forwarding=1` em host puramente IPv4 falha sysctl com warning. Fix: detetar IPv6 e omitir se ausente.
- [ ] **Docker `RunC`** removido em RemoveConflicts mas pode ser dep doutros pkgs. Adicionar `apt-get autoremove --simulate` antes de remove.
- [ ] **Status NetRate** sample 800ms — bloqueia menu por isso. Pode mostrar primeiro valor "—" e reler.

### Coisas a fazer / módulos

- [ ] Adicionar **per-container status** ao `erp24 status` (depois do MVP base).
- [ ] Adicionar **wireguard** module (configurável ou peer-only).
- [ ] Adicionar **ssh-keys** module (deploy authorized_keys + opção desligar PasswordAuth).
- [ ] Adicionar **backup** subcmd (tar de config+state+ufw+sshd drop-in).
- [ ] Considerar adicionar `erp24 doctor` que corre validate + dependency-check.
- [ ] Adicionar suporte a IPv6 nas regras UFW (`ufw status` IPv6 entries são skipped agora; user pode querer add-ip de IPv6).

---

## 18. Coisas que provavelmente vão precisar de cuidado

- **Password SSH não persiste**. É pedida só quando o user ainda não existe. Se rodar `erp24 ssh` sobre user existente, password mantém-se a que já estava. Para reset: `passwd <user>` manual.
- **Sysctl `net.ipv6.conf.all.forwarding=1`** pode ser indesejado em hosts puramente IPv4. Considerar tornar opcional via config.
- **Fail2ban backend `systemd`** assume rsyslog/journald. Em Debian default funciona. Em hosts customizados pode falhar.
- **Docker rootful**: socket `/var/run/docker.sock` é só-root. Não adicionar users ao grupo `docker` (= root sem audit). Workloads correm via `sudo -i` ou root login.
- **`erp24 timesync` em containers/LXC** sem CAP_SYS_TIME falha. Não tratamos esse caso.
- **`hostname` em containers** pode estar bloqueado pelo runtime. `hostnamectl` retorna erro silencioso? Verificar.
