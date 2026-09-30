# my-fedora-server

Cola de comandos do setup do meu Fedora Server. Não é tutorial — é pra eu lembrar o que fiz e
onde buscar cada valor.

Baseado em: [TechHut — the ULTIMATE Fedora Server Guide](https://www.youtube.com/watch?v=kOEcTGZWiUQ)
· [versão escrita](https://techhut.tv/fedora-server-guide-cockpit-zfs-podman)

Convenções:
- `<algo>` = placeholder, pegar o valor com o comando logo acima.
- `# confere` = validar no meu caso antes de rodar.
- Nenhum valor daqui é o do vídeo. Todos os IPs, discos e interfaces saem dos comandos de descoberta.

## Hardware

Beelink · Intel Celeron N5095A (4 núcleos, 800–2900 MHz) · 8 GB RAM · SSD 238 GB · Fedora Server 44

## Estado atual

| | |
|---|---|
| Feito até | atualizações automáticas |
| Próximo | Docker (seção final, ainda não executado) |
| Não feito | ZFS (não preciso por ora) · NVIDIA (sem GPU dedicada) · Podman (vou de Docker) |

---

## 1. BIOS

Del/F7 no boot. Nomes variam no BIOS da Beelink — `# confere` em todos.

- `Advanced` → `CPU Configuration` → **Intel Virtualization Technology (VT-x)** = `Enabled`
- `Advanced` → **VT-d / IOMMU** = `Enabled`
- `Chipset`/`Power` → **Restore on AC Power Loss** / `State After G3` = `Power On`
  (servidor tem que voltar sozinho depois de queda de luz)
- **Fast Boot** = `Disabled` (senão não dá tempo de entrar no setup)
- **Boot order**: USB na frente durante a instalação, depois SSD interno
- **Secure Boot**: deixei ligado. Só desligar se precisar de módulo DKMS (ZFS, NVIDIA)

Conferir depois, já no sistema:

```bash
lscpu | grep -i virtualization          # espera: VT-x
sudo dmesg | grep -iE 'DMAR|IOMMU' | head
```

## 2. Instalação

ISO em <https://fedoraproject.org/server/download>.

Gravar o pendrive (rodei no desktop). Identificar o dispositivo **antes**:

```bash
lsblk -o NAME,SIZE,MODEL,TRAN,MOUNTPOINTS   # o USB é o de TRAN=usb
sudo dd if=Fedora-Server-dvd-x86_64-44-*.iso of=/dev/<sdX> bs=4M status=progress oflag=direct conv=fsync
```

Alternativa sem dd: `flatpak install flathub org.fedoraproject.MediaWriter`

No Anaconda:

- **Installation Destination**: marcar só o disco do SO. Layout default (LVM + XFS), sem mexer.
- **Network & Hostname**: habilitar a interface e já definir o hostname.
- **Root Account**: desabilitado (uso `sudo` pelo meu usuário).
- **User Creation**: usuário admin, marcar "Add administrative privileges".
- **Software Selection**: `Fedora Server Edition` + addon `Headless Management` (é o que traz o Cockpit).

Pegadinha: o instalador aloca só ~15 GB na raiz mesmo em disco grande. Resolvido na seção 4.

## 3. Primeiro acesso

Descobrir o IP do servidor (na tela dele, ou pelo DHCP do roteador):

```bash
ip -4 addr show scope global | grep inet
```

Cockpit em `https://<ip-do-servidor>:9090` — aceitar o certificado self-signed.
Se não subir:

```bash
sudo systemctl enable --now cockpit.socket
```

### SSH por chave

No **cliente** (meu desktop), gerar a chave — Enter em tudo:

```bash
ssh-keygen -t ed25519 -C "$(whoami)@$(hostname)"
cat ~/.ssh/id_ed25519.pub
```

Colar no servidor. O jeito do vídeo, pelo terminal do Cockpit:

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys      # cola a pubkey, Ctrl+X Y Enter
chmod 600 ~/.ssh/authorized_keys
```

Atalho pro futuro (faz as três coisas de uma vez, do cliente):

```bash
ssh-copy-id <usuario>@<ip-do-servidor>
```

Testar antes de travar a senha:

```bash
ssh <usuario>@<ip-do-servidor>
```

Desligar login por senha:

```bash
sudo nano /etc/ssh/sshd_config.d/50-disable-password.conf
```

```
PasswordAuthentication no
```

```bash
sudo systemctl restart sshd
```

Verificar que a senha morreu (tem que dar `Permission denied (publickey,...)`):

```bash
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no <usuario>@<ip-do-servidor>
```

## 4. Expandir a raiz

Ver o tamanho atual e de onde vem a raiz:

```bash
df -h /
findmnt -no SOURCE /        # ex.: /dev/mapper/<vg>-<lv>
sudo vgs                    # coluna VFree = espaço livre no volume group
sudo lvs
```

Usar o caminho que o `findmnt` devolveu:

```bash
sudo lvextend -l +100%FREE <caminho-do-lv>
sudo xfs_growfs /
df -h /
```

## 5. Update e dnf.conf

```bash
sudo dnf upgrade -y
```

Ver o que já está configurado e as opções disponíveis:

```bash
cat /etc/dnf/dnf.conf
man dnf5.conf
```

```bash
sudo nano /etc/dnf/dnf.conf
```

```
[main]
defaultyes=True
max_parallel_downloads=10
fastestmirror=True
keepcache=True
```

## 6. Pacotes básicos

```bash
sudo dnf install -y curl wget git htop net-tools unzip util-linux-user nano
```

## 7. Hostname

```bash
hostnamectl                                  # estado atual
sudo hostnamectl set-hostname <nome-do-host>
```

## 8. Rede — IP fixo

Descobrir interface e nome da conexão:

```bash
nmcli device status                  # coluna DEVICE (ex.: enp1s0) e CONNECTION
nmcli connection show
```

Pegar os valores que o DHCP já entregou — são exatamente os que vou reusar no modo manual:

```bash
nmcli -f IP4 device show <iface>     # IP4.ADDRESS[1], IP4.GATEWAY, IP4.DNS[1]
```

Ou separado:

```bash
ip -4 addr show <iface>              # address/prefixo (ex.: 192.168.x.y/24)
ip route show default                # gateway (o IP depois de "via")
resolvectl dns                       # DNS em uso
```

Escolher o IP fixo **fora da faixa do DHCP do roteador** (ver no painel do roteador), mantendo
o mesmo /prefixo e gateway.

```bash
sudo nmcli connection modify "<nome-da-conexao>" \
    ipv4.method manual \
    ipv4.addresses <ip-escolhido>/<prefixo> \
    ipv4.gateway <gateway> \
    ipv4.dns "<dns1>,<dns2>"

sudo nmcli connection up "<nome-da-conexao>"
ip addr show <iface>
```

Voltar pra DHCP (e aí reservar o IP no roteador, que é a alternativa mais simples):

```bash
sudo nmcli connection modify "<nome-da-conexao>" \
    ipv4.method auto ipv4.addresses "" ipv4.gateway "" ipv4.dns ""
sudo nmcli connection up "<nome-da-conexao>"
```

Se a conexão não obedecer:

```bash
sudo systemctl restart NetworkManager
```

## 9. Firewall

Estado e regras atuais:

```bash
sudo firewall-cmd --state
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all
```

Descobrir o nome certo do serviço antes de liberar, e quais portas estão de fato escutando:

```bash
sudo firewall-cmd --get-services | tr ' ' '\n' | grep -i <termo>
sudo ss -tlnp
```

```bash
sudo firewall-cmd --permanent --add-service=cockpit
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-port=<porta>/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

## 10. Atualizações automáticas

No Fedora 44 é dnf5 — o pacote virou `dnf5-plugin-automatic` e o timer é `dnf5-automatic.timer`
(o `dnf-automatic.timer` do vídeo é o compat antigo).

```bash
sudo dnf install -y dnf-automatic        # resolve pra dnf5-plugin-automatic
systemctl list-unit-files 'dnf*automatic*'
rpm -ql dnf5-plugin-automatic | grep -E 'conf|timer'
```

Defaults ficam em `/usr/share/dnf5/dnf5-plugins/automatic.conf`; meus overrides:

```bash
sudo nano /etc/dnf/automatic.conf
```

```
[commands]
upgrade_type = security
apply_updates = yes
```

```bash
sudo systemctl enable --now dnf5-automatic.timer
systemctl list-timers 'dnf*'          # confirma o próximo disparo
journalctl -u dnf5-automatic.service  # ver o que rodou
man dnf5-automatic                    # emit_via, random_sleep, etc.
```

**← parei aqui.**

---

## Próximo passo: Docker (ainda não executado)

O guia usa Podman; vou de Docker. Repo oficial, não o `moby-engine` do Fedora:

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager addrepo --from-repofile=https://download.docker.com/linux/fedora/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
docker --version && docker compose version
sudo docker run --rm hello-world
```

Usar docker sem sudo (precisa relogar depois):

```bash
sudo usermod -aG docker "$USER"
newgrp docker
docker run --rm hello-world
```

## Fora do escopo por agora

- **ZFS** — não configurado, um disco só.
- **NVIDIA** — sem GPU dedicada neste Beelink.
