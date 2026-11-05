# Guía corta: VPN con WireGuard + respaldo y restauración

**Rocky** = servidor (`10.10.10.50`, tarjeta `enp0s8`) · **Kali** = cliente (`10.10.10.60`, tarjeta `eth1`)
Túnel: Rocky `10.200.200.1` · Kali `10.200.200.2`
SSH desde Windows: Rocky `-p 2223` · Kali `USUARIO@127.0.0.1 -p 2222`

---

## Fase 0: Preparar

**VirtualBox** (VMs apagadas):
- Adaptador 1: NAT, con reenvío de puertos (anfitrión `2223` a invitado `22` en Rocky, `2222` a `22` en Kali)
- Adaptador 2: Red interna `labnet`
- Rocky **9.8**, instalación **Server**, usuario marcado como **administrador**

```bash
# Rocky
sudo dnf update -y
sudo nmcli con add type ethernet ifname enp0s8 con-name labnet ipv4.method manual ipv4.addresses 10.10.10.50/24
sudo nmcli con up labnet
sudo systemctl enable --now sshd

# Kali
sudo apt update && sudo apt upgrade
sudo nmcli con add type ethernet ifname eth1 con-name labnet ipv4.method manual ipv4.addresses 10.10.10.60/24
sudo nmcli con up labnet
sudo systemctl enable --now ssh
ping -c 3 10.10.10.50        # debe responder
```

**Si te equivocas con la IP** (en Rocky o en Kali; cambia `labnet` por el nombre de tu conexión):

```bash
nmcli con show                                              # ver las conexiones y sus nombres
sudo nmcli con modify labnet ipv4.addresses 10.10.10.70/24  # cambiar la IP
sudo nmcli con up labnet                                    # aplicarla
ip a show enp0s8                                            # comprobar (en Kali: eth1)
```

- **Aviso "hay otra conexión con el nombre...":** hay dos conexiones con el mismo nombre. Borrar la que sobra con `sudo nmcli con delete uuid EL-UUID` (el UUID sale en `nmcli con show`) y repetir el cambio.
- **Si cambias la IP del servidor**, usar la nueva también en el `Endpoint` de Kali y en los `ping` y `scp`.
- **Errores de escritura típicos:** es `ipv4.method` y `ipv4.addresses` (con doble `s`).

---

## Fase 1: Crear

```bash
# Rocky: instalar y crear llaves
sudo dnf install -y epel-release
sudo dnf install -y wireguard-tools
wg genkey | sudo tee /etc/wireguard/server_private.key
sudo chmod 600 /etc/wireguard/server_private.key
sudo cat /etc/wireguard/server_private.key | wg pubkey | sudo tee /etc/wireguard/server_public.key

# Kali: instalar y crear llaves
sudo apt install -y wireguard
wg genkey | sudo tee /etc/wireguard/kali_private.key
sudo chmod 600 /etc/wireguard/kali_private.key
sudo cat /etc/wireguard/kali_private.key | wg pubkey | sudo tee /etc/wireguard/kali_public.key
```

Copiar cada llave **pública** a la otra máquina (`sudo cat .../..._public.key`). La privada nunca se comparte.

```bash
# Rocky: configuración
PRIV=$(sudo cat /etc/wireguard/server_private.key)
sudo tee /etc/wireguard/wg0.conf > /dev/null <<EOF
[Interface]
Address = 10.200.200.1/24
ListenPort = 51820
PrivateKey = $PRIV

[Peer]
PublicKey = <PÚBLICA DE KALI>
AllowedIPs = 10.200.200.2/32
EOF
sudo chmod 600 /etc/wireguard/wg0.conf
sudo firewall-cmd --permanent --add-port=51820/udp
sudo firewall-cmd --reload
sudo systemctl enable --now wg-quick@wg0

# Kali: configuración
PRIV=$(sudo cat /etc/wireguard/kali_private.key)
sudo tee /etc/wireguard/wg0.conf > /dev/null <<EOF
[Interface]
Address = 10.200.200.2/24
PrivateKey = $PRIV

[Peer]
PublicKey = <PÚBLICA DE ROCKY>
Endpoint = 10.10.10.50:51820
AllowedIPs = 10.200.200.1/32
PersistentKeepalive = 25
EOF
sudo chmod 600 /etc/wireguard/wg0.conf
sudo systemctl enable --now wg-quick@wg0

# Kali: probar
ping -c 3 10.200.200.1       # debe responder
sudo wg show                 # debe decir "latest handshake"
```

