<p align="center">
  <img src="brand/logo.png" alt="Self-Labs" width="380">
</p>

# SelfDesk

Cliente de acesso remoto da Self-Labs, baseado no [RustDesk](https://github.com/rustdesk/rustdesk).
Ele já vem apontado para o servidor próprio (`rustdesk-id.selflabs.org`) e para o console
[CortenDesk](https://rustdesk.selflabs.org), com a marca Self-Labs. É só instalar e definir a senha:
nada de colar servidor e chave à mão em cada máquina.

Os instaladores são gerados pelo [rdgen](https://github.com/bryangerlach/rdgen) a partir do
[`selfdesk.json`](./selfdesk.json) e publicados nas [Releases](https://github.com/self-labs/selfdesk/releases)
por um workflow que roda todo dia e acompanha as versões novas do RustDesk sozinho.

## Download

Os nomes dos arquivos não levam versão, então estes links sempre entregam a Release mais recente:

| Sistema                | Arquivo                                                                                                      |
| :--------------------- | :----------------------------------------------------------------------------------------------------------- |
| Windows 64             | [`selfdesk.exe`](https://github.com/self-labs/selfdesk/releases/latest/download/selfdesk.exe) ou [`selfdesk.msi`](https://github.com/self-labs/selfdesk/releases/latest/download/selfdesk.msi) |
| Debian/Ubuntu x86_64   | [`selfdesk-x86_64.deb`](https://github.com/self-labs/selfdesk/releases/latest/download/selfdesk-x86_64.deb)   |
| Debian/Ubuntu ARM64    | [`selfdesk-aarch64.deb`](https://github.com/self-labs/selfdesk/releases/latest/download/selfdesk-aarch64.deb) |
| Arch Linux x86_64      | [`selfdesk-x86_64.pkg.tar.zst`](https://github.com/self-labs/selfdesk/releases/latest/download/selfdesk-x86_64.pkg.tar.zst) (`sudo pacman -U`) |
| Outras distros Linux   | `.rpm`, `.AppImage` e `.flatpak` na [Release](https://github.com/self-labs/selfdesk/releases/latest) |

Cada Release traz um `SHA256SUMS` para conferir os arquivos.

## Instalação

> [!WARNING]
> Se a máquina já tem o **RustDesk oficial**, desinstale antes. Os dois usam serviços com nomes
> diferentes (`rustdesk` e `selfdesk`), então rodariam juntos, cada um com o seu ID.

### Windows

Rode o `selfdesk.exe` e clique em **Instalar**. O `.msi` serve para instalação silenciosa ou por GPO.
Depois, em **Configurações → Segurança**, defina a **senha permanente** da máquina: sem ela o
SelfDesk não aceita conexão (a senha temporária fica desligada).

### Debian/Ubuntu (inclusive servidor sem tela)

```bash
cd /tmp && curl -fsSLO https://github.com/self-labs/selfdesk/releases/latest/download/selfdesk-x86_64.deb
sudo apt install -y ./selfdesk-x86_64.deb
```

O pacote instala o comando `selfdesk` e o serviço `selfdesk.service`, que ele mesmo habilita e inicia.
Os nomes com maiúsculas (`SelfDesk`, `SelfDesk.service`) também funcionam: o rdgen dá ao programa o nome
do app, e o workflow acrescenta ao `.deb` o comando e o apelido do serviço em minúsculas. Nos `.rpm`,
AppImage e flatpak, só vale `SelfDesk`.
A senha permanente vai por máquina, pelo terminal (o espaço antes do `sudo` tira a linha do histórico):

```bash
 sudo selfdesk --password 'SENHA_FORTE_DA_MAQUINA'
selfdesk --get-id
systemctl status selfdesk
```

## Migrar e atualizar clientes

Os scripts de [`client/`](./client/) trocam um RustDesk instalado pelo SelfDesk **mantendo o ID e a
senha permanente**, e depois atualizam o SelfDesk para a Release mais recente. Cada um baixa o pacote,
confere o sha256 com o `SHA256SUMS`, compara o ID antes e depois e desfaz a troca se ele mudar. Rodam
desacoplados do terminal, então a conexão remota pode cair no meio que o script termina sozinho.

### Debian/Ubuntu

Um comando baixa (ou atualiza) o script em `/usr/local/sbin/migrate-selfdesk` e roda. No lugar de
`--update`, use a opção que precisar:

```bash
sudo curl -fsSLo /usr/local/sbin/migrate-selfdesk https://raw.githubusercontent.com/self-labs/selfdesk/master/client/migrate-selfdesk.sh && sudo bash /usr/local/sbin/migrate-selfdesk --update
```

| Opção          | O que faz                                                                    |
| :------------- | :--------------------------------------------------------------------------- |
| (nenhuma)      | migra do RustDesk; seguro rodar de novo                                      |
| `--update`     | reinstala o `.deb` da Release mais recente por cima, mesmo ID                |
| `--check`      | só mostra o estado                                                           |
| `--restart`    | reinicia o SelfDesk na tela certa (para o Chrome Remote Desktop no meio)     |
| `--rollback`   | volta para o RustDesk (enquanto ele estiver instalado)                       |
| `--finalize`   | remove o RustDesk, depois de conferir os reboots                             |

Para migrar uma máquina: `--check` (precisa mostrar o ID do RustDesk), sem opção (migra), conexão pelo
SelfDesk no mesmo ID e senha, três reboots com o SelfDesk na tela `:0`, e só então `--finalize`. Se algo
falhar antes disso, `--rollback`.

Vários clientes de uma vez, a partir de uma máquina com acesso SSH a eles (um `sudo` por cliente):

```bash
U=https://raw.githubusercontent.com/self-labs/selfdesk/master/client/migrate-selfdesk.sh
for h in cliente1 cliente2 cliente3; do
  echo "== $h"
  ssh -t "$h" "sudo curl -fsSLo /usr/local/sbin/migrate-selfdesk $U && sudo bash /usr/local/sbin/migrate-selfdesk --update"
done
```

> [!NOTE]
> Não funciona como `curl ... | sudo bash`: o script se relança pelo próprio arquivo para sobreviver à
> queda do SSH. Por isso ele é salvo antes de rodar.

### Windows

> [!WARNING]
> **Migração testada em dois Windows** (21/11/2026): um Windows Server 2025 com o RustDesk do MSI e um
> notebook com o do `.exe`, os dois também com o `-Finalize`, e o notebook com o `-Update`. O `-Rollback` ainda não. Num
> Windows novo, rode antes o `-Check` numa máquina de teste. Para atualizar, use sempre o comando acima, que
> baixa o script do dia: a cópia em `C:\ProgramData\SelfDesk-migrate` é a da última execução desacoplada.

> [!NOTE]
> O `raw.githubusercontent.com` guarda cache por até 5 minutos: logo depois de um push, o comando pode baixar
> a versão anterior. Para pegar uma versão exata, troque `master` na URL pelo hash do commit.

Num PowerShell como Administrador (pelo SSH ou dentro da própria sessão remota: a troca roda numa tarefa
agendada como SYSTEM, então a queda da sessão não a interrompe). No lugar de `-Update`, use a opção que
precisar:

```powershell
[Net.ServicePointManager]::SecurityProtocol='Tls12'; iwr -useb https://raw.githubusercontent.com/self-labs/selfdesk/master/client/migrate-selfdesk.ps1 -OutFile $env:TEMP\migrate-selfdesk.ps1; powershell -NoProfile -ExecutionPolicy Bypass -File $env:TEMP\migrate-selfdesk.ps1 -Update
```

| Opção        | O que faz                                                                        |
| :----------- | :------------------------------------------------------------------------------- |
| (nenhuma)    | migra do RustDesk; o RustDesk fica parado e desabilitado, ainda instalado        |
| `-Update`    | reinstala o `.msi` da Release mais recente por cima, mesmo ID                    |
| `-Check`     | só mostra o estado                                                               |
| `-Rollback`  | volta para o RustDesk (enquanto ele estiver instalado)                           |
| `-Finalize`  | desinstala o RustDesk, depois de conferir a conexão pelo SelfDesk                |

Para migrar uma máquina, nesta ordem:

1. `-Check`: precisa mostrar o ID do RustDesk e a senha permanente como `sim`. Sem senha permanente, o
   script recusa a migração, porque o SelfDesk não aceita a senha temporária.
2. Sem opção: migra (uns 40 segundos).
3. Conecte pelo SelfDesk no mesmo ID, com a mesma senha permanente.
4. `-Finalize` se conectou, `-Rollback` se não.

Log em `C:\ProgramData\SelfDesk-migrate\migrate-selfdesk.log`, numa pasta que só SYSTEM e
Administradores acessam: a tarefa SYSTEM roda uma cópia do script guardada nela, nunca o arquivo baixado. O serviço do RustDesk é achado pelo caminho do executável,
então um `hbbs`/`hbbr` instalado pelo NSSM na mesma máquina nunca é tocado.

### Arch Linux (Omarchy)

Sem script: o pacote conflita com o `rustdesk`, então os dois não convivem como no Debian, e a troca é feita
à mão, pelo SSH ou pelo VNC, nunca de dentro de uma sessão do RustDesk (ela cai no meio). Antes de começar,
confira que o pacote do RustDesk está no cache (`ls /var/cache/pacman/pkg/rustdesk-*`): é ele que permite
voltar.

Atualizar, com o mesmo ID (a identidade fica no `SelfDesk.toml`, que o pacote não toca):

```bash
sudo pacman -U https://github.com/self-labs/selfdesk/releases/latest/download/selfdesk-x86_64.pkg.tar.zst
```

Migrar do RustDesk, como usuário da sessão gráfica:

```bash
# 1. Anota o ID e tira o RustDesk (a pasta /root/.config/rustdesk fica)
rustdesk --get-id
sudo systemctl disable --now rustdesk
sudo pacman -R rustdesk

# 2. Instala o SelfDesk (sobe com um ID novo, trocado logo abaixo)
sudo pacman -U https://github.com/self-labs/selfdesk/releases/latest/download/selfdesk-x86_64.pkg.tar.zst

# 3. Traz a identidade, com o SelfDesk parado. Do usuário vai tudo menos o RustDesk2.toml (opções)
sudo systemctl stop SelfDesk
sudo install -d -m 700 /root/.config/selfdesk
sudo cp -p /root/.config/rustdesk/RustDesk.toml /root/.config/selfdesk/SelfDesk.toml
install -d -m 700 ~/.config/selfdesk
for f in ~/.config/rustdesk/*; do
  b="$(basename "$f")"; [[ "$b" == RustDesk2.toml ]] && continue
  cp -a -T "$f" ~/.config/selfdesk/"${b/#RustDesk/SelfDesk}"
done

# 4. Um drop-in do rustdesk.service (Wayland, D-Bus) passa para o SelfDesk.service
[[ -d /etc/systemd/system/rustdesk.service.d ]] && sudo mv /etc/systemd/system/rustdesk.service.d /etc/systemd/system/SelfDesk.service.d
sudo systemctl daemon-reload && sudo systemctl start SelfDesk

# 5. Confere: tem de ser o ID do passo 1
sleep 8; selfdesk --get-id
```

Se o ID não bater, volte: `sudo systemctl disable --now SelfDesk && sudo pacman -R selfdesk`, reinstale o
RustDesk do cache (`sudo pacman -U /var/cache/pacman/pkg/rustdesk-<versão>-x86_64.pkg.tar.zst`), devolva o
drop-in para `rustdesk.service.d` e `sudo systemctl enable --now rustdesk`.

## O que já vem configurado

Há três níveis: o que é **travado** (vai no `override-settings`, vale até numa máquina que já tinha outra
configuração, e o usuário não consegue trocar), o que é **padrão** (vem pronto, mas dá para mudar em
**Configurações**) e o que é **compilado** no programa.

### Travado

| Opção (chave do RustDesk)                                  | Valor                       | Efeito                                                              |
| :--------------------------------------------------------- | :-------------------------- | :------------------------------------------------------------------ |
| Modo de acesso (`access-mode`)                             | `full`                      | Acesso total                                                        |
| Teclado e mouse (`enable-keyboard`)                        | sim                         | Controla a máquina                                                  |
| Área de transferência (`enable-clipboard`)                 | sim                         | Copiar e colar entre as máquinas                                    |
| Transferência de arquivos (`enable-file-transfer`)         | sim                         |                                                                     |
| Áudio (`enable-audio`)                                     | sim                         | Som da máquina acessada                                             |
| Túnel TCP (`enable-tunnel`)                                | sim                         | Encaminhar portas (RDP, SSH...) pela sessão                         |
| Reiniciar remotamente (`enable-remote-restart`)            | sim                         |                                                                     |
| Gravar a sessão (`enable-record-session`)                  | sim                         |                                                                     |
| Bloquear a entrada do usuário (`enable-block-input`)       | sim                         | Trava teclado e mouse locais durante a sessão                       |
| Alterar a configuração remotamente (`allow-remote-config-modification`) | sim            | Quem acessa pode mudar as configurações desta máquina               |
| Impressora remota (`enable-remote-printer`)                | sim                         |                                                                     |
| Câmera (`enable-camera`)                                   | sim                         |                                                                     |
| Terminal (`enable-terminal`)                               | sim                         | Abre um terminal na máquina acessada                                |
| Senha aceita (`verification-method`)                       | `use-permanent-password`    | Só a permanente; a temporária fica desligada                        |
| Aprovação da sessão (`approve-mode`)                       | `password`                  | Só por senha; ninguém aprova por clique                             |
| Acesso por IP direto (`direct-server`)                     | não                         | Toda conexão passa pelo servidor de ID                              |
| Descoberta na rede local (`enable-lan-discovery`)          | sim                         | Aparece para outras máquinas da LAN                                 |
| Encerrar por inatividade (`allow-auto-disconnect`)         | não                         |                                                                     |
| Ocultar a janela de conexão (`allow-hide-cm`)              | sim                         | Quem está na frente **não** vê a janela da sessão                   |
| Remover o papel de parede na sessão (`allow-remove-wallpaper`) | sim                     | Imagem mais leve no acesso                                          |
| Link "Powered by" (`hide-powered-by-me`)                   | oculto                      |                                                                     |

### Padrão (o usuário pode mudar)

| Item                 | Valor                                                                                     |
| :------------------- | :---------------------------------------------------------------------------------------- |
| Servidor de ID/relay | `rustdesk-id.selflabs.org` (relay deduzido na 21117), em **Configurações → Rede**         |
| Key                  | a chave pública do servidor Self-Labs                                                     |
| API                  | `https://rustdesk.selflabs.org` (CortenDesk: a máquina aparece no console)                |
| Tema                 | segue o sistema                                                                           |
| Instalação e Configurações | liberadas                                                                           |
| Direção              | recebe e faz conexões                                                                     |

### Compilado

Nome (SelfDesk), ícone, logo, tela de privacidade, cores da Self-Labs, e o aviso de nova versão do
RustDesk oficial **desligado** (a atualização vem destas Releases). **Nenhuma senha vem embutida:** a
permanente é definida em cada máquina.

> [!IMPORTANT]
> A janela oculta depende de três opções juntas: aprovação só por senha, só senha permanente e
> `allow-hide-cm` (`hide_cm()` em `hbb_common/src/password_security.rs`). O rdgen só as liga com uma
> senha embutida no build, então elas vão pelo `overrideManual` do [`selfdesk.json`](./selfdesk.json).
> Consequência: **uma máquina sem senha permanente não aceita conexão**. Defina a senha logo na
> instalação.

## Builds automáticos

O workflow [`build.yml`](./.github/workflows/build.yml) roda todo dia às 01:23 (horário de São Paulo):

1. Lê a versão padrão do [rdgen público](https://rdgen.crayoneater.org). O rdgen só compila as versões
   do RustDesk que ele suporta, então uma versão nova só entra depois que ele a adiciona.
2. Se a Release `v<versão>` ainda não existe, gera Windows e depois Linux com o
   [rdgen-cli](https://github.com/AlekseyLapunov/rdgen-cli) (cerca de 40 minutos cada).
3. Publica a Release com todos os arquivos e o `SHA256SUMS`, marcada como a mais recente.

Para rodar na hora: **Actions → build → Run workflow**. O campo `version` força uma versão da lista do
rdgen, e `force` refaz uma Release que já existe (por exemplo, depois de mudar o JSON ou as imagens).

## Personalizar

- **Configuração:** abra o [rdgen](https://rdgen.crayoneater.org), use **Load Configuration** com o
  `selfdesk.json`, ajuste e salve com **Save Configuration** por cima dele. Depois rode o workflow com `force`.
- **Imagens:** as três de [`brand/`](./brand/) (ícone, logo e tela de privacidade) saem do
  [`build-brand.py`](./brand/build-brand.py) a partir do logo da Self-Labs. Regeradas, elas precisam
  voltar para dentro do JSON (pelo **Load/Save** do rdgen).

## Créditos e licença

O SelfDesk **não é um produto oficial do RustDesk**. Ele existe graças a:

- **[RustDesk](https://github.com/rustdesk/rustdesk)**, da Purslane Ltd. e colaboradores, o software de acesso
  remoto em que ele se baseia (AGPL-3.0).
- **[rdgen](https://github.com/bryangerlach/rdgen)**, de Bryan Gerlach, que aplica a personalização e compila
  os clientes (GPL-3.0).
- **[rdgen-cli](https://github.com/AlekseyLapunov/rdgen-cli)**, de Aleksey Lapunov, que dispara o rdgen pelo
  workflow (MIT).

Este repositório está sob a [AGPL-3.0](./LICENSE), a mesma licença do RustDesk. O código-fonte de cada
build está indicado nas notas da Release correspondente: a tag do RustDesk usada, os patches do rdgen e
a configuração deste repositório no commit que gerou os arquivos.

A marca Self-Labs e suas imagens são de propriedade da Self-Labs, todos os direitos reservados, e não
estão incluídas na licença AGPL-3.0 do código.
