# Feilsøking

Tilbake til [README](../README.md) · Kommandoer i [Kommandoliste](02-kommandoliste.md)

Finn symptomet, gjør stegene i rekkefølge.

## SSH timer ut eller sier "host unreachable"

1. **Er Tailscale på på PC-en?** Ikonet nede til høyre ved klokka skal vise at du er koblet til. Prøv `ssh server` igjen.
2. **Er serveren på?** Lyser strømlampa? Er laptopen i strøm? Se [Serveren svarer ikke i det hele tatt](#serveren-svarer-ikke-i-det-hele-tatt).
3. **Test fra PC-en:** `Test-NetConnection 100.x.x.x -Port 22`
   - `TcpTestSucceeded : True` → nettet er ok, prøv `ssh` igjen.
   - `False` → gå til laptopen og logg inn direkte. Kjør `minip`.
4. **Bruker du hjemme-IP-en (10.0.0.x)?** Den virker bare når PC-en er på hovedruteren hjemme. Bruk Tailscale-IP-en i stedet.

## Serveren viser bare 172.17.0.1 / har mistet nettet

`172.17.0.1` er Docker sin interne adresse. Står den alene, har serveren ingen ordentlig nettverkstilkobling.

1. Sjekk at kabelen sitter i laptopen og i **hovedruteren** (ikke den andre ruteren). Lyser lampen ved porten?
2. Kjør `networkctl` og se på linja for kortet som starter med `en`:

| Står det | Betyr | Gjør |
|---|---|---|
| `routable` | Alt ok | Kjør `minip`, IP-en skal være der |
| `no-carrier` | Ingen signal i kabelen | Sjekk kabelen, prøv en annen port i ruteren |
| `configuring` | Venter på IP | Vent 30 sekunder, kjør `minip` igjen |
| `unmanaged` | Kortet er ikke satt opp | `sudo netplan set ethernets.NAVN.dhcp4=true` og `sudo netplan apply` (NAVN fra lista, f.eks. `enp3s0`) |

3. Hjelper ingenting: `sudo netplan apply`, så `sudo reboot`.

## Oppstarten henger på "systemd-networkd-wait-online"

Normalt når serveren ikke får nett under oppstart. Den venter i opptil to minutter og fortsetter så. Vent, logg inn, og følg punktet over.

## En nettside (Home Assistant, Homepage, ESPHome) laster ikke

1. Skriv `http://` (ikke `https://`) og riktig port, f.eks. `http://100.x.x.x:8123`.
2. Kjører containeren? `docker ps` – den skal stå med `Up`. Hvis ikke: `cd ~/homeassistant && docker compose up -d` (eller `~/homepage`).
3. Er porten åpen i brannmuren? `sudo ufw status`. Mangler den: `sudo ufw allow PORT/tcp`.
4. Rett etter oppstart kan det ta et par minutter før siden svarer.
5. Fortsatt ikke: `docker logs CONTAINERNAVN --tail 30` og les feilmeldingen.

## Homepage sier "Host validation failed"

Adressen i nettleseren står ikke i `HOMEPAGE_ALLOWED_HOSTS`. Bruk `http://100.x.x.x:3000`, eller legg til adressen du bruker i `~/homepage/compose.yaml` (komma mellom flere) og kjør `cd ~/homepage && docker compose up -d`. Se [Homepage (dashbordet)](03-tjenester.md#homepage-dashbordet).

## Z:-disken er borte eller frakoblet

1. Er Tailscale på på PC-en?
2. Koble den til på nytt i CMD:
   ```cmd
   net use Z: /delete
   cmdkey /add:100.x.x.x /user:jonas /pass
   net use Z: \\100.x.x.x\Photos /persistent:yes
   ```
3. Feilkoder fra `net use`:

| Feil | Betyr | Fiks |
|---|---|---|
| 53 | PC-en når ikke serveren | Sjekk Tailscale og at serveren er på |
| 67 | Feil delingsnavn | Skal være nøyaktig `Photos` |
| 86 / 1326 | Feil brukernavn/passord | Bruk Samba-passordet. Glemt? `sudo smbpasswd -a jonas` |
| 1219 | Gammel innlogging lagret | `net use * /delete` og prøv igjen |

4. På serveren: `sudo systemctl status smbd`. Står den ikke `active (running)`: `sudo systemctl restart smbd`.

## Nye filer vises ikke på Z:

- Trykk **F5** i Utforsker.
- Er filene eid av root? `sudo chown -R jonas:jonas /data/photos`.

## Bilder sendt via Tailscale dukker ikke opp

På Linux lagres de i en venteplass til de hentes. Tjenesten `tailscale-mottak` skal gjøre det automatisk (se [Tailscale](03-tjenester.md#tailscale)). Kjører den ikke, hent dem selv og fiks rettighetene:

```bash
sudo tailscale file get --conflict=rename /data/photos
sudo chown -R jonas:jonas /data/photos
sudo chmod -R u+rwX,go+rX /data/photos
```

Ligger de i `/data/photos`, men vises ikke på Z:, er det nesten alltid rettighetene. Kjør de to siste linjene og trykk F5.

Sjekk venteplassen: `sudo ls -R /var/lib/tailscale/files/`. Står det bare en mappe med et langt tall og ingenting under, er den tom. Da er filene enten hentet allerede, eller sendt til feil enhet (PC-en legger dem i Nedlastinger).

## Får ikke limt inn i PowerShell

- Kopier med kopier-knappen på kodeblokken.
- Prøv høyreklikk, **Ctrl+V**, **Ctrl+Shift+V** eller **Shift+Insert**.
- Gammelt vindu: ikonet øverst til venstre → Rediger → Lim inn.
- Skriver du rett på laptopen, går det ikke. Koble til med `ssh server` fra PC-en først.

## "permission denied" på docker

Sett `sudo` foran, eller fiks det for godt: `sudo usermod -aG docker jonas`, `exit`, og logg inn igjen.

## Tailscale-appen på Windows åpner seg ikke

Den har ikke noe vindu. Ikonet ligger ved klokka nede til høyre (under pila **^**). Klikk på det for å logge inn. Finner du det ikke: `tailscale up` i PowerShell.

## Serveren svarer ikke i det hele tatt

1. Gå til laptopen. Er den på? Lyser noe?
2. Åpne lokket og trykk en tast. Får du innloggingsbilde, er den på. Logg inn og kjør `serverstatus`.
3. Er den helt død: start den. Alt i Docker starter av seg selv.
4. Gikk batteriet tomt? Sjekk at laderen sitter godt, og kjør `batteri` når den er oppe.

## Disken er full

```bash
df -h                       # hvor full er den
docker image prune -f       # slett gamle Docker-nedlastinger
sudo apt autoremove -y      # slett ubrukte pakker
sudo journalctl --vacuum-size=200M   # krymp systemloggene
du -sh /data/photos         # hvor mye tar bildene
```

## Glemt Linux-passordet

Fra laptopen: start på nytt, hold **Shift** (eller trykk **Esc**) under oppstart for GRUB-menyen → *Advanced options* → *recovery mode* → *root*. Kjør så `mount -o remount,rw /` og `passwd jonas`, og til slutt `reboot`.
