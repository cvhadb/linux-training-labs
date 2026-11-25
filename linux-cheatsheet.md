# Linux cheat sheet

## Labo-omgeving

- Repo: fork van HoGentTIN/linux-training-labs, lokaal in `~/linux-training-labs`
- Basis-VMs: map `~/linux-training-labs/vagrant`
- AlmaLinux-VM: naam `el`, AlmaLinux 10.1, IP `192.168.56.10`

| Interface | Adres | Rol |
|---|---|---|
| `enp0s3` | 10.0.2.15/24 | NAT (internet), gateway 10.0.2.2, DNS 10.0.2.3 |
| `enp0s8` | 192.168.56.10/24 | host-only (bereikbaar vanaf de laptop) |

Let op: de cursus gebruikt `eth0` en `eth1` in de voorbeelden. Bij mij is dat `enp0s3` en `enp0s8`.

## Vagrant (op de laptop, in de map met de Vagrantfile)

| Commando | Wat het doet |
|---|---|
| `vagrant status` | toestand van de VMs in deze map |
| `vagrant global-status` | alle Vagrant-VMs op de laptop |
| `vagrant up el` | VM aanmaken (eerste keer, met provisioning) of starten |
| `vagrant ssh el` | inloggen als gebruiker vagrant |
| `exit` | uit de VM, terug naar de laptop |
| `vagrant halt el` | VM netjes afsluiten |
| `vagrant suspend el` / `vagrant resume el` | pauzeren / hervatten |
| `vagrant reload el` | herstarten (nieuwe instellingen uit vagrant-hosts.yml) |
| `vagrant provision el` | provisioningscript opnieuw uitvoeren |
| `vagrant destroy el` | VM met schijf verwijderen |
| `vagrant box list` | lokaal gedownloade boxes |

Provisioning loopt enkel bij de eerste `vagrant up`, niet bij een gewone herstart.

Bestand van de laptop naar de VM kopiëren:

```
vagrant upload <bestand-op-laptop> <pad-in-vm> el
```

Let op: plak altijd één regel tegelijk en wacht op je prompt. Kijk naar de prompt om te weten waar je bent: `audrey@audrey-ThinkPad...` = laptop, `vagrant@el` = VM.

## Probleem: /vagrant mount niet na een kernelupdate

Foutmelding bij `vagrant up` of `vagrant reload`: `mount.vboxsf: mounting failed ... No such device`.

Oorzaak: de Guest Additions (kernelmodule `vboxsf`) waren gebouwd voor de oude kernel.

Oplossing (Guest Additions van de VirtualBox op de laptop installeren):

```
# op de laptop
vagrant upload /usr/share/virtualbox/VBoxGuestAdditions.iso /tmp/VBoxGuestAdditions.iso el
vagrant ssh el
# in de VM
sudo dnf install -y kernel-devel-$(uname -r) gcc make perl bzip2 elfutils-libelf-devel
sudo mkdir -p /mnt/iso && sudo mount -o loop,ro /tmp/VBoxGuestAdditions.iso /mnt/iso
sudo /mnt/iso/VBoxLinuxAdditions.run
sudo umount /mnt/iso && rm /tmp/VBoxGuestAdditions.iso
exit
# op de laptop
vagrant reload el
```

`sudo /sbin/rcvboxadd quicksetup all` herbouwt de bestaande versie, maar dat faalt als die te oud is voor de nieuwe kernel (log: `/var/log/vboxadd-setup.log`).

## Eerste controle in een nieuwe VM

```
cat /etc/os-release | head -3   # welke distributie en versie
ip -br a                        # interfaces en IP-adressen
getenforce                      # SELinux-mode (Enforcing / Permissive / Disabled)
uname -r                        # welke kernel draait
```


## Prompt in Debian-stijl (PS1)

EL toont `[vagrant@webt ~]$`, Debian `vagrant@webt:~$`. Omschakelen:

```
# enkel deze sessie, zonder kleur (kort, makkelijk te onthouden):
PS1='\u@\h:\w\$ '

# met kleur, blijvend in deze VM:
echo 'PS1='"'"'\[\e[01;32m\]\u@\h\[\e[00m\]:\[\e[01;34m\]\w\[\e[00m\]\$ '"'"'' >> ~/.bashrc && source ~/.bashrc

# alias debprompt (in ~/.bashrc van een VM die je zelf beheert):
echo 'alias debprompt='"'"'PS1="\[\e[01;32m\]\u@\h\[\e[00m\]:\[\e[01;34m\]\w\[\e[00m\]\\$ "'"'" >> ~/.bashrc && source ~/.bashrc
debprompt
```

| Deel | Betekenis |
|---|---|
| `\u@\h` | gebruiker@host |
| `\w` | volledige map (`~` voor home) |
| `\$` | `$` voor gewone gebruiker, `#` voor root |
| `\[\e[01;32m\]` ... `\[\e[00m\]` | kleur aan (groen) en uit; `\[ \]` = neemt geen plaats in |

- `~/.bashrc` geldt per gebruiker en per machine. In een nieuwe VM (of op het examen) bestaat de alias nog niet.
- Op het examen: gewoon `PS1='\u@\h:\w\$ '` typen (enkel die sessie). Dat is korter dan eerst een alias aanmaken.

## H4 Pakketbeheer op EL (dnf en rpm)

| Commando | Wat het doet |
|---|---|
| `sudo dnf upgrade` | alle updates installeren (vernieuwt de lijsten zelf; geen `apt update` nodig) |
| `sudo dnf install -y pakket` | installeren met afhankelijkheden, zonder vragen |
| `sudo dnf remove pakket` | verwijderen, ook overbodige afhankelijkheden |
| `dnf search woord` | pakketten zoeken |
| `dnf info pakket` | details (repo `@System` = geïnstalleerd) |
| `dnf provides '*bin/scp'` | welk pakket levert dit bestand |
| `dnf list --installed` / `--available` | geïnstalleerde / beschikbare pakketten |
| `dnf repolist` | actieve repositories |
| `rpm -q pakket` | is dit pakket geïnstalleerd? |
| `rpm -qa` | alle geïnstalleerde pakketten |
| `rpm -ql pakket` | bestanden van een pakket |

