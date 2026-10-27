# Ordliste

Tilbake til [README](../README.md)

| Ord | Betyr |
|---|---|
| **Server** | En datamaskin som står på hele tiden og gjør tjenester tilgjengelige for andre enheter. Din er den gamle laptopen. |
| **Ubuntu Server** | Linux-versjonen serveren kjører. Uten grafisk skrivebord, bare tekst. |
| **Terminal / kommandolinje** | Der du skriver kommandoer. PowerShell på PC-en, og skallet (bash) på serveren. |
| **sudo** | "Gjør dette som administrator". Spør om Linux-passordet. |
| **IP-adresse** | Adressen til en enhet på nettet, f.eks. `10.0.0.23` eller `100.64.1.2`. |
| **Port** | Et nummer som sier hvilket program på serveren du vil snakke med. 22 = SSH, 8123 = Home Assistant. |
| **Ruter** | Boksen som lager hjemmenettet og deler ut IP-adresser. Du har to. |
| **DHCP** | Systemet i ruteren som deler ut IP-adresser automatisk. |
| **DHCP-reservasjon** | Ber ruteren alltid gi samme IP til samme enhet. |
| **NAT / dobbel-NAT** | Ruteren gjemmer alle hjemmeenhetene bak én adresse. To rutere etter hverandre = dobbel-NAT, som gjør ting vanskeligere. |
| **SSH** | Sikker fjernstyring av serveren fra en annen maskin. |
| **Samba / SMB** | Måten Windows deler mapper over nettet. Samba er Linux-programmet som snakker SMB. |
| **VPN** | Kryptert tunnel mellom enheter over internett. |
| **Tailscale** | En VPN som kobler dine enheter sammen med faste 100.x-adresser, uten å åpne porter. |
| **WireGuard** | Krypteringen Tailscale bruker under panseret. |
| **Taildrop** | Tailscale sin funksjon for å sende filer mellom enhetene dine. |
| **Docker** | Kjører programmer i lukkede bokser (containere). |
| **Image** | Ferdig pakke med et program, lastet ned av Docker. |
| **Container** | En kjørende kopi av en image. |
| **compose.yaml** | Oppskriften Docker følger: hvilken image, porter, mapper. |
| **`docker compose up -d`** | Start det som står i `compose.yaml`, i bakgrunnen (`-d`). |
| **YAML** | Filformat for innstillinger. Bruker innrykk (mellomrom) for struktur, og er nøye på dem. |
| **Home Assistant** | Smarthus-system som samler enheter, sensorer og automasjoner. |
| **ESPHome** | Lager programvare til ESP-kort (D1 Mini) så de snakker med Home Assistant. |
| **HACS** | Butikk for ekstra integrasjoner til Home Assistant, laget av andre brukere. |
| **Homepage** | Startside/dashbord med knapper til alle tjenestene. |
| **Brannmur / ufw** | Blokkerer innkommende trafikk unntatt portene du åpner. |
| **netplan** | Der Ubuntu lagrer nettverksinnstillingene. |
| **localhost / 127.0.0.1** | "Denne maskinen selv". |
| **mount / montere** | Koble en disk til en mappe, f.eks. SSD-en til `/mnt/ssd`. |
| **fstab** | Liste over disker som skal kobles til automatisk ved oppstart (`/etc/fstab`). |
| **ext4** | Linux sitt vanlige filsystem. |
| **exFAT / NTFS** | Filsystemer Windows bruker. Linux kan lese dem, men de passer dårligere på en server. |
| **rsync** | Kopierer filer smart: bare det som er nytt eller endret. |
| **cron** | Kjører kommandoer automatisk til faste tider. |
| **`~`** | Hjemmemappa di, `/home/jonas`. |
| **root** | Administratorbrukeren. Filer eid av root kan du ikke endre uten `sudo`. |
| **chown** | Bytter eier av filer, f.eks. fra root til jonas. |
