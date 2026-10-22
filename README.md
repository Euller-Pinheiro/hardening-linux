# hardening-linux

![CI](https://github.com/Euller-Pinheiro/hardening-linux/actions/workflows/ci.yml/badge.svg)
![Status](https://img.shields.io/badge/status-em%20desenvolvimento-C9A84C)
![Licença MIT](https://img.shields.io/badge/licen%C3%A7a-MIT-green)

**[English below](#english)**

Role **Ansible** para **endurecer servidores Ubuntu** de forma repetível — e que explica, no próprio README, o que cada controle faz e por que ele existe.

## Por que este projeto

A maioria dos servidores expostos na internet recebe milhares de tentativas de login por dia. Endurecer à mão, servidor por servidor, é lento e esquece passos. Aqui tudo é declarado, testado em contêiner a cada mudança e pode rodar primeiro em **modo auditoria**, que só relata o que está fora do padrão sem alterar nada.

## O que está pronto e o que está a caminho

O projeto está sendo construído em entregas pequenas, cada uma com testes e CI. Acompanhe pelos commits e pull requests.

- [x] SSH endurecido, fail2ban e atualizações automáticas de segurança
- [ ] Firewall (ufw) com perfis prontos
- [ ] Parâmetros de kernel e rede, e auditd
- [ ] Modo auditoria: relatório de conformidade sem alterar o servidor

Sistemas suportados: **Ubuntu 24.04 LTS** e **Ubuntu 26.04 LTS** (os dois são testados no CI).

## Uso rápido

```bash
pip install ansible-core
git clone https://github.com/Euller-Pinheiro/hardening-linux.git
cd hardening-linux

# inventario.ini
# [servidores]
# meu-servidor ansible_host=203.0.113.10 ansible_user=admin

ansible-playbook -i inventario.ini site.yml --check --diff   # mostra o que mudaria
ansible-playbook -i inventario.ini site.yml                  # aplica
```

> **Regra de ouro ao mexer em SSH:** mantenha uma sessão SSH aberta enquanto aplica a role e só feche depois de testar um login novo, em outro terminal. Se algo der errado, a sessão aberta é o seu caminho de volta.

Para usar a role dentro do seu próprio playbook, copie `roles/hardening` para o seu `roles/` e:

```yaml
- hosts: servidores
  become: true
  roles:
    - role: hardening
      vars:
        hardening_fail2ban_ignoreip: ["198.51.100.7"]   # IP fixo do escritório
```

## Os controles, um a um

Cada bloco pode ser desligado sozinho (`hardening_ssh_enabled`, `hardening_fail2ban_enabled`, `hardening_auto_updates_enabled`).

### 1. SSH endurecido

A role grava **um único arquivo**, `/etc/ssh/sshd_config.d/00-hardening.conf`, e não edita o `sshd_config` do pacote.

**Por que um arquivo `00-`?** O Ubuntu inclui `sshd_config.d/*.conf` logo no começo do `sshd_config`, em ordem alfabética, e no `sshd` **vale o primeiro valor encontrado** para cada opção (ao contrário da maioria dos programas, em que vale o último). Assim o `00-hardening.conf` vence o resto da configuração — inclusive o `50-cloud-init.conf` que muitos provedores de nuvem criam com `PasswordAuthentication yes`. Para ver o que o `sshd` está usando de fato, rode `sudo sshd -T`.

| Controle | Padrão | Por quê |
|---|---|---|
| `PermitRootLogin` | `prohibit-password` | Root só entra com chave, nunca com senha. Depois de criar um usuário com `sudo`, troque para `no`. |
| `PasswordAuthentication` | `yes` (!) | Fica ligado **de propósito**: veja "Desligando o login por senha" abaixo. |
| `KbdInteractiveAuthentication` | `no` | É outro caminho de senha via PAM; só faz sentido com segundo fator (TOTP). |
| `PermitEmptyPasswords`, `HostbasedAuthentication`, `PermitUserEnvironment` | `no` | Mecanismos antigos ou perigosos que quase ninguém usa. |
| `MaxAuthTries` | `3` | Encerra a conexão depois de 3 tentativas: o robô precisa reconectar, e cada reconexão conta no fail2ban. |
| `LoginGraceTime` | `30` | Conexões que não autenticam em 30 s são derrubadas e não ocupam vagas. |
| `ClientAliveInterval` × `CountMax` | `300` × `2` | Sessões mortas (rede caiu) são encerradas em ~10 min. |
| `X11Forwarding`, `AllowAgentForwarding` | `no` | Encaminhar agente SSH expõe suas chaves a quem for root no servidor. Use `ssh -J` (ProxyJump) no lugar. |
| `AllowTcpForwarding` | `local` | Mantém túneis `ssh -L` (acessar um banco pelo SSH) e bloqueia `ssh -R`, que publica portas no servidor. |
| `LogLevel` | `VERBOSE` | Registra a impressão digital da chave de cada login: você sabe **qual** chave usar e qual revogar. |
| `KexAlgorithms`, `Ciphers`, `MACs` | lista moderna | Só curvas modernas (Curve25519), troca de chaves híbrida pós-quântica e cifras autenticadas. Ficam de fora SHA-1, CBC e curvas NIST. |

**Criptografia que acompanha a versão.** O OpenSSH 9.9+ (Ubuntu 26.04) tem `mlkem768x25519-sha256`, troca de chaves resistente a computador quântico. O 9.6 (Ubuntu 24.04) não conhece esse nome — e o `sshd` **se recusa a subir** com um algoritmo desconhecido na configuração. Por isso a role lê `ssh -V` e só inclui o ML-KEM quando o servidor suporta.

**Configuração validada antes de aplicar.** O arquivo passa por `sshd -t` antes de ser gravado, e o handler roda `sshd -t` de novo antes de reiniciar. Configuração inválida nunca chega a derrubar o serviço.

**Ativação por socket.** No Ubuntu 24.04+ quem escuta a porta 22 é o `ssh.socket` do systemd, montado a partir do `sshd_config` por um gerador. Trocar `hardening_ssh_port` exige `daemon-reload` e reinício do socket — os handlers da role fazem isso. (E lembre: mudar a porta reduz ruído nos logs, mas não é controle de segurança.)

#### Desligando o login por senha (com segurança)

Desligar a senha é o controle mais eficaz contra força bruta — e o que mais tranca administradores fora do próprio servidor. Por isso:

1. A role **não** desliga por padrão. Você pede explicitamente: `hardening_ssh_password_authentication: false`.
2. Antes de mexer no `sshd`, a role confere se cada usuário administrador tem `~/.ssh/authorized_keys` com conteúdo. Se não tiver, **a execução para** com uma mensagem explicando o que fazer, e nada é alterado.
3. Os usuários conferidos são `hardening_ssh_admin_users` ou, se a lista estiver vazia, o usuário da conexão do Ansible.

```yaml
hardening_ssh_password_authentication: false
hardening_ssh_admin_users: [admin, deploy]
```

### 2. fail2ban

O fail2ban lê os registros do `sshd` e, depois de `maxretry` falhas dentro de `findtime`, bloqueia o IP no firewall por `bantime`.

| Variável | Padrão | Por quê |
|---|---|---|
| `hardening_fail2ban_maxretry` | `5` | Erros humanos (senha digitada errado) não bloqueiam; robôs, sim. |
| `hardening_fail2ban_findtime` | `10m` | Janela em que as falhas são contadas. |
| `hardening_fail2ban_bantime` | `1h` | Primeiro bloqueio. |
| `hardening_fail2ban_bantime_increment` | `true` | Reincidentes recebem bloqueios cada vez maiores (1 h, 2 h, 4 h…) até `hardening_fail2ban_bantime_maxtime` (`1w`). |
| `hardening_fail2ban_ignoreip` | `[]` | IPs que nunca são bloqueados — coloque o IP fixo do escritório ou da VPN. |

Decisões de projeto:

- **`backend = systemd`**: lê o journal direto. Instalações mínimas do Ubuntu podem nem ter `/var/log/auth.log`.
- **`mode = aggressive`**: conta também conexões que desistem antes da senha (usuário inexistente, troca de chaves abortada), comportamento típico de varredura.
- **Arquivo em `jail.d/00-hardening.local`**: o `jail.conf` do pacote fica intocado, e o `apt` atualiza o fail2ban sem conflito.

Comandos úteis: `sudo fail2ban-client status sshd` (quem está bloqueado) e `sudo fail2ban-client set sshd unbanip 198.51.100.7` (desbloquear).

### 3. Atualizações automáticas de segurança

A maioria das invasões explora falhas **já corrigidas** em servidores que não foram atualizados. O `unattended-upgrades` instala sozinho, todo dia, as correções do repositório de segurança do Ubuntu.

| Variável | Padrão | Por quê |
|---|---|---|
| `hardening_auto_updates_reboot` | `false` | Reiniciar produção é decisão do administrador. Ligue em servidores que toleram reinício (ex.: atrás de balanceador). |
| `hardening_auto_updates_reboot_time` | `"04:00"` | Horário do reinício, se ligado. |
| `hardening_auto_updates_remove_unused_kernels` | `true` | Evita `/boot` cheio — causa clássica de atualização travada. |
| `hardening_auto_updates_mail` | `""` | E-mail para relatório (exige um MTA configurado). |

A role grava `20auto-upgrades` (liga as rotinas diárias) e `52unattended-upgrades-hardening` (ajustes), e **não** edita o `50unattended-upgrades` do pacote, que já limita as origens à segurança do Ubuntu. Como o apt lê `apt.conf.d` em ordem alfabética, o `52` sobrescreve só as opções que define. Para conferir: `apt-config dump | grep Unattended` e `sudo unattended-upgrade --dry-run --debug`.

## Como é testado

O CI roda em cada push e pull request:

1. **yamllint** e **ansible-lint** (perfil `production`).
2. **Molecule + Docker** em Ubuntu 24.04 e 26.04, com systemd rodando dentro do contêiner (os serviços sobem de verdade). A sequência é: aplicar a role → aplicar de novo e exigir **zero mudanças** (idempotência) → verificar.

A verificação (`molecule/default/verify.yml`) testa o **efeito**, não só a existência de arquivos:

- `sshd -T` (configuração efetiva) tem cada controle esperado e nenhum algoritmo fraco;
- ML-KEM aparece **se e somente se** o OpenSSH é 9.9+;
- `fail2ban-client` confirma a jail `sshd` ativa, lendo o journal, com os limites configurados;
- `apt-config dump` confirma as rotinas diárias e o reinício desligado, e `unattended-upgrade --dry-run` roda sem erro;
- a proteção anti-trancamento: pedir senha desligada para um usuário sem chave **tem de falhar**, e o `sshd` tem de continuar como estava.

Para rodar localmente (precisa de Docker):

```bash
pip install -r requirements-dev.txt
ansible-galaxy collection install -r requirements.yml
yamllint . && ansible-lint
molecule test
```

> O Ubuntu 26.04 traz systemd 259, que só funciona com **cgroup v2** no host. Docker em distribuições atuais e os runners do GitHub Actions já usam cgroup v2.

## Estrutura

```
roles/hardening/
├── defaults/main.yml      # todas as opções, comentadas
├── tasks/
│   ├── main.yml           # checa o sistema e chama cada bloco
│   ├── ssh_preflight.yml  # proteção anti-trancamento
│   ├── ssh.yml
│   ├── fail2ban.yml
│   └── auto_updates.yml
├── templates/             # arquivos gerados no servidor
└── handlers/main.yml      # reinícios (com sshd -t antes)
molecule/default/          # cenário de teste: Dockerfile, converge e verify
site.yml                   # playbook de exemplo
```

## Tecnologias

Ansible · Molecule + Docker · ansible-lint · yamllint · GitHub Actions

## Autor

**Euller Pinheiro**, Diretor Técnico da [ESX Tecnologia](https://esxtecnologia.com) — desenvolvimento de sistemas, infraestrutura e TI gerenciada em São Paulo.

---

<a name="english"></a>
## English

Ansible role to harden Ubuntu 24.04/26.04 servers, tested with Molecule (systemd inside Docker) on both releases.

Ready now:

- **SSH**: a single `sshd_config.d/00-hardening.conf` drop-in (sshd uses the *first* value it reads, so it wins over cloud-init's file), validated with `sshd -t` before it is written. Modern key exchange, ciphers and MACs, with post-quantum ML-KEM added only when OpenSSH is 9.9 or newer. Password login stays **on** by default; turning it off is explicit and guarded by a pre-flight check that aborts if an admin user has no `authorized_keys`.
- **fail2ban**: `sshd` jail reading the systemd journal, aggressive mode, incremental ban times and an allowlist.
- **Unattended security upgrades**: daily updates, automatic reboot off by default, old kernels removed.

Coming next: ufw profiles, sysctl and auditd, and an audit-only mode. Licensed under [MIT](LICENSE).