- Debian: `sudo apt update` + `sudo apt upgrade` (twee commando's). EL: enkel `sudo dnf upgrade`.
- Herstarten is meestal enkel nodig na een kernelupdate. Een nieuwe kernel staat onder "Installed": EL houdt de oude kernel ernaast.
- Pakketnaam: `naam-versie-release.architectuur`, bv. `vim-minimal-2:9.1.083-26.el10_3.x86_64` (`2:` = epoch, `el10` = Enterprise Linux 10).

### Labo H4 pakketbeheer: wat ik leerde

- `rpm -q` werkt met **pakketnamen**, niet met commando's. `rpm -q scp` zegt "not installed", maar het commando `scp` bestaat wel (pakket `openssh-clients`).
- Welk pakket levert een commando: `dnf provides '*bin/scp'`. `Repo : @System` = al geïnstalleerd.
- Staat een pakket niet in de standaardrepo's (bv. `zork`), kijk dan in EPEL:

```
sudo dnf install -y epel-release
dnf list --available zork        # repo-kolom toont: epel
sudo dnf install zork
```

- Afhankelijkheden: `git` zelf is maar 40 kB; het echte programma zit in `git-core`, dat dnf automatisch meeneemt (+ `git-core-doc`, `perl-Error`, `perl-Git`).
- Elke VM heeft zijn eigen pakketten: wat in de Debian-VM staat, staat niet in de AlmaLinux-VM.
- Webmin: niet in de repo's; via een `.repo`-bestand van webmin.com in `/etc/yum.repos.d/`.

## H4 Netwerkconfiguratie (labo)

Labo-omgeving: `~/linux-training-labs/network-config`, twee VMs met een intern netwerk zonder automatische configuratie.

```
---
- name: deb
  box: bento/debian-13
  intnet: true          # tweede adapter op een intern netwerk
  auto_config: false    # Vagrant configureert die adapter niet
- name: alma
  box: bento/almalinux-10
  intnet: true
  auto_config: false
```

### Netwerkdiagram

```
               Internet
                  |
        +---------+---------+
        |                   |
     [NAT 1]             [NAT 2]          (apart per VM)
        | eth0              | enp0s3
      (deb)              (alma)
        | eth1              | enp0s8
        +------[intnet]-----+             (intern netwerk, geen DHCP)
```

### IP-tabel

| Host | Iface | MAC | IPv4 | IPv6 | GW |
|---|---|---|---|---|---|
| deb | eth0 | 08:00:27:59:a8:77 | 10.0.2.15/24 | fe80::e845:83b0:bc1:ed0d/64 | 10.0.2.2 |
| | eth1 | 08:00:27:96:4d:1d | 192.168.100.1/24 | fd00:1337:cafe:1::1/64 | – |
| alma | enp0s3 | 08:00:27:c1:d0:d8 | 10.0.2.15/24 | fe80::a00:27ff:fec1:d0d8/64 | 10.0.2.2 |
| | enp0s8 | 08:00:27:dc:2c:26 | 192.168.100.2/24 | fd00:1337:cafe:1::2/64 | – |

### Informatie opvragen

| Wat | Commando |
|---|---|
| MAC-adressen | `ip -br link` |
| IP-adressen en maskers | `ip a` of `ip -br a` |
| Routeringstabel, default gateway | `ip r` |
| DNS-servers | `cat /etc/resolv.conf` (werkt altijd) / `resolvectl dns` (enkel met systemd-resolved) |
| ARP-cache | `ip n` |
| Publiek IP | `curl ifconfig.me` (IPv6 als je thuis IPv6 hebt) / `curl -4 ifconfig.me` |

### Wat ik zag

- Beide VMs hebben 10.0.2.15 op de NAT-interface: geen conflict, want elke VM heeft een eigen NAT-netwerk.
- Debian (bento-box): namen `eth0`/`eth1`, de voorspelbare naam staat als `altname enp0s3`. AlmaLinux: `enp0s3`/`enp0s8`.
- Tweede interface zonder IP: op Debian **DOWN** (niets configureert ze), op AlmaLinux **UP** (NetworkManager zet ze aan en probeert DHCP, maar een intern netwerk heeft geen DHCP-server).
- DOWN zonder `NO-CARRIER` = kabel zit erin, interface staat uit. `NO-CARRIER` = geen kabel.
- Een interface zonder IP-adres staat niet in de routeringstabel.
- De interface op het interne netwerk krijgt geen gateway: internetverkeer gaat via de NAT-interface.
- `resolv.conf` op Debian: geschreven door dhcpcd. Op AlmaLinux: door NetworkManager. `resolvectl dns` faalt op AlmaLinux ("The name is not activatable"): er draait geen systemd-resolved.
- In de ARP-cache hebben 10.0.2.2 (gateway) en 10.0.2.3 (DNS) hetzelfde MAC-adres: beide zijn de NAT-engine van VirtualBox.

### Debian: /etc/network/interfaces

Standaardinhoud (bento debian-13):

```
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

allow-hotplug eth0
iface eth0 inet dhcp
iface eth0 inet6 auto
pre-up sleep 2
```

| Regel | Betekenis |
|---|---|
| `source /etc/network/interfaces.d/*` | extra configuratiebestanden uit die map inlezen |
| `auto lo` | interface automatisch opbrengen bij het booten |
| `iface lo inet loopback` | lo is de loopback-interface (127.0.0.1) |
| `allow-hotplug eth0` | opbrengen zodra de kernel de interface detecteert |
| `iface eth0 inet dhcp` | IPv4 via DHCP |
| `iface eth0 inet6 auto` | IPv6 configureert zichzelf (SLAAC) |
| `pre-up sleep 2` | voor het opbrengen eerst 2 seconden wachten |
| `iface eth1 inet static` + `address 192.168.100.1/24` | vast IPv4-adres |
| `iface eth1 inet6 static` + `address fd00:1337:cafe:1::1/64` | vast IPv6-adres |

Statisch adres toevoegen (eerst backup):

```
sudo cp /etc/network/interfaces /etc/network/interfaces.bak && sudo tee -a /etc/network/interfaces <<'EOF'

auto eth1
iface eth1 inet static
  address 192.168.100.1/24
iface eth1 inet6 static
  address fd00:1337:cafe:1::1/64
EOF
```

Valkuil: `sudo echo tekst >> /etc/bestand` werkt niet ("Permission denied"). De omleiding `>>` voert je eigen shell uit, zonder sudo. Gebruik `echo tekst | sudo tee -a /etc/bestand` (of `tee -a` met een heredoc zoals hierboven).

Toepassen: `sudo ifup eth1` (na een wijziging aan een actieve interface: `sudo ifdown eth1 && sudo ifup eth1`).

### AlmaLinux: nmcli

```
nmcli connection show            # profielen (connections) en hun device
nmcli device                     # interfaces (devices) en hun toestand
sudo nmcli connection add con-name enp0s8 type ethernet ifname enp0s8 \
  ipv4.method manual ipv4.addresses 192.168.100.2/24 \
  ipv6.method manual ipv6.addresses fd00:1337:cafe:1::2/64
ip -br a show enp0s8             # controle
```

- Er was geen profiel voor `enp0s8`: een device zonder connection. `connection add` maakt er een en activeert het meteen (autoconnect).
- Een profiel is blijvend. `sudo ip a add ...` is weg na een reboot.

### Testen en resultaat

```
sudo ifup eth1                                         # Debian: eth1 opbrengen
ip -br a show eth1                                     # controle
ping -c3 192.168.100.2; ping -c3 fd00:1337:cafe:1::2   # IPv4 en IPv6 naar alma
ip n                                                   # ARP/NDP-cache
```

- Ping lukt over IPv4 en IPv6: `ttl=64`, geen router ertussen (zelfde LAN). firewalld op AlmaLinux blokkeert ping (ICMP) niet.
- ARP-cache na de ping: nieuwe regels via `eth1` met het MAC-adres van alma (`08:00:27:dc:2c:26`), voor 192.168.100.2 én voor de IPv6-adressen.
- IPv4 gebruikt ARP, IPv6 gebruikt NDP (Neighbor Discovery). `ip n` toont beide.
- Link-local `fe80::a00:27ff:fe96:4d1d` op Debian bevat het MAC-adres `08:00:27:96:4d:1d` met `ff:fe` in het midden (EUI-64). AlmaLinux (NetworkManager) kiest een willekeurig link-local adres (privacy).

## H4 Kea DHCP (labo)

Labo-omgeving: `~/linux-training-labs/kea`, VM `kea-el` (AlmaLinux 10). NAT op `enp0s3`, host-only `enp0s8` met 192.168.42.254/24. Het provisioningscript zet de VM op als router met NAT (ip_forward + firewalld zones external/internal + policy int2ext) en installeert Kea met een kant-en-klare `/etc/kea/kea-dhcp4.conf`.

### Probleem: host-only IP niet toegelaten

Foutmelding bij `vagrant up`: `The IP address configured for the host-only network is not within the allowed ranges ... Ranges: 192.168.56.0/21`.

Oorzaak: VirtualBox op Linux laat standaard enkel host-only netwerken in 192.168.56.0/21 toe (.56 tot .63). Oplossing op de laptop:

```
sudo mkdir -p /etc/vbox && echo '* 192.168.0.0/16 fe80::/10' | sudo tee /etc/vbox/networks.conf
```

### Kea controleren

```
systemctl status kea-dhcp4         # draait de dienst?
sudo ss -ulnp | grep :67           # luistert Kea op UDP 67, op het juiste IP?
journalctl -flu kea-dhcp4          # logs live volgen
kea-dhcp4 -t /etc/kea/kea-dhcp4.conf   # syntax controleren
```

- `ss` toont `UNCONN 0 0 192.168.42.254:67 ... kea-dhcp4`: UDP kent geen verbindingen, dus een luisterende UDP-socket staat als `UNCONN` (bij TCP zou het `LISTEN` zijn). Het IP toont dat Kea enkel op `enp0s8` luistert.
- Logregels bij de start: `added IPv4 subnets: 1` (subnet ingelezen), `loading leases from file /var/lib/kea/kea-leases4.csv` (uitgedeelde adressen, memfile), `DHCP4_STARTED` en status `Dispatching packets...` (wacht op aanvragen).
- De melding "Use of this system is acceptance of the OS vendor EULA" bij het inloggen is een standaardtekst (motd) van de bento-boxes, geen probleem.

### Client toevoegen (Debian op hetzelfde host-only netwerk)

```
- name: client
  box: bento/debian-13
  ip: 192.168.42.10      # enkel om op hetzelfde host-only netwerk te komen
  auto_config: false     # Vagrant zet dat IP niet: de client vraagt het aan Kea
```

Zonder `ip` zou Vagrant de DHCP-server van VirtualBox aanzetten, die met Kea concurreert.

- Tweede VM in dezelfde omgeving: SSH-poort 2200 in plaats van 2222 (2222 is al in gebruik). Vagrant kiest zelf.
- "Vagrant insecure key detected": elke box heeft dezelfde publieke sleutel; Vagrant vervangt die door een unieke per VM.

Op de client (Debian):

```
sudo tee -a /etc/network/interfaces <<'EOF'

auto eth1
iface eth1 inet dhcp
EOF
sudo ifup eth1
```

dhcpcd toont het verloop: `offered 192.168.42.100 from 192.168.42.254` → `probing address` (ARP-controle) → `leased 192.168.42.100 for 7200 seconds` → `adding default route via 192.168.42.254`.

### DHCP-verkeer bekijken (tweede terminal, op kea-el)

```
sudo tcpdump -n -i enp0s8 port 67 or port 68
```

| tcpdump | Richting | DORA |
|---|---|---|
| `0.0.0.0.bootpc > 255.255.255.255.bootps ... Request` | client → iedereen | Discover |
| `192.168.42.254.bootps > 192.168.42.100.bootpc ... Reply` | Kea → client | Offer |
| `0.0.0.0.bootpc > 255.255.255.255.bootps ... Request` | client → iedereen | Request |
| `192.168.42.254.bootps > 192.168.42.100.bootpc ... Reply` | Kea → client | Ack |

- `bootps` = poort 67 (server), `bootpc` = poort 68 (client). Met `-nn` zie je de nummers.
- Client heeft nog geen adres: van 0.0.0.0 naar broadcast 255.255.255.255.
- Zonder `-v` toont tcpdump Discover en Request allebei als "Request", Offer en Ack als "Reply".

### Twee default routes (valkuil)

```
default via 10.0.2.2       dev eth0  metric 1002   <- wint: laagste metric
default via 192.168.42.254 dev eth1  metric 1003
```

Vagrant geeft elke VM een NAT-adapter, dus de client gaat standaard niet via de router kea-el. Om de router te testen, enkel de default route via eth0 weghalen (niet de interface: vagrant ssh loopt via eth0):

```
sudo ip route del default via 10.0.2.2 dev eth0   # tijdelijk, terug na reboot
ip r
ip route get 1.1.1.1                              # welke weg neemt een pakket? via 192.168.42.254 dev eth1
curl -sI https://www.linux-training.be | head -1  # HTTP-statusregel: internet via kea-el werkt
```

Valkuil: pings naar 10.0.2.x vanaf de client zeggen niets over de router. De client heeft zelf 10.0.2.0/24 op eth0, dus die pakketten gaan nooit via kea-el. Ook de DNS 10.0.2.3 bereikt hij via zijn eigen NAT-adapter.

### Reservatie (vast adres per MAC) met langere leasetijd

Een reservatie staat **in het subnet**. Een eigen leasetijd per host kan niet rechtstreeks in de reservatie: dat gaat via een **client class**. Kies het vaste adres buiten de pool, zodat het nooit botst met een dynamisch adres.

```
"client-classes": [
  { "name": "long-lease", "valid-lifetime": 14400, "max-valid-lifetime": 36000 }
],
"subnet4": [ {
  ...
  "reservations": [
    { "hw-address": "08:00:27:4a:20:0a", "ip-address": "192.168.42.50", "client-classes": [ "long-lease" ] }
  ]
} ]
```

Werkwijze:

```
sudo cp /etc/kea/kea-dhcp4.conf /etc/kea/kea-dhcp4.conf.bak    # eerst backup
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf                       # syntax (WARN over multi-threading = normaal)
sudo systemctl restart kea-dhcp4 && systemctl is-active kea-dhcp4
sudo journalctl -flu kea-dhcp4                                  # logs live (sudo nodig!)
# op de client:
sudo ifdown eth1 && sudo ifup eth1                              # lease vrijgeven en opnieuw aanvragen
```

Resultaat: client kreeg 192.168.42.50 voor 14400 s (was .100 voor 7200 s). Kea-log: `DHCP4_LEASE_ALLOC [hwtype=1 08:00:27:4a:20:0a] ... lease 192.168.42.50 has been allocated for 14400 seconds`.

- Het MAC-adres van de client vind je in tcpdump (`Request from 08:00:27:4a:20:0a`), in de Kea-log of op de client met `ip -br link`.
- `journalctl` zonder sudo: "No journal files were opened due to insufficient permissions". Systeemlogs vragen rootrechten (of lid van `wheel`/`adm`/`systemd-journal`, na opnieuw inloggen).
- Twee DHCP-servers op hetzelfde netwerk (laatste vraag van het labo): de client neemt het eerste aanbod dat binnenkomt. Welke server dat is, is onvoorspelbaar. Daarom maar één DHCP-server per netwerk (een ongewenste tweede heet een rogue DHCP-server).

## H5 systemd (labo, op el)

| Commando | Wat het doet |
|---|---|
| `systemctl get-default` | standaard target (hier `multi-user.target`: CLI met netwerk) |
| `systemctl list-units --type=target --state=active` | actieve targets |
| `systemctl list-dependencies multi-user.target` | boom van wat een target nodig heeft |
| `systemctl -t service` / `-t socket` | units van een type |
| `systemctl -t help` | alle unit-types |
| `systemctl status crond --no-pager` | status zonder pager |
| `systemctl is-enabled X` / `is-active X` | kort: enabled/disabled, active/inactive/failed |
| `sudo systemctl enable --now X` | bij het booten + nu starten |
| `sudo systemctl disable --now X` | niet meer bij het booten + nu stoppen |

- Pager: `lines 1-23` onderaan = je zit in `less`. `q` om eruit te gaan, of `--no-pager`.
- Veel targets tegelijk actief: een target trekt andere targets mee als afhankelijkheden (`basic`, `network`, `local-fs` ...).
- **Socket unit** (`man systemd.socket`): systemd luistert zelf op een poort en start de service pas bij een verbinding (socket activation). Bv. `cockpit.socket` luistert op 9090, `cockpit.service` start pas als iemand surft.
- Unit-types: service, socket, target, timer, mount, automount, swap, path, device, slice, scope.
- Cron heet `cron` op Debian en `crond` op EL.
- `disable` zonder `--now` stopt de dienst niet, het haalt hem enkel uit de opstart.
- enable/disable = een symlink maken/verwijderen in `/etc/systemd/system/multi-user.target.wants/` naar het unit-bestand in `/usr/lib/systemd/system/`.
- `Loaded: ... enabled; preset: enabled`: `preset` = toestand meteen na installatie.

### Apache en nginx tegelijk installeren

| | Debian | EL |
|---|---|---|
| Na installatie | meteen gestart en enabled | disabled en inactive (`preset: disabled`) |

```
sudo dnf install -y httpd nginx
sudo systemctl enable --now httpd; sudo systemctl start nginx    # nginx faalt
systemctl status nginx.service
sudo journalctl -u nginx --no-pager | tail -5                     # sudo nodig!
sudo ss -tlnp | grep :80
sudo systemctl disable --now httpd && sudo systemctl enable --now nginx   # wisselen
```

- Status nginx: `ExecStartPre=/usr/sbin/nginx -t ... SUCCESS` (syntax OK) maar `ExecStart ... FAILURE`: het probleem zit niet in de configuratie.
- Log: `bind() to [::]:80 failed (98: Address already in use)`: een ander proces (httpd) heeft poort 80 al. Twee webservers kunnen niet op dezelfde poort draaien.
- `Address already in use` = ander proces op die poort. `Permission denied` bij bind = meestal SELinux.
- `ss` bij httpd: één socket `*:80` (IPv4 en IPv6 samen). Bij nginx: twee sockets, `0.0.0.0:80` (IPv4) en `[::]:80` (IPv6).
- Meerdere processen met dezelfde socket: één hoofdproces en een aantal workers die de requests afhandelen.

## H5 Apache op EL (labo, op el)

```
sudo systemctl disable --now nginx && sudo systemctl enable --now httpd
curl -si http://localhost/ | head -1                           # in de VM
echo '<h1>Hallo van el</h1>' | sudo tee /var/www/html/index.html
curl -si --max-time 5 http://192.168.56.10/ | head -1          # vanaf de laptop
sudo firewall-cmd --list-all
sudo firewall-cmd --add-service=http --permanent && sudo firewall-cmd --reload
```

- Vers geïnstalleerde httpd op EL: `403 Forbidden`, want er is geen standaard `index.html` in `/var/www/html` (Apache toont een testpagina met status 403). Debian: "It works" met 200.
- curl in een pipe toont een voortgangsbalk: `-s` zet die af. `curl: Failed writing body` bij `| head -1` is onschuldig (head sluit de pipe).
- Vanaf de laptop: geen antwoord binnen 5 s = firewall dropt de pakketten. Zou Apache niet draaien: meteen "Connection refused".
- Zone `public` laat standaard enkel `cockpit dhcpv6-client ssh` toe. `http` moet je zelf toevoegen, met `--permanent` en `--reload`.
- **Waarom werkt localhost wel?** Verkeer naar `localhost` gaat over de loopback-interface `lo`. Die staat niet in de zone `public` (zie `interfaces: enp0s3 enp0s8`) en firewalld laat loopbackverkeer altijd door. De pakketten komen nooit langs enp0s8.
- Valkuil bij troubleshooting: "curl localhost werkt" bewijst niet dat de dienst bereikbaar is vanaf het netwerk. Test altijd ook vanaf een andere machine.

### Virtual hosts op andere poorten (8472, 31337, 1201)

Op EL moet een nieuwe poort langs **drie** plaatsen:

| Laag | Wat | Commando |
|---|---|---|
| Apache | `Listen` + `<VirtualHost *:poort>` in `/etc/httpd/conf.d/*.conf` | `sudo apachectl configtest` |
| SELinux | poort moet type `http_port_t` hebben | `sudo semanage port -a -t http_port_t -p tcp poort` |
| Firewall | poort toelaten | `sudo firewall-cmd --add-port=poort/tcp --permanent` + `--reload` |

```
for p in 8472 31337 1201; do sudo mkdir -p /var/www/port$p && echo "<h1>Poort $p</h1>" | sudo tee /var/www/port$p/index.html; done

sudo tee /etc/httpd/conf.d/ports.conf <<'EOF'
Listen 8472
Listen 31337
Listen 1201

<VirtualHost *:8472>
    DocumentRoot /var/www/port8472
</VirtualHost>
<VirtualHost *:31337>
    DocumentRoot /var/www/port31337
</VirtualHost>
<VirtualHost *:1201>
    DocumentRoot /var/www/port1201
</VirtualHost>
EOF

sudo apachectl configtest
sudo systemctl restart httpd            # faalt eerst: SELinux
sudo journalctl -u httpd --no-pager | tail -8
sudo semanage port -l | grep -w http_port_t
for p in 8472 31337 1201; do sudo semanage port -a -t http_port_t -p tcp $p || sudo semanage port -m -t http_port_t -p tcp $p; done
sudo systemctl restart httpd && sudo ss -tlnp | grep httpd
for p in 8472 31337 1201; do sudo firewall-cmd --add-port=$p/tcp --permanent; done && sudo firewall-cmd --reload
# vanaf de laptop:
for p in 8472 31337 1201; do curl -s --max-time 5 http://192.168.56.10:$p/; done
```

- `Syntax OK` maar de herstart faalt: in de log `(13)Permission denied: AH00072: make_sock: could not bind to address 0.0.0.0:8472`. Permission denied bij bind = SELinux.
- Standaard in `http_port_t`: 80, 81, 443, 488, 8008, 8009, 8443, 9000.
- `semanage port -a` faalt met "already defined" als de poort al een ander type heeft: dan `-m` (modify).
- `semanage` is traag (10 tot 20 s per poort).
- `AH00558: Could not reliably determine the server's fully qualified domain name` bij configtest is enkel een waarschuwing (op te lossen met `ServerName`).

### Named virtual hosts (startrek.local, starwars.local, stargate.local)

Alle drie op poort 80: Apache kiest de site op basis van de **naam** in de `Host`-header. Geen SELinux- of firewallwerk, want poort 80 is al toegelaten.

```
for s in startrek starwars stargate; do sudo mkdir -p /var/www/$s && echo "<h1>Welkom op $s</h1>" | sudo tee /var/www/$s/index.html; done
sudo tee /etc/httpd/conf.d/names.conf <<'EOF'
<VirtualHost *:80>
    ServerName startrek.local
    DocumentRoot /var/www/startrek
</VirtualHost>
<VirtualHost *:80>
    ServerName starwars.local
    DocumentRoot /var/www/starwars
</VirtualHost>
<VirtualHost *:80>
    ServerName stargate.local
    DocumentRoot /var/www/stargate
</VirtualHost>
EOF
sudo apachectl configtest && sudo systemctl restart httpd
# testen vanaf de laptop zonder DNS: de naam zelf in de Host-header zetten
for s in startrek starwars stargate; do curl -s -H "Host: $s.local" http://192.168.56.10/; done
curl -s http://192.168.56.10/     # zonder naam
```

- **Valkuil: de eerste VirtualHost voor `*:80` wordt de standaardsite.** Een request zonder (gekende) naam, bv. enkel op IP, krijgt startrek, niet meer de pagina uit `/var/www/html`. Bestanden in `conf.d` worden alfabetisch geladen.
- Wil je een eigen standaardsite: een bestand dat alfabetisch eerst komt, bv. `/etc/httpd/conf.d/00-default.conf` met `<VirtualHost *:80>` en `DocumentRoot /var/www/html`.
- In een browser: de namen in `/etc/hosts` van de laptop zetten (`192.168.56.10 startrek.local starwars.local stargate.local`).

### Tweede IP-adres op enp0s8 (voor een vhost op een ander IP)

```
nmcli -g NAME,DEVICE connection show                       # profielnamen (-g = enkel deze velden)
nmcli -g ipv4.method,ipv4.addresses connection show enp0s8 # wat staat er in het profiel?
sudo nmcli connection modify enp0s8 +ipv4.addresses 192.168.56.11/24   # + = toevoegen, zonder + = vervangen
sudo nmcli connection up enp0s8                            # toepassen
ip -br a show enp0s8
```

**Probleem: `nmcli connection modify` geeft permission denied, ook met sudo.**

```
sudo ls -Z /etc/NetworkManager/system-connections/
#   system_u:object_r:NetworkManager_etc_rw_t:s0  enp0s3.nmconnection
#   unconfined_u:object_r:user_tmp_t:s0           enp0s8.nmconnection   <- fout type
sudo restorecon -Rv /etc/NetworkManager/system-connections/
```

- Oorzaak: Vagrant maakte het profiel in `/tmp` en verplaatste het met `mv`. `mv` behoudt het oude SELinux-label (`user_tmp_t`), net zoals `index.html` met `user_home_t` in de cursus. NetworkManager mag niet schrijven naar `user_tmp_t`.
- SELinux (MAC) gaat boven gewone rechten en boven sudo (DAC).
- `restorecon` zet het type terug volgens de policy (`NetworkManager_etc_rw_t`). `-v` toont wat er verandert.
- Dit staat ook als commentaar in het provisioningscript van het Kea-labo: het is een fout in de Vagrant-box.

### Virtual host op een ander IP-adres

```
sudo mkdir -p /var/www/ip11 && echo '<h1>Site op 192.168.56.11</h1>' | sudo tee /var/www/ip11/index.html
sudo tee /etc/httpd/conf.d/ip11.conf <<'EOF'
<VirtualHost 192.168.56.11:80>
    DocumentRoot /var/www/ip11
</VirtualHost>
EOF
echo 'ServerName el.local' | sudo tee /etc/httpd/conf.d/servername.conf   # weg met AH00558
sudo apachectl configtest && sudo systemctl restart httpd
# vanaf de laptop:
curl -s http://192.168.56.11/; curl -s http://192.168.56.10/
```

- Resultaat: .11 toont de IP-site, .10 toont startrek (de standaardsite van `*:80`).
- Een VirtualHost met een **specifiek IP** (`192.168.56.11:80`) gaat voor op `*:80` voor requests naar dat IP.
- Geen extra firewallregel nodig: het nieuwe IP zit op dezelfde interface `enp0s8`, dus in dezelfde zone waar `http` al open staat.
- `ServerName` buiten een VirtualHost geldt globaal en haalt de waarschuwing AH00558 weg.

### Site beveiligen met gebruiker en wachtwoord (Basic Auth)

Labo-login: gebruiker `luke`, wachtwoord `warm` (enkel testwachtwoord in de labo-VM).

```
sudo htpasswd -c /etc/httpd/.htpasswd luke          # -c = bestand aanmaken (enkel de eerste keer!), vraagt wachtwoord
sudo htpasswd -b /etc/httpd/.htpasswd luke warm     # -b = wachtwoord op de commandoregel (komt in je history)
sudo cat /etc/httpd/.htpasswd                       # luke:$apr1$... = hash, geen wachtwoord

sudo tee /etc/httpd/conf.d/starwars-auth.conf <<'EOF'
<Directory /var/www/starwars>
    AuthType Basic
    AuthName "Alleen voor Jedi"
    AuthUserFile /etc/httpd/.htpasswd
    Require valid-user
</Directory>
EOF
sudo apachectl configtest && sudo systemctl restart httpd

# vanaf de laptop:
curl -si -H "Host: starwars.local" http://192.168.56.10/ | head -1        # HTTP/1.1 401 Unauthorized
curl -s -u luke:warm -H "Host: starwars.local" http://192.168.56.10/      # de pagina
```

- Valkuil: `htpasswd -c` op een bestaand bestand overschrijft het en wist alle andere gebruikers. Extra gebruiker: zonder `-c`.
- `$apr1$` = Apache-MD5-hash met salt.
- **401 Unauthorized** = je moet inloggen (geen of fout wachtwoord). **403 Forbidden** = je mag niet, ook niet met login (bv. geen index, verkeerde SELinux-context, geen leesrechten).
- Basic Auth stuurt gebruiker en wachtwoord enkel base64-gecodeerd mee, niet versleuteld: in productie altijd samen met HTTPS.
- `curl -u gebruiker:wachtwoord` = inloggen met Basic Auth.

## H6 Firewall (labo, op el)

Doel: NAT-interface `enp0s3` (vagrant ssh) in zone `internal` met enkel ssh, cockpit, dhcpv6-client. Host-only `enp0s8` in zone `public` met enkel webverkeer.

```
sudo firewall-cmd --list-all --zone=internal        # standaard: cockpit dhcpv6-client mdns samba-client ssh
sudo firewall-cmd --permanent --zone=internal --remove-service=mdns
sudo firewall-cmd --permanent --zone=internal --remove-service=samba-client
sudo firewall-cmd --permanent --zone=internal --change-interface=enp0s3
sudo firewall-cmd --permanent --zone=public --add-service=https
for s in ssh dhcpv6-client cockpit; do sudo firewall-cmd --permanent --zone=public --remove-service=$s; done
sudo firewall-cmd --reload && sudo firewall-cmd --get-active-zones
# vanaf de laptop:
curl -s --max-time 5 http://192.168.56.10/            # werkt
ssh -o ConnectTimeout=5 vagrant@192.168.56.10          # No route to host
```

- Volgorde: alles `--permanent`, pas op het einde `--reload`. Zorg eerst dat de zone van je SSH-interface `ssh` toelaat, anders sluit je jezelf buiten.
- "The interface is under control of NetworkManager, setting zone to 'internal'": firewalld bewaart de zone in het NetworkManager-profiel (`connection.zone`). Daarvoor moet het profielbestand de juiste SELinux-context hebben.
- `vagrant ssh` blijft werken: die loopt via `enp0s3` (zone `internal`, ssh toegelaten).
- **"No route to host" is hier geen routeprobleem.** De zone `public` weigert verkeer actief met een ICMP-bericht (host prohibited), en ssh toont dat als "No route to host". Een ping naar hetzelfde IP werkt wel.
- Firewall dropt (geen antwoord) → time-out. Firewall weigert (reject) → "No route to host" of "Connection refused". Niets luistert → "Connection refused".

## H9 Troubleshooting (labo)

Labo-omgeving: `~/linux-training-labs/troubleshooting`, twee AlmaLinux 9 VMs. `dbt` (MariaDB) is correct, `webt` (Apache + PHP) is opzettelijk kapot. Doel: `test.php` op webt toont data uit de databank.

Eerst: **ken je omgeving** (verwachte waarden in een tabel).

| Host | Interface | Verwacht IP | Dienst | Poort |
|---|---|---|---|---|
| dbt | eth1/enp0s8 | 192.168.76.73/24 | MariaDB | 3306 |
| webt | eth1/enp0s8 | 192.168.76.72/24 | Apache | 80 (en 443) |

- `vagrant ssh` werkt enkel in de map met de Vagrantfile. Anders: "A Vagrant environment or target machine is required".
- Prompt op EL 9: `[vagrant@webt ~]$`. Op Debian (en de bento-box van EL 10): `vagrant@el:~$`. `$` = gewone gebruiker, `#` = root.

### Linklaag

```
ip -br link       # enp0s8 DOWN <NO-CARRIER,...> = geen signaal
```

Gevonden op webt:
- `enp0s8` **NO-CARRIER**: in VirtualBox stond **Virtual Cable Connected** uit.
- Adapter 2 van webt zat op **vboxnet3**, die van dbt op **vboxnet2**: twee aparte netwerken.

Oplossen in VirtualBox: webt → Settings → Network → Adapter 2: Name = hetzelfde netwerk als dbt, en **Virtual Cable Connected** aanvinken. Kan terwijl de VM draait. (In VirtualBox 7 staat de optie soms onder "Advanced", te zien in Expert-modus.)

### Internetlaag

```
ip -br a; ip r; cat /etc/resolv.conf
nmcli -g NAME,DEVICE connection show                 # profielnaam: 'System enp0s8' (met spatie!)
sudo nmcli connection modify 'System enp0s8' ipv4.addresses 192.168.76.72/24
sudo nmcli connection down 'System enp0s8' && sudo nmcli connection up 'System enp0s8'
ping -c1 192.168.76.73; ping -c1 10.0.2.2; ping -c1 10.0.2.3; dig @10.0.2.3 www.linux-training.be +short
```

- Fout: enp0s8 had **192.168.56.172/25**: verkeerd subnet én verkeerd masker. Hosts in hetzelfde LAN: zelfde netwerkdeel en zelfde masker.
- Op EL 9 heet het profiel `System enp0s8` en is het een ifcfg-bestand in `/etc/sysconfig/network-scripts/` (daarom niet in `system-connections`). nmcli werkt op beide.
- Gateway (10.0.2.2) en DNS (10.0.2.3) waren in orde.

### Transportlaag

```
systemctl status httpd --no-pager
sudo systemctl enable --now httpd && sudo ss -tlnp | grep httpd
grep -n 8080 /etc/httpd/conf/httpd.conf
sudo sed -i.bak '47s/8080/80/' /etc/httpd/conf/httpd.conf     # backup + vervangen op regel 47
sudo apachectl configtest && sudo systemctl restart httpd && sudo ss -tlnp | grep httpd
sudo firewall-cmd --list-all
sudo firewall-cmd --add-service=http --permanent && sudo firewall-cmd --add-service=https --permanent && sudo firewall-cmd --reload
# vanaf de laptop:
curl -si --max-time 5 http://192.168.76.72/ | head -1          # 403 = Apache antwoordt (geen index), transportlaag OK
```

- httpd stond `disabled` en `inactive (dead)`.
- Apache luisterde op 443 en **8080** in plaats van 80 (`Listen 8080` op regel 47). 8080 mocht van SELinux, want het is een `http_cache_port_t`.
- Firewall liet enkel cockpit, dhcpv6-client en ssh toe.
- `Drop-In: .../httpd.service.d/php-fpm.conf`: Apache werkt samen met php-fpm voor PHP.
- `sed -i.bak` = bestand ter plaatse aanpassen en het origineel als `.bak` bewaren.

### Applicatielaag (+ SELinux)

```
curl -s --max-time 5 http://192.168.76.72/test.php       # vanaf de laptop
sudo tail -5 /var/log/httpd/error_log
ls -Z /var/www/html/
sudo restorecon -v /var/www/html/test.php
getsebool httpd_can_network_connect_db
sudo ausearch -m AVC -ts recent | grep -o 'denied.*tcontext=[^ ]*' | tail -2
sudo setsebool -P httpd_can_network_connect_db on
curl -s --max-time 5 http://192.168.76.72/test.php | sed -n '/<body>/,$p'
```

- "Access denied." op test.php. Log: `AH01071: ... Unable to open primary script: /var/www/html/test.php (Permission denied)`. `ls -Z`: type **user_home_t** (met mv uit een home-map). Oplossing: `restorecon`.
- Log `AH01276: Cannot serve directory ... No matching DirectoryIndex` = de 403 op `/` (geen index).
- Daarna: "Could not connect to database server: Permission denied". AVC: `{ name_connect } ... comm="php-fpm" dest=3306 ... tcontext=...mysqld_port_t`. Boolean **httpd_can_network_connect_db** stond `off`. Oplossing: `setsebool -P ... on` (`-P` = blijvend).
- Resultaat: de tabel met Tuxedo T. Penguin en Bobby Tables.

### Overzicht: 8 fouten, per laag

| Laag | Fout | Oplossing |
|---|---|---|
| Link | Virtual Cable Connected uit (NO-CARRIER) | aanvinken in VirtualBox |
| Link | adapter 2 op ander host-only netwerk (vboxnet3 i.p.v. vboxnet2) | zelfde netwerk als dbt |
| Internet | IP 192.168.56.172/25 | `nmcli connection modify ... ipv4.addresses 192.168.76.72/24` |
| Transport | httpd disabled en gestopt | `systemctl enable --now httpd` |
| Transport | Listen 8080 i.p.v. 80 | httpd.conf aanpassen, configtest, restart |
| Transport | firewall laat http/https niet toe | `firewall-cmd --add-service=http/https --permanent` + `--reload` |
| Applicatie | test.php met type user_home_t | `restorecon -v` |
| Applicatie | boolean httpd_can_network_connect_db off | `setsebool -P ... on` |

Puntenverdeling troubleshootingtest: 5/15 na de internetlaag, 8/15 na de transportlaag, 12/15 na applicatie + SELinux, +3 voor het verslag.

## H10 Opslag (labo, op raidsrv)

Labo-omgeving: `~/linux-training-labs/raid`, VM `raidsrv` (AlmaLinux 9) met drie lege schijven van 5 GB: `sdb`, `sdc`, `sdd`. H10 op sdb en sdc, H12 (RAID) daarna.

### Schijven bekijken

```
lsblk
sudo dmesg | grep '\[sd[b-d]\]' | head -6
ls -l /dev/sd*
```

- `lsblk` MAJ:MIN: **8** = major van de sd-driver. Minor stijgt per 16 per schijf (sda 0, sdb 16, sdc 32): ruimte voor 15 partities per schijf (sda1 = 8:1).
- dmesg: `10485760 512-byte logical blocks: (5.37 GB/5.00 GiB)`. 10485760 × 512 = 5 GiB (1024³); 5.37 GB is hetzelfde in 1000³.
- `sd 0:0:1:0` = SCSI-adres: controller, kanaal, target, LUN.
- `ls -l /dev/sd*`: eerste letter **b** = block device; `8, 16` = major, minor in plaats van een bestandsgrootte.

### Partities met fdisk

```
sudo fdisk /dev/sdb      # n, p, 1, Enter, +200M, p, w
sudo fdisk /dev/sdc      # n, p, 1, Enter, +400M
                         # n, e, 2, Enter, Enter     (extended over de rest)
                         # n, Enter, +300M           (logisch: sdc5)
                         # n, Enter, +300M           (logisch: sdc6)
                         # p, w
```

| Partitie | Grootte | Id | Type |
|---|---|---|---|
| sdb1 | 200M | 83 | primair, Linux |
| sdc1 | 400M | 83 | primair, Linux |
| sdc2 | 4.6G | 5 | extended (container) |
| sdc5 | 300M | 83 | logisch |
| sdc6 | 300M | 83 | logisch |

- Lege schijf: fdisk maakt een **DOS-disklabel** (= MBR). MBR: max. 4 primaire partities (`4 free`).
- Eerste sector 2048 = start op 1 MiB (alignment).
- Wijzigingen staan enkel in het geheugen tot `w`. `q` = stoppen zonder schrijven.
- Is alle ruimte voor primaire partities op, dan kiest fdisk zelf **logisch** en vraagt geen `l` meer. Valkuil: je volgende toets komt dan bij "First sector" terecht ("Value out of range"): gewoon Enter.
- Logische partities beginnen altijd bij **5**, ook als 3 en 4 vrij zijn.
- Een logische partitie begint net na het begin van de extended (823296 i.p.v. 821248): daar staat de beschrijving (EBR).
- Na `w`: `Calling ioctl() to re-read partition table` = fdisk laat de kernel de tabel opnieuw lezen (anders: `partprobe`).

### fdisk -l versus df -h

```
sudo fdisk -l /dev/sdb /dev/sdc | grep '^/dev'
df -h | grep sd
```

- `fdisk -l` = wat er **op de schijf** staat (partitietabel). Toont de nieuwe partities.
- `df -h` = gemounte **bestandssystemen** met vrije ruimte. Toont nieuwe partities pas na mkfs + mount. Swap staat er ook niet in.

### Backup van de partitietabel

```
sudo dd if=/dev/sdb of=sdb.mbr bs=512 count=1     # MBR = eerste 512 bytes
sudo sfdisk -d /dev/sdc > sdc.sfdisk              # met logische partities
cat sdc.sfdisk
# terugzetten: sudo dd if=sdb.mbr of=/dev/sdb  /  sudo sfdisk /dev/sdc < sdc.sfdisk, daarna sudo partprobe
```

- Het MBR bevat enkel primaire en extended partities. De logische staan in de EBRs binnen de extended partitie, dus voor sdc is `dd` van 512 bytes niet genoeg: gebruik `sfdisk -d`.

### Bestandssystemen maken en controleren

```
sudo mkfs -t ext2 /dev/sdb1 && sudo mkfs -t ext3 /dev/sdc5 && sudo mkfs -t ext4 /dev/sdc1
sudo tune2fs -m 0 /dev/sdc5          # gereserveerde ruimte voor root: 5% → 0%
lsblk -f /dev/sdb /dev/sdc           # FSTYPE en UUID per partitie
for p in sdb1 sdc5 sdc1; do sudo fsck -f /dev/$p; done   # enkel op niet-gemounte bestandssystemen
```

- `Creating journal` bij ext3/ext4: dat is het verschil met ext2 (geen journal, trage controle na een crash).
- ext reserveert standaard **5%** voor root, zodat het systeem blijft werken als gebruikers de schijf vullen. Op een datapartitie mag dat 0.
- mkfs geeft elk bestandssysteem meteen een **UUID**.
- `fsck` is een front-end: voor ext2/3/4 roept het `e2fsck` op. `-f` = controleren, ook als het bestandssysteem "clean" is.
- `11/51200 files` op een leeg bestandssysteem: 11 gereserveerde inodes (rootmap, `lost+found` ...).
- `sdc2` (extended) en `sdc6` hebben geen bestandssysteem.

### Mounten

```
sudo mkdir -p /mnt/project22 /mnt/salesnumbers
sudo mount /dev/sdb1 /mnt/project22                    # type wordt automatisch herkend
sudo mount /dev/sdc1 /mnt/salesnumbers
sudo cp -r /etc /mnt/salesnumbers/
df -h | grep mnt
sudo umount /mnt/salesnumbers                          # umount, zonder n!
sudo mkdir -p /srv/nfs/salesnumbers
sudo mount -o ro /dev/sdc1 /srv/nfs/salesnumbers
ls /mnt/salesnumbers /srv/nfs/salesnumbers
findmnt --real | grep -E 'sdb|sdc'                     # mount zonder argumenten is deprecated om te tonen
grep sdc1 /proc/mounts
sudo touch /srv/nfs/salesnumbers/test                  # Read-only file system
```

- df toont 186M voor een partitie van 200M: overhead van het bestandssysteem (inodetabellen, superblokkopieën).
- **De bestanden staan op het bestandssysteem van de partitie, niet in de map.** Na umount is `/mnt/salesnumbers` leeg; op het nieuwe mountpunt staan `etc` en `lost+found`.
- `lost+found`: aangemaakt door mkfs; fsck zet er losse stukken van bestanden in na een herstelling.
- `ro` geldt voor iedereen, ook root (ook met sudo: "Read-only file system").
- `seclabel` = het bestandssysteem ondersteunt SELinux-labels.
- Mount op een map met bestanden: die bestanden zijn **verborgen** (niet weg), na umount weer zichtbaar.
- Twee bestandssystemen op hetzelfde mountpunt: het laatste ligt **bovenop**; umount haalt het bovenste weg.

### Permanent mounten met UUID (/etc/fstab)

```
sudo cp /etc/fstab /etc/fstab.bak                      # eerst backup: een fout in fstab kan het opstarten blokkeren
echo "UUID=$(sudo blkid -s UUID -o value /dev/sdb1) /mnt/project22 ext2 defaults 0 2" | sudo tee -a /etc/fstab
echo "UUID=$(sudo blkid -s UUID -o value /dev/sdc1) /srv/nfs/salesnumbers ext4 ro 0 2" | sudo tee -a /etc/fstab
sudo umount /mnt/project22 /srv/nfs/salesnumbers && sudo mount -a && findmnt --real | grep -E 'sdb|sdc'
sudo systemctl daemon-reload
```

- `blkid -s UUID -o value` = enkel de UUID, zonder iets anders.
- fstab-kolommen: bron (`UUID=...`) · mountpunt · type · opties (`defaults`, `ro` ...) · dump (0) · fsck-volgorde (1 = root, 2 = de rest, 0 = niet).
- UUID i.p.v. `/dev/sdc1`: een apparaatnaam kan veranderen als er een schijf verdwijnt, de UUID hoort bij het bestandssysteem zelf.
- `mount -a` zonder fout = het opstarten zal ook lukken.
- Hint "your fstab has been modified, but systemd still uses the old version": systemd maakt van elke fstab-regel een mount unit (bv. `mnt-project22.mount`). `sudo systemctl daemon-reload` laat systemd fstab opnieuw inlezen.

## H12 RAID (labo, op raidlab)

Nieuwe VM `raidlab` in `~/linux-training-labs/raid` met vier schijven van 8 GB (dynamisch gealloceerd: nemen enkel ruimte in voor wat er effectief op staat).

```
# vagrant-hosts.yml (in de raid-map), daarna: cp provisioning/raidsrv.sh provisioning/raidlab.sh
- name: raidlab
  box: bento/almalinux-9
  ip: 192.168.56.6
  memory: 1024
  disks:
    - name: 'lab01'
      size: '8GB'
    - name: 'lab02'
      size: '8GB'
    - name: 'lab03'
      size: '8GB'
    - name: 'lab04'
      size: '8GB'
```

### RAID 5 met spare maken

```
for d in sdb sdc sdd sde; do echo 'type=fd' | sudo sfdisk /dev/$d; done    # partitie over hele schijf, type fd
sudo fdisk -l | grep raid
sudo mdadm --create /dev/md0 --level=5 --raid-devices=3 --spare-devices=1 /dev/sdb1 /dev/sdc1 /dev/sdd1 /dev/sde1
cat /proc/mdstat
sudo mkfs -t ext4 /dev/md0 && sudo mkdir -p /srv/raid && sudo mount /dev/md0 /srv/raid
echo "UUID=$(sudo blkid -s UUID -o value /dev/md0) /srv/raid ext4 defaults 0 2" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload && df -h /srv/raid
```

- `sfdisk` niet-interactief: `type=fd` = één partitie over de hele schijf, type fd (Linux raid autodetect). Met fdisk: n, p, 1, Enter, Enter, t, fd, w.
- `--spare-devices=1` (of `-x 1`) = reserveschijf die overneemt als een actieve schijf wegvalt.
- Bruikbare ruimte RAID 5: **(aantal actieve schijven − 1) × grootte** = (3 − 1) × 8 = 16 GB. De spare telt niet mee.
- Meteen na het maken: `recovery = 19%` en `[3/2] [UU_]`: mdadm bouwt de pariteit op. Je kan intussen al verder werken.
- `mkfs`: "Superblock backups stored on blocks ...": ext4 bewaart kopieën van de superblock, zodat fsck een beschadigde eerste kan herstellen.

### /proc/mdstat lezen

| Stuk | Betekenis |
|---|---|
| `active raid5` | array actief, niveau 5 |
| `sde1[3](S)` | (S) = spare |
| `[3/3] [UUU]` | 3 nodig, 3 in orde (U = up) |
| `[3/2] [UU_]` | degraded: één schijf ontbreekt (`_`) |
| `recovery = 19.1%` | heropbouw/pariteit bezig |

In `lsblk`: schijf → partitie → `md0` (TYPE raid5, major 9 = md-driver).

### Schijven weghalen

```
sudo dd if=/dev/urandom of=/srv/raid/bigfile bs=1M count=1024 status=progress
sha256sum /srv/raid/bigfile | tee ~/bigfile.sha256
# VM stoppen, schijf uitcommentariëren in vagrant-hosts.yml, opnieuw starten:
sed -i "/name: 'lab02'/,+1 s/^/#/" vagrant-hosts.yml
lsblk; cat /proc/mdstat; df -h /srv/raid
sha256sum -c ~/bigfile.sha256      # /srv/raid/bigfile: OK
```

| Weggehaald | Toestand |
|---|---|
| 1e schijf | spare neemt over tijdens het opstarten → weer `[UUU]`, geen spare meer |
| 2e schijf | **degraded** `[UU_]`, data nog intact, geen bescherming meer |
| 3e schijf | array kapot: RAID 5 overleeft geen twee ontbrekende actieve schijven |

- De **schijfnamen verschuiven** na het weghalen (sdd werd sdc, sde werd sdd). mdadm herkent de schijven aan hun RAID-metadata, en dankzij de **UUID** in fstab wordt `/srv/raid` toch correct gemount.
- Lezen uit een degraded RAID 5 is trager: de ontbrekende data wordt per blok uit de pariteit berekend.
- `/dev/urandom` i.p.v. `/dev/zero`: willekeurige data, geen nullen.
- `sha256sum -c` = vergelijkt met de bewaarde checksum: `OK` = geen enkele bit veranderd.
- Andere nuttige commando's: `sudo mdadm --detail /dev/md0`, `sudo mdadm --stop /dev/md0`, schijf vervangen: `sudo mdadm /dev/md0 --add /dev/sde1 --fail /dev/sdb1 --remove /dev/sdb1`.

## H11 DNS met BIND (labo)

Labo-omgeving: `~/linux-training-labs/dns-bind`, domein `linuxtrn.lan`, netwerk 172.16.76.0/24.

| Host | IP | Rol |
|---|---|---|
| srv001 (ns1) | 172.16.76.251 | primaire DNS (volledig door provisioning ingesteld) |
| srv002 (ns2) | 172.16.76.252 | secundaire DNS (zelf ingesteld) |
| srv003 (smtp, imap) | 172.16.76.3 | mailserver (enkel in de zone) |
| srv004 (www) | 172.16.76.4 | webserver (enkel in de zone) |

172.16.76.0/24 moet toegelaten zijn in VirtualBox:

```
echo '* 192.168.0.0/16 172.16.0.0/12 fe80::/10' | sudo tee /etc/vbox/networks.conf
```

### Primaire server (srv001)

`/etc/named.conf` (provisioning):
- `listen-on port 53 { any; };` + `allow-query { any; };`: bereikbaar vanaf het netwerk (standaard enkel 127.0.0.1 en localhost)
- `forwarders { 1.1.1.1; 1.0.0.1; };`: Cloudflare als forwarder
- `recursion yes;`
- `zone "linuxtrn.lan" IN { type primary; file "linuxtrn.lan"; ... };` → zonebestand in `/var/named/` (de `directory`)

Zone transfer toelaten naar de secundaire en die verwittigen bij wijzigingen:

```
sudo cp /etc/named.conf /etc/named.conf.bak
sudo sed -i '/allow-update { none; };/a\	allow-transfer { 172.16.76.252; };\n\tnotify yes;' /etc/named.conf
sudo named-checkconf && sudo systemctl restart named
sudo rndc querylog on && sudo journalctl -flu named      # tweede terminal: logs volgen
```

- `rndc querylog on` geldt enkel tot de volgende herstart van named.

### Testen vanaf de laptop

```
dig +short @172.16.76.251 www.linuxtrn.lan     # srv004.linuxtrn.lan. + 172.16.76.4 (CNAME, dan A)
dig +short @172.16.76.251 linuxtrn.lan          # 172.16.76.4 (A-record voor @)
dig +short @172.16.76.251 MX linuxtrn.lan       # 10 srv003.linuxtrn.lan.
dig +short @172.16.76.251 NS linuxtrn.lan       # srv001 en srv002
```

### Secundaire server (srv002)

```
sudo dnf install -y bind bind-utils
sudo systemctl enable --now firewalld
sudo firewall-cmd --add-service=dns --permanent && sudo firewall-cmd --reload
sudo systemctl enable --now named                 # op EL niet automatisch gestart
sudo cp /etc/named.conf /etc/named.conf.bak
sudo sed -i -e 's/listen-on port 53 { 127.0.0.1; };/listen-on port 53 { any; };/' \
            -e 's/listen-on-v6 port 53 { ::1; };/listen-on-v6 port 53 { any; };/' \
            -e 's/allow-query     { localhost; };/allow-query     { any; };/' \
            -e 's/recursion yes;/recursion no;/' /etc/named.conf
sudo tee -a /etc/named.conf <<'EOF'

zone "linuxtrn.lan" IN {
	type secondary;
	primaries { 172.16.76.251; };
	file "slaves/linuxtrn.lan";
	allow-transfer { none; };
};
EOF
sudo named-checkconf && sudo systemctl restart named
sudo ls -l /var/named/slaves/                     # linuxtrn.lan, eigenaar named
```

- `recursion no` op de secundaire: enkel autoritatief (best practice).
- `primaries` = waar de zone opgehaald wordt. `file "slaves/..."` = de ontvangen kopie in `/var/named/slaves/` (schrijfbaar voor named).

Log van de primaire bij de eerste start van de secundaire:

| Logregel | Betekenis |
|---|---|
| `query: linuxtrn.lan IN SOA -E(0)` | secundaire vraagt eerst de SOA om serienummers te vergelijken |
| `query: linuxtrn.lan IN AXFR -T` | volledige zone transfer, `-T` = via **TCP** |
| `AXFR ended: ... 13 records ... (serial 24100102)` | de hele zone is overgezet |

- Gewone DNS-vragen gaan via **UDP** 53, zone transfers via **TCP** 53.

### AXFR-test (beveiliging)

```
dig @172.16.76.251 AXFR linuxtrn.lan +short | head -5   # Transfer failed (enkel .252 toegelaten)
dig @172.16.76.252 AXFR linuxtrn.lan +short | head -5   # zonder allow-transfer: de hele zone!
```

- **Valkuil:** zonder `allow-transfer` laat BIND 9.16 een zone transfer toe aan **iedereen**. De secundaire lekte zo de hele zone. Daarom `allow-transfer` op **elke** server zetten; een secundaire die niets doorgeeft: `allow-transfer { none; };`.

### Serienummer-valkuil

```
# record wijzigen op de primaire ZONDER serienummer te verhogen:
sudo sed -i 's/^srv004    IN  A      172.16.76.4$/srv004    IN  A      172.16.76.44/' /var/named/linuxtrn.lan
sudo named-checkzone linuxtrn.lan /var/named/linuxtrn.lan && sudo systemctl restart named
for s in 251 252; do dig +short @172.16.76.$s srv004.linuxtrn.lan; done    # .44 op primaire, .4 op secundaire!
# serienummer verhogen (conventie JJMMDDnn):
sudo sed -i 's/24100102  ; Serial/26112501  ; Serial/' /var/named/linuxtrn.lan
sudo named-checkzone linuxtrn.lan /var/named/linuxtrn.lan && sudo systemctl restart named
for s in 251 252; do dig +short @172.16.76.$s srv004.linuxtrn.lan; done    # beide .44
```

- Bij een herstart stuurt de primaire een **notify**. De secundaire vraagt de SOA en vergelijkt het **serienummer**: hetzelfde = "niets veranderd", geen transfer.
- Regel: na **elke** wijziging aan een zonebestand het serienummer verhogen.
- Log na de verhoging: `sending notifies (serial 26112501)` → `IXFR version not in journal, falling back to AXFR` → `AXFR-style IXFR ended ... 13 records`. De secundaire vroeg een **IXFR** (enkel de wijzigingen), maar de primaire had geen journaal van de wijziging (we pasten het bestand rechtstreeks aan), dus werd het een volledige transfer.
