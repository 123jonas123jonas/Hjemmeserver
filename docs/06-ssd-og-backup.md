# SSD og backup

Tilbake til [README](../README.md)

**Mål:** Koble den eksterne 1 TB SSD-en fast til serveren og bruke den til to ting:

1. **Automatisk kopi av bildene hver natt.** Nå ligger bildene bare på laptop-disken. Dør den, er de borte.
2. **Ekstra lagringsplass** for egne filer, film osv. Synlig på PC-en som disk **Y:**.

> [!WARNING]
> **Flytte er ikke backup**
> Flytter du bildene over på SSD-en, har du fortsatt bare én kopi. Poenget er at de ligger begge steder.

## Plan

- [x] 1. Kopier alt fra SSD-en til PC-en
- [x] 2. Plugg SSD-en i serveren og finn navnet
- [x] 3. Formater som ext4
- [x] 4. Koble den til fast på `/mnt/ssd`
- [x] 5. Del den med Samba (Y:)
- [x] 6. Kopier tingene dine tilbake
- [x] 7. Sett opp nattlig backup

## Hvorfor ext4

ext4 er Linux sitt eget filsystem. Sammenlignet med exFAT/NTFS:

- tåler strømbrudd bedre (exFAT har ingen journal og blir lettere korrupt hvis strømmen går midt i en kopiering)
- rettigheter og eierskap virker som på resten av serveren
- backup med `rsync` blir mer pålitelig

Ulempen: Windows kan ikke lese den hvis du plugger den rett i PC-en. Den skal stå på serveren, og du når filene via Samba, så det gjør lite.

## 1. Kopier alt av SSD-en

På PC-en: plugg i SSD-en, kopier alt til en mappe. Sjekk at kopien er komplett: høyreklikk på begge mappene → Egenskaper → antall filer og størrelse skal stemme.

## 2. Finn navnet på SSD-en

Plugg SSD-en i serveren og kjør:

```bash
lsblk -o NAME,SIZE,FSTYPE,LABEL,MOUNTPOINT,TRAN,MODEL
```

SSD-en har `usb` under `TRAN` og rundt `931G` under `SIZE`. Navnet er f.eks. `sdb`. Laptop-disken (der bildene ligger nå) har `/` under `MOUNTPOINT` på en av partisjonene sine. **Den skal du ikke røre.**

> [!CAUTION]
> **Dobbeltsjekk disknavnet før steg 3**
> Steg 3 sletter alt på disken du velger. Velges feil disk, forsvinner bildene. Blokken under stopper av seg selv hvis disken ikke er USB eller er systemdisken.

## 3. Formater

Bytt `sdX` med navnet fra steg 2 (uten tall, f.eks. `sdb`). Lim inn alt:

```bash
DISK=sdX
sudo apt install -y parted
if [ "$(lsblk -dno TRAN /dev/$DISK)" = "usb" ] && ! lsblk -no MOUNTPOINT /dev/$DISK | grep -qx '/'; then
  echo "Formaterer /dev/$DISK: $(lsblk -dno SIZE,MODEL /dev/$DISK)"
  sudo umount /dev/${DISK}* 2>/dev/null
  sudo wipefs -a /dev/$DISK
  sudo parted -s /dev/$DISK mklabel gpt mkpart ssd ext4 0% 100%
  sleep 2
  sudo mkfs.ext4 -F -L ssd /dev/${DISK}1
  echo "FERDIG"
else
  echo "STOPP: /dev/$DISK er ikke en USB-disk, eller systemet kjører fra den. Ingenting er gjort."
fi
```

## 4. Koble den til fast

I samme vindu (så `DISK` fortsatt er satt):

```bash
UUID=$(sudo blkid -s UUID -o value /dev/${DISK}1)
sudo mkdir -p /mnt/ssd
echo "UUID=$UUID /mnt/ssd ext4 defaults,nofail,x-systemd.device-timeout=10s 0 2" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
sudo chown jonas:jonas /mnt/ssd
df -h /mnt/ssd
```

`nofail` gjør at serveren starter som normalt selv om SSD-en ikke er plugget i. Den kobles til `/mnt/ssd` automatisk ved hver oppstart. `serverstatus` viser den med ledig plass.

## 5. Del den med Samba

```bash
sudo tee -a /etc/samba/smb.conf > /dev/null << 'EOF'

[SSD]
   path = /mnt/ssd
   valid users = jonas
   read only = no
   browseable = yes
EOF
sudo systemctl restart smbd
```

På PC-en (CMD):

```cmd
net use Y: \\100.x.x.x\SSD /persistent:yes
```

Samme brukernavn og Samba-passord som Z:.

## 6. Kopier tingene dine tilbake

Dra mappa fra PC-en over til **Y:** i Utforsker.

## 7. Nattlig backup

Kopierer bildene og innstillingene for Home Assistant og Homepage til `/mnt/ssd/backup` hver natt kl. 03:00, men bare hvis SSD-en er koblet til. Ellers ville kopien havnet på laptop-disken og fylt den opp.

```bash
sudo apt install -y rsync
sudo timedatectl set-timezone Europe/Oslo
mkdir -p /mnt/ssd/backup
sudo tee /etc/cron.d/hjemmeserver-backup > /dev/null << 'EOF'
# Nattlig backup til SSD-en kl. 03:00 (bare hvis SSD-en er koblet til)
0 3 * * * root mountpoint -q /mnt/ssd && rsync -a /data/photos/ /mnt/ssd/backup/photos/ && rsync -a /home/jonas/homeassistant/ /mnt/ssd/backup/homeassistant/ && rsync -a /home/jonas/homepage/ /mnt/ssd/backup/homepage/ && date -Iminutes > /mnt/ssd/backup/sist-kjort.txt
EOF
```

Ta den første kopien med en gang, så du slipper å vente til i natt:

```bash
sudo rsync -a --info=progress2 /data/photos/ /mnt/ssd/backup/photos/
```

**Sjekk at den virker:** `serverstatus` viser "Backup:" med tidspunktet for siste kjøring, eller `cat /mnt/ssd/backup/sist-kjort.txt`.

Godt å vite:
- Sletter du et bilde på Z:, blir det liggende i backupen. Det er med vilje, så du ikke mister noe ved et uhell. Backupen vokser derfor litt over tid.
- Backupen ligger synlig på Y: under `backup`. Ikke slett noe der.
- Home Assistant kopieres mens den kjører. Det holder for innstillingene, men Home Assistant har også egen backup under Innstillinger → System → Sikkerhetskopier.

## Neste nivå

Serveren og SSD-en står på samme sted. Brann, tyveri eller et overspenningsstøt tar begge. Den tryggeste løsningen er en tredje kopi et annet sted: en disk du oppbevarer utenfor huset, eller en skytjeneste for de viktigste bildene.
