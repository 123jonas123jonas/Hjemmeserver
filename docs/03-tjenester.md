# Tjenester

Tilbake til [README](../README.md) · Kommandoer i [Kommandoliste](02-kommandoliste.md)

Én seksjon per ting som kjører på serveren.

## SSH

| | |
|---|---|
| Hva | Styre serveren fra PowerShell på PC-en |
| Port | 22 |
| Koble til | `ssh server` (snarvei) eller `ssh jonas@100.x.x.x` |
| Passord | Linux-passordet |
| Sjekk | `sudo systemctl status ssh` |

## Samba (bildemappa)

| | |
|---|---|
| Hva | Deler `/data/photos` på nettet |
| Delingsnavn | `Photos` (bildene) og `SSD` (den eksterne SSD-en) |
| Port | 445 |
| På PC-en | Disk **Z:** = `\\100.x.x.x\Photos` og **Y:** = `\\100.x.x.x\SSD` |
| På mobilen | iPhone: Filer-appen → Koble til server → `smb://100.x.x.x/Photos`. Android: CX File Explorer → Network → SMB |
| Bruker | `jonas` med **Samba-passordet** (ikke Linux-passordet) |
| Innstillinger | `/etc/samba/smb.conf`, nederst under `[Photos]` |
| Sjekk | `sudo systemctl status smbd` |

Delingen i `smb.conf` ser slik ut:

```ini
[Photos]
   path = /data/photos
   valid users = jonas
   read only = no
   browseable = yes
```

## Home Assistant

| | |
|---|---|
| Hva | Smarthus: styre og overvåke enheter, lage automasjoner, vise sensordata |
| Adresse | `http://100.x.x.x:8123` |
| Kjører i | Docker, container `homeassistant` |
| Mappe | `~/homeassistant` (oppskrift i `compose.yaml`, data i `config/`) |
| Oppdatere | `cd ~/homeassistant && docker compose pull && docker compose up -d` |
| Logg | `docker logs homeassistant --tail 30` |

Alt er lokalt. Ingenting sendes ut med mindre du selv slår på Home Assistant Cloud (Nabu Casa) eller logger inn med GitHub for HACS.

### HACS

Butikken for ekstra integrasjoner laget av andre brukere. Installert med `docker exec -it homeassistant bash -c "wget -O - https://get.hacs.xyz | bash -"` og logget inn med GitHub. Fra Home Assistant 2026.11 er den innebygd som *Marketplace*, og det du har installert flyttes over automatisk ved oppdatering.

### Tapo C460-kameraet

| | |
|---|---|
| Integrasjon | *Tapo: Cameras Control* (fra HACS) |
| Kameraets IP | `10.0.0.y` (må være på hovedruteren) |
| Port | 443 |
| Bildet | Bruk **SD/HD Stream (Direct)**. De vanlige HD/SD Stream virker ikke, siden kameraet ikke har RTSP. |
| Innstillinger | *Use Stream from Home Assistant* og webhooks er av |
| Virker ikke | Bevegelsessensor og webhooks (kameraet mangler ONVIF) |
| Endre valg | Innstillinger → Enheter og tjenester → Tapo: Cameras Control → Konfigurer, så `docker restart homeassistant` |

Kameraet går på batteri og sover det meste av tiden. Live-bilde vekker det og bruker strøm, så ikke ha et live-kort åpent hele tiden. Får du «Connection failed», åpne live-bildet i Tapo-appen for å vekke det og prøv igjen.

I Tapo-appen må *Meg → Tredjepartskompatibilitet* være på.

## ESPHome

| | |
|---|---|
| Hva | Lager programvare til D1 Mini / ESP-kort, så sensorer dukker opp i Home Assistant |
| Adresse | `http://100.x.x.x:6052` |
| Kjører i | Docker, container `esphome` |
| Mappe | `~/homeassistant/esphome` (YAML-filene for hvert kort) |
| Oppdatere | Samme som Home Assistant (samme `compose.yaml`) |

