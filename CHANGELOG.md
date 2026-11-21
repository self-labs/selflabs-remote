# Changelog

Todas as mudanças relevantes do SelfDesk ficam registradas neste arquivo.

O formato segue o [Keep a Changelog 1.1.0](https://keepachangelog.com/en/1.1.0/).
A versão é a do RustDesk compilado pelo rdgen (`v1.5.3` = RustDesk 1.5.3), não um SemVer próprio:
mudanças no empacotamento entre duas versões do RustDesk ficam em `[Unreleased]` até a próxima.

## [Unreleased]

### ✨ Added

- **Migrar e atualizar clientes com um comando** ([`client/`](./client/), README, seção "Migrar e
  atualizar clientes"): o `migrate-selfdesk.sh` troca o RustDesk pelo SelfDesk num Debian/Ubuntu
  mantendo o ID e a senha permanente, e com `--update` reinstala o `.deb` da Release mais recente
  por cima. Ele veio do repositório privado dos servidores para cá, sem nomes de clientes, para cada
  máquina poder baixá-lo pelo `raw.githubusercontent.com` num só comando. Sem `--server` rodando,
  o usuário da sessão passa a ser o de UID 1000.
  No Windows, o [`client/migrate-selfdesk.ps1`](./client/migrate-selfdesk.ps1) faz o mesmo (migrar,
  `-Update`, `-Check`, `-Rollback`, `-Finalize`) pelo MSI, numa tarefa agendada como SYSTEM que roda uma cópia do script numa pasta só de SYSTEM e
  Administradores (`C:\ProgramData\SelfDesk-migrate`, que já nasce com essa ACL e é recusada se tiver
  item de outro dono ou link), e copia só a config do serviço, nunca para dentro
  dos perfis de usuário (um link criado pelo usuário desviaria a escrita do SYSTEM). Migração testada no
  jordao (Windows Server 2025, RustDesk 1.4.9, MSI) e na camilla (notebook, RustDesk 1.5.3 pelo `.exe`)
  em 2026-11-21: mesmo ID, NSSM intacto. O `-Finalize` da camilla removeu o RustDesk do `.exe` e o
  serviço extra `RustDeskSvc`, com o SelfDesk no mesmo ID, e o `-Update` reinstalou o MSI por cima
  (`REINSTALL=ALL`) mantendo o ID. O `-Finalize` do jordao removeu o RustDesk do MSI (`msiexec /x`) com os
  serviços NSSM do `hbbs`/`hbbr` intactos.

### 🔧 Changed

- **README com o passo a passo de cada sistema** (seção "Migrar e atualizar clientes"): a ordem da
  migração no Debian/Ubuntu e no Windows (`--check`/`-Check`, migrar, testar a conexão, `--finalize`/
  `-Finalize`, e `--rollback`/`-Rollback` para voltar), a tabela de opções do script do Windows, e uma
  seção do Arch (Omarchy) com o `pacman -U` da atualização e a migração feita à mão, porque o pacote
  conflita com o `rustdesk`.
- **Pacote do Arch no padrão do `.deb`**: o `.pkg.tar.zst` do rdgen saía como pacote `rustdesk`,
  então o `rustdesk` do repositório da distro (o do Omarchy, por exemplo) passaria por cima dele no
  próximo `pacman -Syu`. Agora ele é o pacote `selfdesk` (conflita com `rustdesk`), com a unit
  `SelfDesk.service` em `/usr/lib/systemd/system` e o apelido `selfdesk.service`, o comando
  `/usr/bin/selfdesk` e os `.desktop` em `/usr/share/applications`. Feito por
  [`scripts/repack-arch.sh`](./scripts/repack-arch.sh), chamado pelo workflow depois do passo dos
  `.deb`. O pacote da v1.5.3 foi reempacotado e trocado na Release em 2026-11-20, junto da linha
  dele no `SHA256SUMS`.

### 🐛 Fixed

- **Windows com dois serviços do RustDesk** ([`client/migrate-selfdesk.ps1`](./client/migrate-selfdesk.ps1)):
  na camilla (2026-11-21) havia `RustDesk` e `RustDeskSvc` com o mesmo exe, e a migração só parava o
  primeiro. Habilitado, o outro subiria o RustDesk no boot com a identidade do SelfDesk. Agora a migração
  e o `-Finalize` param e desabilitam todos, o principal é o que tem o nome da pasta do exe, e o `-Check`
  lista os extras e as entradas de desinstalação. Um extra rodando não bloqueia o `-Finalize` (só o
  principal rodando indica rollback), e rodar a migração de novo também desabilita os extras.
- **`-Finalize` recusava o RustDesk instalado pelo `.exe`** ("sem ProductCode"): agora roda o
  `RustDesk.exe --uninstall` da entrada `RustDesk`, só quando o exe é o de um serviço do RustDesk, e
  apaga com `sc.exe delete` o serviço extra que ficar sem exe.
- **Aviso falso "a tarefa agendada ficou registrada"** no fim de toda migração: o resumo roda dentro da
  própria tarefa. O aviso agora só aparece fora dela.
- **O console do Windows dava a migração por encerrada no meio do `msiexec`** ([`client/migrate-selfdesk.ps1`](./client/migrate-selfdesk.ps1)):
  no jordao (2026-11-21) o `Get-ScheduledTask` respondeu "não está rodando" com a tarefa ainda no meio
  (resultado 267009, `SCHED_S_TASK_RUNNING`). A migração terminou certa, mas o console saiu antes com
  "terminou sem o marcador de fim". Agora o fim só conta quando o resultado também não é "rodando", por
  5 consultas seguidas.
- **O `--update` deixava o SelfDesk na tela do Chrome Remote Desktop** ([`client/migrate-selfdesk.sh`](./client/migrate-selfdesk.sh)):
  reiniciado com o Google no ar, o `--server` via também o `:20` dele e ficava com essa tela (três
  clientes em 2026-11-20). O `--update` e a opção nova `--restart` param o Chrome Remote Desktop,
  reiniciam o SelfDesk, esperam a tela certa (`:0`) e sobem o Google de novo, como na ordem do boot.
- **Ícone ausente no Arch**: os `.desktop` pediam `Icon=SelfDesk`, mas os ícones do pacote se
  chamavam `rustdesk.png` e `rustdesk.svg`. Agora são `SelfDesk.png` e `SelfDesk.svg`.
- **Unit fora do controle do pacman**: o `.INSTALL` do rdgen copiava `rustdesk.service` para
  `/etc/systemd/system`, e ela ficava para trás na remoção. Agora a unit faz parte do pacote e o
  `.INSTALL` só roda o `systemctl`.
- **Acesso remoto falhava no Wayland do Arch (Hyprland)** com "The desktop portal on the remote
  device is missing a capability": o RustDesk reconhece o próprio `--server` procurando
  `<nome do app em minúsculas> --server` no `ps` (`is_server_running`, `pipewire.rs`), e o binário
  do pacote se chamava `SelfDesk`. Sem casar, o servidor pedia o portal RemoteDesktop, que o
  `xdg-desktop-portal-hyprland` não tem. Agora o binário real é `/usr/share/SelfDesk/selfdesk`, e
  `SelfDesk` fica como link para ele. O log do servidor passa a ser `selfdesk_rCURRENT.log`.
  Testado no Omarchy em 2026-11-20 pelo CortenDesk, com o mesmo ID. O pacote da v1.5.3 foi
  trocado de novo na Release no mesmo dia, junto da linha dele no `SHA256SUMS`.
- **O mesmo nos `.deb`**: o binário real passa a ser `/usr/share/SelfDesk/selfdesk` também no
  Debian/Ubuntu, pelo novo [`scripts/repack-deb.sh`](./scripts/repack-deb.sh), que o workflow chama
  no lugar do passo que só criava `/usr/bin/selfdesk`. Os dois `.deb` da v1.5.3 foram reempacotados
  e trocados na Release e no `/downloads` do CortenDesk em 2026-11-20, com o `SHA256SUMS`.

## [1.5.3] - 2026-11-19

### ✨ Added

- Cliente SelfDesk gerado pelo rdgen a partir do [`selfdesk.json`](./selfdesk.json): já aponta para
  `rustdesk-id.selflabs.org` e para o CortenDesk (`https://rustdesk.selflabs.org`), com nome, ícone,
  logo, tela de privacidade e cores da Self-Labs, sem senha embutida.
- Workflow diário [`build.yml`](./.github/workflows/build.yml): lê a versão padrão do rdgen público,
  gera Windows e Linux com o rdgen-cli e publica a Release com nomes sem versão, para que
  `/releases/latest/download/<arquivo>` seja uma URL estável. O disparo manual aceita `version` e
  `force`.
- Janela de conexão oculta (aprovação só por senha permanente e `allow-hide-cm`), ligada pelo
  `overrideManual` porque o rdgen só a liga com senha embutida.
- Comando `/usr/bin/selfdesk` e apelido `selfdesk.service` nos `.deb`, ao lado dos nomes
  `SelfDesk` que o rdgen gera.
- README com download, instalação, configurações travadas e créditos, e a licença AGPL-3.0.

### 🐛 Fixed

- **Marca do RustDesk nos builds**: o rdgen-cli manda o JSON como formulário e o Django descarta o
  texto enviado aos campos de arquivo, então os primeiros builds saíram com o ícone do RustDesk e sem
  logo. O `selfdesk.json` passou a levar `iconbase64`, `logobase64` e `privacybase64`, e o job Linux
  falha quando o `.deb` sai sem a logo.

## Legenda

| Tipo          | Significado                               |
| ------------- | ----------------------------------------- |
| ✨ Added      | funcionalidade nova                       |
| 🔧 Changed    | mudança em funcionalidade existente       |
| ⚠️ Deprecated | funcionalidade que será removida em breve |
| 🗑️ Removed    | funcionalidade removida nesta versão      |
| 🐛 Fixed      | correção de defeito                       |
| 🔐 Security   | vulnerabilidade tratada e endurecimento   |

[Unreleased]: https://github.com/self-labs/selfdesk/compare/v1.5.3...HEAD
[1.5.3]: https://github.com/self-labs/selfdesk/releases/tag/v1.5.3
