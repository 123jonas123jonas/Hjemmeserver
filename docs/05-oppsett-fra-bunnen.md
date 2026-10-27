# Oppsett fra bunnen

Tilbake til [README](../README.md)

Alt som er gjort, i rekkefølge, med kommandoene. Fungerer både som gjennomgang av hva vi har gjort og som oppskrift hvis laptopen dør og alt må bygges på nytt.

> [!NOTE]
> **Hvis du bygger på nytt**
> Bildene og innstillingene ligger bare på serveren. Har du backup på SSD-en ([SSD og backup](06-ssd-og-backup.md)), kopierer du `/data/photos`, `~/homeassistant` og `~/homepage` tilbake etter steg 9, og kjører `docker compose up -d` i hver mappe.

## 1. Installere Ubuntu Server

- Ubuntu Server 24.04 LTS, installert fra USB-pinne.
- Bruker: `jonas`.
- Huk av for **OpenSSH server** under installasjonen hvis du blir spurt (sparer steg 4).

## 2. Skru av dvale

En laptop sovner av seg selv, og da forsvinner den fra nettet.

```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

Ekstra sikring så lokket kan være lukket:

```bash
sudo sed -i 's/^#\?HandleLidSwitch=.*/HandleLidSwitch=ignore/; s/^#\?HandleLidSwitchExternalPower=.*/HandleLidSwitchExternalPower=ignore/; s/^#\?HandleLidSwitchDocked=.*/HandleLidSwitchDocked=ignore/' /etc/systemd/logind.conf
sudo reboot
```

## 3. Nettverk med kabel

Wi-Fi på gamle laptoper er ustabilt, så serveren står med kabel til **hovedruteren**.

```bash
ip link                                              # finn kortet som starter med en, f.eks. enp3s0
sudo netplan set ethernets.enp3s0.dhcp4=true         # bytt enp3s0 med ditt navn
sudo netplan apply
hostname -I                                          # skal vise 10.0.0.x
```

Reserver IP-en i ruteren (DHCP-reservasjon) så den ikke endrer seg. Står i ruterens innstillinger, ofte under LAN / DHCP / tilkoblede enheter.

## 4. SSH

```bash
sudo apt install openssh-server -y
sudo systemctl enable --now ssh
sudo ufw allow ssh
sudo ufw enable
```

`ufw allow ssh` må komme før `ufw enable`, ellers stenger brannmuren deg ute.

Fra PC-en: `ssh jonas@10.0.0.x`. Senere satt opp med Tailscale og snarvei, se [Koble til med ett ord](02-kommandoliste.md#koble-til-med-ett-ord).

## 5. Samba med bildemappa

```bash
sudo apt install samba -y
sudo mkdir -p /data/photos
sudo chown jonas:jonas /data/photos
sudo tee -a /etc/samba/smb.conf > /dev/null << 'EOF'

[Photos]
   path = /data/photos
   valid users = jonas
   read only = no
   browseable = yes
EOF
sudo smbpasswd -a jonas          # velg Samba-passord
sudo systemctl restart smbd
sudo ufw allow samba
```

På PC-en (CMD). Bruk hjemme-IP-en til Tailscale er satt opp, og bytt til 100.x.x.x etterpå:

```cmd
cmdkey /add:10.0.0.x /user:jonas /pass
net use Z: \\10.0.0.x\Photos /persistent:yes
```

## 6. Docker

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker jonas
exit
```

Logg inn igjen etter `exit`, så slipper du `sudo` foran `docker`.

## 7. Home Assistant og ESPHome

```bash
mkdir -p ~/homeassistant && cd ~/homeassistant
cat > compose.yaml << 'EOF'
services:
  homeassistant:
    container_name: homeassistant
    image: ghcr.io/home-assistant/home-assistant:stable
    volumes:
      - ./config:/config
      - /etc/localtime:/etc/localtime:ro
      - /run/dbus:/run/dbus:ro
    environment:
      - TZ=Europe/Oslo
    restart: unless-stopped
    privileged: true
    network_mode: host

  esphome:
    container_name: esphome
    image: ghcr.io/esphome/esphome
    volumes:
      - ./esphome:/config
      - /etc/localtime:/etc/localtime:ro
    restart: unless-stopped
    network_mode: host
EOF
docker compose up -d
sudo ufw allow 8123/tcp
sudo ufw allow 6052/tcp
```

Første gang: åpne `http://IP:8123`, lag bruker. Sett hjemmeposisjon omtrentlig (f.eks. nærmeste tettsted) hvis du ikke vil oppgi adressen. La analyse-boksene stå av.

## 8. Tailscale

På serveren:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo systemctl enable --now tailscaled
sudo tailscale up            # kopier lenka, åpne på PC-en, logg inn
tailscale ip -4              # serverens faste 100.x-adresse
sudo ufw allow in on tailscale0
```

Er det vanskelig å kopiere lenka: `sudo tailscale up --qr` og skann med mobilen.

På PC-en og mobilen: installer Tailscale, logg inn med **samme konto**.

I admin-panelet: *Disable key expiry* på serveren.

## 9. Homepage

```bash
mkdir -p ~/homepage/config && cd ~/homepage
TSIP=$(tailscale ip -4)

cat > compose.yaml << EOF
services:
  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    container_name: homepage
    ports:
      - 3000:3000
    volumes:
      - ./config:/app/config
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      HOMEPAGE_ALLOWED_HOSTS: ${TSIP}:3000
    restart: unless-stopped
EOF

cat > config/settings.yaml << EOF
title: Hjemmeserver
theme: dark
EOF

cat > config/docker.yaml << EOF
my-docker:
  socket: /var/run/docker.sock
EOF

cat > config/services.yaml << EOF
- Smarthus:
    - Home Assistant:
        href: http://${TSIP}:8123
        icon: home-assistant.png
        description: Smarthus
        server: my-docker
        container: homeassistant
    - ESPHome:
        href: http://${TSIP}:6052
        icon: esphome.png
        description: Sensorer
        server: my-docker
        container: esphome
EOF

cat > config/widgets.yaml << EOF
- resources:
    cpu: true
    memory: true
    disk: /
- datetime:
    text_size: xl
    format:
      timeStyle: short
EOF

cat > config/bookmarks.yaml << EOF
- Admin:
    - Tailscale:
        - abbr: TS
          href: https://login.tailscale.com/admin/machines
EOF

sudo ufw allow 3000/tcp
docker compose up -d
echo "Åpne: http://$TSIP:3000"
```

## 10. Snarveier

`serverstatus`, `batteri` og `minip`: se [Installer snarveiene (én gang)](02-kommandoliste.md#installer-snarveiene-én-gang).

## Valgt bort

- **Immich** (automatisk bildebackup fra mobilen) – ikke ønsket. Alternativ hvis behovet kommer: Syncthing, se [Videre arbeid](07-videre-arbeid.md).