## Homepage (dashbordet)

| | |
|---|---|
| Hva | Startside med knapper til alt, viser om containerne kjører, CPU, minne og disk |
| Adresse | `http://100.x.x.x:3000` |
| Kjører i | Docker, container `homepage` |
| Mappe | `~/homepage` (oppskrift i `compose.yaml`, innstillinger i `config/`) |
| Oppdatere | `cd ~/homepage && docker compose pull && docker compose up -d` |

Filene i `~/homepage/config`:

| Fil | Styrer |
|---|---|
| `services.yaml` | Knappene (tjenestene) |
| `bookmarks.yaml` | Lenker (f.eks. Tailscale-admin) |
| `widgets.yaml` | Øverste linje (CPU, minne, disk, klokke) |
| `settings.yaml` | Tittel og tema |
| `docker.yaml` | Kobling til Docker, så den ser om containerne kjører |

**Legge til en ny knapp:** `nano ~/homepage/config/services.yaml` og legg til under riktig gruppe. Siden oppdaterer seg selv når du lagrer.

```yaml
    - Navn på tjenesten:
        href: http://100.x.x.x:PORT
        icon: navn.png
        description: Kort beskrivelse
        server: my-docker
        container: containernavn
```

Innrykket (mellomrommene) må være nøyaktig likt som de andre knappene. YAML er nøye på det.

Homepage godtar bare adresser som står i `HOMEPAGE_ALLOWED_HOSTS` i `compose.yaml`. Den er satt til Tailscale-IP-en med `:3000`. Vil du åpne den via hjemme-IP-en også, legg den til med komma: `100.x.x.x:3000,10.0.0.x:3000`, og kjør `docker compose up -d`.

## Tailscale

| | |
|---|---|
| Hva | Privat, kryptert nettverk mellom dine enheter |
| Serverens IP | `100.x.x.x` (fast) – `tailscale ip -4` |
| Admin | login.tailscale.com/admin/machines |
| Enheter | Server, PC, (mobil) – alle logget inn med samme konto |
| Sjekk | `tailscale status` |

**Sende filer fra mobilen:** Del → Tailscale → velg serveren. På Linux havner filene i en venteplass til de hentes. Tjenesten `tailscale-mottak` henter dem automatisk inn i `/data/photos` med en gang de kommer. Sjekk at den kjører med `systemctl status tailscale-mottak`.

Er den ikke satt opp, lim inn dette én gang:

```bash
sudo tailscale set --operator=jonas
sudo tee /etc/systemd/system/tailscale-mottak.service > /dev/null << 'EOF'
[Unit]
Description=Hent filer sendt via Tailscale til /data/photos
After=tailscaled.service
Wants=tailscaled.service

[Service]
User=jonas
ExecStart=/usr/bin/tailscale file get --loop --conflict=rename /data/photos
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now tailscale-mottak
```

**På Windows** har Tailscale ikke noe vindu. Ikonet ligger nede til høyre ved klokka (under pila ^).

Tailscale logger ut enheter automatisk etter 180 dager. Slå av for serveren: Machines → ⋯ ved serveren → *Disable key expiry*.

## Brannmur (ufw)

| | |
|---|---|
| Hva | Blokkerer alt inn til serveren unntatt åpne porter |
| Åpne nå | 22 (SSH), Samba, 8123, 6052, 3000 |
| Sjekk | `sudo ufw status numbered` |

Ny tjeneste med ny port → `sudo ufw allow PORT/tcp`. Alternativt `sudo ufw allow in on tailscale0`, som slipper inn alt som kommer via Tailscale (bare dine egne enheter).

## Ubuntu Server

| | |
|---|---|
| Versjon | Ubuntu Server 24.04 LTS |
| Nettverk | Kabel til hovedruteren, styrt av netplan (`/etc/netplan/`) |
| Dvale | Skrudd av med `systemctl mask` – serveren sovner aldri, heller ikke med lokket lukket |
| Bruker | `jonas` (har `sudo`) |
