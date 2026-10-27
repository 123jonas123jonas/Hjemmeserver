# Kommandoliste

Tilbake til [README](../README.md)

> [!TIP]
> **Lim inn i PowerShell**
> Høyreklikk i vinduet, eller **Ctrl+V** / **Ctrl+Shift+V**. Kopier med kopier-knappen på kodeblokken.
> Du kan bare lime inn når du er koblet til med SSH fra PC-en, ikke når du skriver rett på laptopen.

## De tre du bruker mest

Virker etter at du har kjørt [Installer snarveiene (én gang)](#installer-snarveiene-én-gang).

| Skriv | Viser |
|---|---|
| `serverstatus` | Alt på én gang: batteri, IP-er, disk, minne, temperatur, containere. Vises også automatisk når du logger inn med SSH. |
| `batteri` | Bare batteriprosent |
| `minip` | Hjemme-IP og Tailscale-IP |

Uten snarveiene:

```bash
cat /sys/class/power_supply/BAT0/capacity    # batteriprosent (prøv BAT1 hvis BAT0 ikke finnes)
cat /sys/class/power_supply/BAT0/status      # lader / går på batteri
hostname -I                                  # alle IP-er
tailscale ip -4                              # Tailscale-IP-en
```

## Installer snarveiene (én gang)

Koble til serveren med SSH og lim inn hele blokken. Den lager kommandoen `serverstatus`, snarveiene `batteri` og `minip`, og gjør at statusen vises hver gang du logger inn med SSH.

```bash
sudo tee /usr/local/bin/serverstatus > /dev/null << 'EOF'
#!/usr/bin/env bash
# serverstatus – kort oversikt over hjemmeserveren

linje() { printf '%-12s %s\n' "$1" "$2"; }

echo "────────── Hjemmeserver ──────────"

for b in /sys/class/power_supply/BAT*; do
  [ -e "$b/capacity" ] || continue
  s=$(cat "$b/status" 2>/dev/null)
  case "$s" in
    Charging) s="lader" ;;
    Discharging) s="GÅR PÅ BATTERI – sjekk strømmen" ;;
    Full) s="fullt" ;;
    "Not charging") s="plugget i, lader ikke" ;;
  esac
  linje "Batteri:" "$(cat "$b/capacity")% ($s)"
done

lan=$(ip -4 -o addr show scope global 2>/dev/null | awk '$2 !~ /^(docker|br-|veth|tailscale)/ {split($4,a,"/"); printf "%s (%s)  ", a[1], $2}')
linje "Hjemme-IP:" "${lan:-INGEN – serveren har mistet nettet}"
ts=$(tailscale ip -4 2>/dev/null | head -n1)
linje "Tailscale:" "${ts:-ikke tilkoblet}"

linje "Oppe i:" "$(uptime -p | sed 's/^up //')"
linje "Disk /:" "$(df -h / | awk 'NR==2 {print $3 " av " $2 " brukt (" $5 ")"}')"
if mountpoint -q /mnt/ssd 2>/dev/null; then
  linje "SSD:" "$(df -h /mnt/ssd | awk 'NR==2 {print $3 " av " $2 " brukt (" $5 ")"}')"
fi
[ -f /mnt/ssd/backup/sist-kjort.txt ] && linje "Backup:" "sist $(cat /mnt/ssd/backup/sist-kjort.txt)"
linje "Minne:" "$(free -h | awk '/^Mem:/ {print $3 " av " $2 " brukt"}')"

t=$(cat /sys/class/thermal/thermal_zone*/temp 2>/dev/null | sort -n | tail -n1)
[ -n "$t" ] && linje "Temp:" "$((t / 1000)) °C"

echo "Containere:"
if out=$(docker ps --format '{{.Names}}: {{.Status}}' 2>/dev/null) || out=$(sudo -n docker ps --format '{{.Names}}: {{.Status}}' 2>/dev/null); then
  if [ -n "$out" ]; then echo "$out" | sed 's/^/  /'; else echo "  ingen kjører"; fi
else
  echo "  (skriv 'sudo serverstatus' for å se Docker)"
fi
EOF
sudo chmod +x /usr/local/bin/serverstatus

grep -q '>>> hjemmeserver >>>' ~/.bashrc || cat >> ~/.bashrc << 'EOF'

# >>> hjemmeserver >>>
batteri() { for b in /sys/class/power_supply/BAT*/capacity; do [ -e "$b" ] && echo "$(cat "$b")%"; done; }
minip() {
  echo "Hjemme:    $(ip -4 -o addr show scope global | awk '$2 !~ /^(docker|br-|veth|tailscale)/ {split($4,a,"/"); print a[1]}' | head -n1)"
  echo "Tailscale: $(tailscale ip -4 2>/dev/null | head -n1)"
}
[ -n "$SSH_CONNECTION" ] && serverstatus
# <<< hjemmeserver <<<
EOF
source ~/.bashrc
serverstatus
```

Står det "skriv 'sudo serverstatus' for å se Docker" under Containere, er brukeren din ikke i docker-gruppa. Fiks det én gang med `sudo usermod -aG docker jonas`, logg ut (`exit`) og inn igjen.

## Koble til med ett ord

Gjør dette én gang i PowerShell **på PC-en**.

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.ssh" | Out-Null
Add-Content "$env:USERPROFILE\.ssh\config" "`nHost server`n    HostName 100.x.x.x`n    User jonas"
```

Fra nå skriver du bare:

```powershell
ssh server
```

### Slippe passordet (valgfritt)

Lager en nøkkel på PC-en og legger den på serveren. Etterpå logger `ssh server` inn uten passord. Trykk Enter på alle spørsmålene fra `ssh-keygen`.

```powershell
ssh-keygen -t ed25519
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh server "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

`sudo` spør fortsatt om Linux-passordet. Det er meningen.

## Tastatur i terminalen

| Tast | Gjør |
|---|---|
| **Pil opp** | Forrige kommando |
| **Ctrl+R** | Søk i gamle kommandoer (skriv f.eks. `docker`) |
| **Tab** | Fullfører fil- og mappenavn |
| **Ctrl+C** | Avbryt det som kjører |
| **Ctrl+L** | Tøm skjermen |
| `exit` | Logg ut av serveren |

I `nano` (teksteditoren): **Ctrl+O** + Enter lagrer, **Ctrl+X** avslutter.

## Batteri og strøm

```bash
batteri                                       # prosent
cat /sys/class/power_supply/BAT*/status       # Charging / Discharging / Full / Not charging
sudo reboot                                   # start på nytt
sudo shutdown now                             # skru av
```

Batteriet fungerer som en innebygd nødstrøm: går strømmen, fortsetter serveren på batteri en stund. `serverstatus` sier fra med "GÅR PÅ BATTERI".

## Nettverk og IP

```bash
minip                       # hjemme-IP og Tailscale-IP
hostname -I                 # alle IP-er (172.17.0.1 er Docker, se bort fra den)
ip -4 a                     # IP per nettverkskort (en... = kabel, wl... = Wi-Fi)
networkctl                  # status på nettverkskortene (routable = ok)
sudo netplan apply          # les inn nettverksinnstillingene på nytt
ping -c 3 8.8.8.8           # har serveren internett?
```

## Fra PC-en (PowerShell)

```powershell
ssh server                                   # koble til (etter oppsett over)
ssh jonas@100.x.x.x                          # koble til uten snarvei
Test-NetConnection 100.x.x.x -Port 22        # svarer serveren på SSH?
ipconfig                                     # PC-ens egne IP-er
scp .\fil.txt server:~/                      # kopier en fil fra PC til serveren
scp server:~/fil.txt .                       # kopier en fil fra serveren til PC
```

Z:-disken (i CMD, ikke PowerShell):

```cmd
net use Z: /delete
cmdkey /add:100.x.x.x /user:jonas /pass
net use Z: \\100.x.x.x\Photos /persistent:yes
```

## Status og ressurser

```bash
serverstatus                # alt på én gang
uptime                      # hvor lenge den har vært oppe
df -h                       # ledig diskplass
free -h                     # minne
top                         # hva bruker CPU nå (q for å avslutte)
du -sh /data/photos         # hvor mye plass bildene tar
```

## Docker

```bash
docker ps                                   # hvilke containere kjører
docker ps -a                                # alle, også de som har stoppet
docker logs homeassistant --tail 30         # siste linjer fra loggen
docker restart homeassistant                # start én container på nytt
cd ~/homeassistant && docker compose up -d  # start alt i den mappa
cd ~/homeassistant && docker compose down   # stopp alt i den mappa
docker stats                                # CPU og minne per container (Ctrl+C for å avslutte)
docker image prune -f                       # slett gamle nedlastinger og frigjør plass
```

Får du `permission denied`: sett `sudo` foran, eller kjør `sudo usermod -aG docker jonas` én gang og logg inn på nytt.

**Oppdatere en tjeneste** (henter nyeste versjon, beholder innstillingene):

```bash
cd ~/homeassistant && docker compose pull && docker compose up -d
cd ~/homepage && docker compose pull && docker compose up -d
docker image prune -f
```

## Tailscale

```bash
tailscale status                              # hvilke enheter er koblet til
tailscale ip -4                               # serverens Tailscale-IP
sudo tailscale up                             # logg inn / koble til på nytt
sudo tailscale file get /data/photos          # hent filer sendt fra mobilen via Tailscale
sudo chown -R jonas:jonas /data/photos        # gjør deg til eier av filene etterpå
```

## Samba (Z:-disken)

```bash
sudo systemctl status smbd          # kjører Samba?
sudo systemctl restart smbd         # start Samba på nytt
sudo smbpasswd -a jonas             # sett nytt Samba-passord
testparm                            # sjekk smb.conf for feil
sudo nano /etc/samba/smb.conf       # rediger innstillingene
```

## Brannmur (ufw)

```bash
sudo ufw status numbered            # hvilke porter er åpne
sudo ufw allow 8123/tcp             # åpne en port
sudo ufw allow in on tailscale0     # slipp inn alt som kommer via Tailscale
sudo ufw delete 5                   # fjern regel nummer 5 fra lista
```

## Filer og mapper

```bash
ls -lt /data/photos | head          # nyeste filer først
cd ~/homepage                       # gå til en mappe (~ = /home/jonas)
cd ..                               # en mappe opp
pwd                                 # hvor er jeg
nano filnavn                        # rediger en fil
cp fil kopi                         # kopier
mv fil nytt-navn                    # flytt eller gi nytt navn
sudo chown -R jonas:jonas MAPPE     # gjør deg til eier
```

Finn bilder som har kommet inn den siste timen, uansett hvor:

```bash
sudo find / \( -path /proc -o -path /sys -o -path /var/lib/docker \) -prune -o -type f -cmin -60 \( -iname '*.jpg' -o -iname '*.jpeg' -o -iname '*.heic' -o -iname '*.png' -o -iname '*.mp4' -o -iname '*.mov' \) -print
```

## Vedlikehold

```bash
sudo apt update && sudo apt upgrade -y      # oppdater Ubuntu
sudo reboot                                 # start på nytt etterpå hvis den ber om det
timedatectl                                 # sjekk klokke og tidssone
sudo timedatectl set-timezone Europe/Oslo   # sett norsk tid (viktig for planlagte jobber)
journalctl -b -p err                        # feilmeldinger siden siste oppstart
```

Ubuntu installerer sikkerhetsoppdateringer automatisk i bakgrunnen, men en manuell runde en gang i måneden er lurt.