**Hardening (Rocky):**

```bash
getenforce                                    # debe decir Enforcing
sudo passwd -l root
sudo firewall-cmd --permanent --remove-service=cockpit
sudo firewall-cmd --reload
sudo firewall-cmd --list-all                  # solo ssh, dhcpv6-client y 51820/udp
sudo dnf install -y fail2ban
sudo tee /etc/fail2ban/jail.local > /dev/null <<EOF
[sshd]
enabled = true
maxretry = 5
bantime = 1h
EOF
sudo systemctl enable --now fail2ban
```

---

## Fase 2: Verificar

```bash
# Kali
nmap 10.10.10.50             # solo debe verse el puerto 22
```

Wireshark en Kali, capturando en `eth1` mientras haces `ping 10.200.200.1`: se ve `WireGuard` y el contenido ilegible.

---

## Fase 3: Respaldar

```bash
# Rocky
sudo tar czf /tmp/respaldo-rocky.tar.gz /etc/wireguard /etc/fail2ban/jail.local
sudo chmod 600 /tmp/respaldo-rocky.tar.gz
sudo gpg --symmetric --cipher-algo AES256 --pinentry-mode loopback /tmp/respaldo-rocky.tar.gz
sudo chmod 600 /tmp/respaldo-rocky.tar.gz.gpg
sudo chown USUARIO:USUARIO /tmp/respaldo-rocky.tar.gz.gpg
sha256sum /tmp/respaldo-rocky.tar.gz.gpg

# Kali
scp USUARIO@10.10.10.50:/tmp/respaldo-rocky.tar.gz.gpg ~/
sha256sum ~/respaldo-rocky.tar.gz.gpg        # debe ser igual a la de Rocky
gpg --pinentry-mode loopback --decrypt ~/respaldo-rocky.tar.gz.gpg | tar tz    # debe listar los archivos

# Rocky: borrar el original sin cifrar
sudo shred -u /tmp/respaldo-rocky.tar.gz
```

La contraseña del cifrado no se puede recuperar si se pierde.

---

## Fase 4: Destruir

VirtualBox: apagar la VM de Rocky, clic derecho, **Quitar**, **Eliminar todos los archivos**.

---

## Fase 5: Restaurar

Crear la VM nueva de Rocky 9.8 (administrador, `labnet` y reenvío de puertos), y luego:

```bash
# Rocky
sudo nmcli con add type ethernet ifname enp0s8 con-name labnet ipv4.method manual ipv4.addresses 10.10.10.50/24
sudo nmcli con up labnet
sudo systemctl enable --now sshd
sudo dnf update -y
sudo dnf install -y epel-release
sudo dnf install -y wireguard-tools fail2ban

# Kali
scp ~/respaldo-rocky.tar.gz.gpg USUARIO@10.10.10.50:~/

# Rocky
gpg --pinentry-mode loopback --output respaldo-rocky.tar.gz --decrypt respaldo-rocky.tar.gz.gpg
sudo tar xzf respaldo-rocky.tar.gz -C /
shred -u respaldo-rocky.tar.gz           # borrar la copia descifrada
sudo firewall-cmd --permanent --add-port=51820/udp
sudo firewall-cmd --permanent --remove-service=cockpit   # el respaldo no guarda el firewall
sudo firewall-cmd --reload
sudo passwd -l root
sudo systemctl enable --now fail2ban
sudo systemctl enable --now wg-quick@wg0

# Kali: verificar
ping -c 3 10.200.200.1
sudo wg show                 # debe aparecer la MISMA llave pública del servidor que antes
```

---

## Si algo falla

- **`not in the sudoers file`:** el usuario no es administrador (reinstalar marcando la casilla, o `usermod -aG wheel usuario` desde root).
- **El ping entre máquinas falla:** revisar con `ip a` que cada una tenga su IP de `labnet`.
- **`gpg` no pide contraseña:** agregar `--pinentry-mode loopback`.
- **`tar -C` da error:** falta el `/` al final (`-C /`).
- **Aviso de identidad del servidor cambiada:** `ssh-keygen -R "[127.0.0.1]:2223"` en Windows.
- **Antes de ejecutar un comando:** revisar el prompt, que diga en qué máquina estás.
