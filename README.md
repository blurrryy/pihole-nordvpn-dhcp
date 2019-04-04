# Einrichten des Raspberry Pi als NordVPN Gateway, zusammen mit PiHole unter Nutzung des PiHole-Integrierten DHCP-Servers

## Vorwort

Ich empfehle unter Windows als Terminal-Programm unter Git-Bash, da es mit cmder bei mir zu Anzeigeproblemen kam.

Download: https://gitforwindows.org/

## 1. Setup des Raspberry PI

### Download des Raspbian Lite Images

Aktuelles Raspbian Lite Image herunterladen

Download: https://downloads.raspberrypi.org/raspbian_lite_latest

### Tools zum weiteren Vorgehen

- MiniTool Partition Wizard Free
  https://www.partitionwizard.com/free-partition-manager.html
- Etcher
  https://www.balena.io/etcher/

### Vorbereiten der SD-Karte

Sofern die SD-Karte bereits schon einmal verwendet wurde, öffne ich MiniTool Partition Wizard und lösche alle Partitionen auf der SD-Karte und erstelle eine neue FAT32 Partition über die gesamte SD-Karte

### Flashes des Images

Ich öffne Etcher, wähle das Raspian Image aus und Flash es auf die SD-Karte

### SSH-Zugriff aktivieren

Bei den neueren Raspian-Installationen ist der SSH-Zugriff standardmäßig deaktiviert... Ich benötige also einen Monitor bzw. eine Tastatur am Raspi.

Um das zu verhindern, kann ich eine leere Datei mit dem Namen "ssh" auf die Boot-Partition der SD-Karte schreiben.

Sofern die boot-Partition nicht angezeigt wird unter Windows, öffne ich MiniTool Partition Wizard, rechtsklicke auf die boot-Partition der SD-Karte und teile dieser einen Laufwerksbuchstaben (in meinem Beispiel H:) zu.

Danach klicke ich auf übernehmen und kann auf die boot-Partition unter Windows zugreifen.

Ich erstelle eine leere Datei ohne Endung mit dem Namen "ssh" im root-Verzeichnis der boot-Partition.

Ich kann z.B. Git-Bash öffnen und wechsle dort zuerst auf Laufwerk H: mit

```bash
cd /h
```

und erstelle das File mit

```bash
touch ssh
```

Danach ist das Raspian Image fertig zum booten...

## 2. Raspberry booten, Login, Standard-Setup

Karte in den Raspi und starten. Der Raspi sollte im Netzwerk sein und vom Router DHCP eine IP zugewiesen kriegen. Es kann ein weniger Tricky sein, die IP des Raspi herauszubekommen, falls man ihn über den Hostname "raspberrypi" nicht im Netzwerk findet, aber die meisten Webinterfaces der Router (z.B. vom Speedport unter http://speedport.ip/) stellen eine Liste der verbundenen Geräte zur verfügung, weshalb man die IP-Adresse des Raspberry eigentlich relativ problemlos herausbekommt.

Ich öffne also Git Bash und logge mich auf dem Raspberry ein.
Sollte der Hostname verfügbar sein, ist dies mit

```bash
ssh pi@raspberrypi
```

möglich. Ansonsten nutze ich die IP-Adresse des Raspberry, welche ich vorher in Erfahrung gebracht habe, z.B. 192.168.2.108 (eine typische Speedport-DHCP-IP).

```bash
ssh pi@192.168.2.108
```

Das Standard-Passwort des RaspberryPi lautet

```
raspberry
```

### Start-Einstellungen des RaspberryPi

Zunächst möchte ich das Dateisystem auf die gesamte SD-Karte auslagern und mein Passwort ändern.
Alternativ kann ich auch einen neuen Nutzer erstellen und den pi-Nutzer entfernen. (Anleitung ist einfach zu googlen)
Ebenfalls kann ich den Hostname des RaspberryPi ändern, wenn ich dies möchte.
Am besten eignet sich dazu das eingebaute Config-Programm.

```bash
pi@raspberrypi:~ $ sudo raspi-config
```

Unter "Change User Password" ändere ich das Passwort.
Im Anschluss gehe ich auf die "Advanced Options" und wähle "Expand File System"

Unter "Network Options" kann ich außerdem (optional) den Hostname des RaspberryPi ändern. (Da wir später den DNS-Server auf dem RaspberryPi laufen haben, ist dies jedoch prinzipiell nicht so wichtig)

Danach verlasse ich den Konfigurator und starte den RaspberryPi einmal neu.

```bash
sudo reboot
```

### Setzen einer statischen IP-Adresse

Da der RaspberryPi später einmal als Gateway und DNS-Server agiert, empfiehlt es sich, eine statische IP-Adresse für das Netzwerk festzulegen.

Wie [hier](https://www.elektronik-kompendium.de/sites/raspberry-pi/1912151.htm) beschrieben, nutze ich dazu am besten die DHCP Client Daemon Variante.

Ich überprüfe, ob der dhcpcd läuft.

```bash
sudo service dhcpcd status
```

Sollte dieser nicht als "active" gekennzeichnet sein, starte ich ihn.

```bash
sudo service dhcpcd start
```

Nun editiere ich die IPv4-Konfiguration des DHCPCD

```bash
sudo nano /etc/dhcpcd.conf
```

Meistens ist dort eine Zeile schon entsprechend vorbereitet, die ich nur auskommentieren muss.
Ich weiß, dass mein Speedport Router IPs zwischen 192.168.2.101 und 192.168.2.199 vergibt, außerdem weiß ich, dass mein Router selbst die IP
192.168.2.1 besitzt, welche auch als DNS-Server agiert, deshalb entscheide ich mich dazu, eine statische IP im gleichen IP-Bereich, aber außerhalb der DHCP-Vergabe zu wählen und entscheide mich für 192.168.2.2.

```shell
interface eth0
static ip_address=192.168.2.2/24
static routers=192.168.2.1
static domain_name_servers=192.168.2.1 8.8.8.8
```

static routers gibt dabei das Gateway, also die IP meines Hardware-Routers an, ebenfalls gebe ich als DNS-Server meinen Router an und als Ausweich den DNS-Server von Google 8.8.8.8. Das wird später von unserem VPN-Client bzw. PiHole ohnehin gesteuert.

nano verlasse ich durch Strg+O, Enter, Strg+X

Wenn ich diese Einstellungen getätigt habe, muss ich, da ich per SSH verbunden bin, bestenfalls erneut neustarten.

```bash
sudo reboot
```

Wenn der Raspi neu gestartet hat, ist er nun bestenfalls unter der neuen, von mir zugewiesenen IP-Adresse erreichbar.

```bash
ssh pi@192.168.2.2
```

## 3. Update aller installierten Packages

Bevor ich nun fortfahre und alles benötigte installiere, stelle ich sicher, dass alles auf dem Raspberry auf dem neusten Stand ist. Ich führe nacheinander folgendes aus:

```bash
sudo apt-get update
sudo apt-get upgrade
```

## 4. Installation von NordVPN

Dazu gehe ich grundsätzlich wie auch [hier](https://www.instructables.com/id/Raspberry-Pi-VPN-Gateway-NordVPN/) beschrieben vor:

### OpenVPN installieren

Als erstes muss OpenVPN installiert werden, damit wir zu einem VPN-Server verbinden können.

```bash
sudo apt-get install openvpn
```

Danach starten wir openvpn

```bash
sudo systemctl enable openvpn
```

### NordVPN Setup

NordVPN stellt glücklicherweise bereits Standard-Konfigurations-Files bereit, auf die wir zugreifen können.

Wir wechseln in das Verzeichnis von OpenVPN und laden diese heruter und entpacken sie.

```bash
cd /etc/openvpn
sudo wget https://nordvpn.com/api/files/zip
sudo unzip zip
sudo rm zip
```

Danach müssen wir uns für einen der NordVPN-Server entscheiden.
Dazu können wir uns die [Server-Liste](https://nordvpn.com/servers) von NordVPN anschauen und uns einen Server heraussuchen, bei dem ein OpenVPN-UDP vorhanden ist.
Ich wähle bei den recommended Servers Germany, Standard VPN und OpenVPN UDP aus und mir wird "de540.nordvpn.com" vorgeschlagen.

Ich muss also schauen, ob dafür ein Konfigurations-File vorhanden ist und finde im Ordner auch die entsprechende Datei:

```
ls -l
```

Dort ist die:

```
de540.nordvpn.com.udp1194.ovpn
```

Vorher muss ich jedoch meinen NordVPN Nutzername und Passwort noch speichern.
Dies erledige ich in einer separaten Datei.

```bash
sudo nano /etc/openvpn/nordvpn_auth.txt
```

Dort beschreibe ich die ersten 2 Zeilen

```bash
username
password
```

Strg+O, Enter, Strg+X und raus da.
Wir sichern die Datei ab:

```bash
chmod 600 /etc/openvpn/nordvpn_auth.txt
```

Als nächstes nennen wir unsere gefundene Konfig-Datei um, um Sie vom Autostart aufrufbar und für uns besser auffindbar zu machen:

```bash
sudo mv /etc/openvpn/de540.nordvpn.com.udp1194.ovpn server.conf
```

Als nächstes editieren wir die Datei, um auf unsere Login-Daten zugreifen zu können.

```bash
sudo nano server.conf
```

Dort erweitern wir die Zeile auth-user-pass um den Pfard zu unserer Login-Textdatei:

```
auth-user-pass /etc/openvpn/nordvpn_auth.txt
```

Strg+O, Enter, Strg+X und raus da.

Danach starten wir unseren OpenVPN neu.

```bash
sudo systemctl restart openvpn@server
```

Wenn alles gut geht, sind wir nun mit dem NordVPN verbunden.

Wir sollten ein neues Tunnel-Netzwerk "tun0" sehen beim Aufruf von

```bash
ifconfig
```

Außerdem sollte der Raspberry nun nach außen eine andere IP haben als unser Rechner.

```bash
wget  http://ipinfo.io/ip  -qO -
```

Sollte es Probleme geben bzw. kein Tunnel-Netzwerk vorhanden sein, empfiehlt es sich, erneut neuzustarten oder die Einstellungen noch einmal zu überprüfen.

## 5. Routing und Firewall

Da der Raspberry als VPN Gateway genutzt werden soll, benötigen wir die Möglichkeit, dass dieser auch als Router agiert.
Danach müssen wir noch Firewall Einstellungen setzen, damit unser Netz geschützt ist und sämtliche Netzwerkstreams getunnelt werden.

### RaspberryPi als Router

Wir erlauben den Raspberry als Router zu agieren:

```bash
sudo /bin/su -c "echo -e '\n#Enable IP Routing\nnet.ipv4.ip_forward = 1' > /etc/sysctl.conf"
```

Überprüfen können wir die Einstellung mit

```bash
sudo sysctl -p
```

wo uns

```
net.ipv4.ip_forward = 1
```

angezeigt werden sollte.

### Raspberry als Firewall

Nun benötigen wir eine iptables Konfiguration, damit der Raspberry als Firewall agiert und den Netzwerktraffic richtig über das NordVPN Tunnel-Netzwerk leitet.

Als erstes aktivieren wir NAT, damit das richtige Gerät im Netzwerk auch den richtigen Traffic zugeordnet bekommt:

```bash
sudo iptables -t nat -A POSTROUTING -o tun0 -j MASQUERADE
```

Dann leiten wir den internen Netzwerktraffic eth0 über das Tunnel-Netzwerk tun0 und zurück.

```bash
sudo iptables -A FORWARD -i eth0 -o tun0 -j ACCEPT
sudo iptables -A FORWARD -i tun0 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

Letztere Option erlaubt nur eingehenden Traffic, wenn dieser vorher von einem Gerät innerhalb des Netzwerkes ausging.

Außerdem müssen wir den eigenen Loopback-Traffic des Raspis erlauben:

```bash
sudo iptables -A INPUT -i lo -j ACCEPT
```

Außerdem wollen wir im nächsten Schritt, dass man den Raspi weiterhin pingen und über SSH erreichen kann.

```bash
sudo iptables -A INPUT -i eth0 -p icmp -j ACCEPT
sudo iptables -A INPUT -i eth0 -p tcp --dport 22 -j ACCEPT
```

Und auch der Traffic vom Raspi darf zurückgegeben werden (siehe oben):

```bash
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
```

Als nächstes müssen wir noch, in Vorbereitung auf die Installation von PiHole, ein paar weitere Ports für das lokale Netzwerk freigeben, damit DHCP, DNS, eventuelle VPN-Verbindungen und das PiHole Webinterface auch funktionieren:

```bash
sudo iptables -I INPUT -i eth0 -p udp --dport 67:68 --sport 67:68 -j ACCEPT
sudo iptables -A INPUT -i eth0 -p tcp --destination-port 53 -j ACCEPT
sudo iptables -A INPUT -i eth0 -p udp --destination-port 53 -j ACCEPT
sudo iptables -A INPUT -i eth0 -p udp --destination-port 1194 -j ACCEPT
sudo iptables -A INPUT -i eth0 -p tcp --destination-port 1194 -j ACCEPT
sudo iptables -A INPUT -i eth0 -p tcp --destination-port 80 -j ACCEPT
```

Im Anschluss blocken wir sämtlichen anderen Netzwerk-Traffic, der keiner der vorher genannten Regeln entspricht:

```bash
sudo iptables -P FORWARD DROP
sudo iptables -P INPUT DROP
```

Danach können wir uns unser Werk betrachten und noch einmal nachvollziehen:

```bash
sudo iptables -L
```

### Speichern der Firewall-Einstellungen

Damit die Iptables auch nach dem Reboot erhalten bleiben, müssen wir noch ein kleines Hilfsprogramm installieren und die gerade gemachten Einstellungen speichern.

```bash
sudo apt-get install iptables-persistent
sudo systemctl enable netfilter-persistent
```

Der erste Befehl installiert das Programm, der zweite speichert die entsprechenden Einstellungen.

## 6. Installation von PiHole

Zunächst wechseln wir wieder in unser Home-Verzeichnis.

```bash
cd ~
```

und laden das PiHole Setup herunter:

```bash
wget -O basic-install.sh https://install.pi-hole.net
```

Danach führen wir das PiHole Setup aus:

```bash
sudo bash basic-install.sh
```

Das Setup ist ziemlich selbsterklärend, es gibt dort wenige Fallstricke.
Wichtig ist nur, dass wir für das genutzte Netzwerk unser internes Netzwerk wählen, d.h. eth0.
Außerdem wollen wir sicher gehen, dass wir als IP-Adressen - nicht wie meist vorgeschlagen die vom Tunnel (10.8.x.x) nutzen, sondern unsere statische IP im eth0, d.h. in meinem Beispiel.

```
192.168.2.2
```

Für das Gateway bzw. den Router nutzen wir nun das von uns neu erstellte Gateway des Raspberry selbst, was in den Tunnel und dann zum Hardware-Router führt, d.h. in meinem Beispiel ebenfalls

```
192.168.2.2
```

Ansonsten nutzen wir einfach die Standard-Einstellungen, da der Rest später sowieso im Webinterface noch angepasst werden wird.
Kurze Zeit später ist die Installation abgeschlossen, PiHole ist installiert und wir speichern uns die Zugangsdaten zum Webinterface ab.

## 7. Anpassen der OpenVPN-Einstellungen

PiHole ist nun installiert und läuft. Nun wird es Zeit, auch sämtlichen Tunnel-Traffic explizit auf unseren DNS-Server zu leiten, bevor er auf den Standard-DNS-Server von NordVPN weitergeleitet wird.
Wir öffnen also unsere OpenVPN-Konfiguration erneut.

```bash
sudo nano /etc/openvpn/server.conf
```

Am Ende des Files weisen wir explizit einen DNS-Server zu:

```
push "dhcp-option DNS 192.168.2.2"
```

Strg+O, Enter, Strg+X und raus da.

Danach starten wir unseren erneut OpenVPN neu.

```bash
sudo systemctl restart openvpn@server
```

## 8. Ändern der Einstellungen per PiHole Webinterface

Da wir derzeit von unserem Rechner noch nicht den PiHole DNS-Server nutzen, ist unser Webinterface höchstwahrscheinlich nicht unter http://pi.hole/admin erreichbar, wohl aber unter der IP-Adresse, in meinem Beispiel unter
http://192.168.2.2/admin

Dort loggen wir uns ein, da wir noch ein paar Einstellungen vornehmen müssen und den DHCP-Server aktivieren.

### Nutzen des NordVPN DNS-Servers

Bisher nutzen wir noch Google als unseren HauptDNS (sofern nicht anders in der PiHole Konfiguration bestimmt).

Wir klicken also im Webinterface auf >>Settings

Danach wählen wir oben im Settings-Panel >>DNS

Wir entfernen alle Haken bei den Standard "Upstream DNS Servers" und fügen bei Custom 1 (IPv4) und Custom 2 (IPv4) die Adressen der NordVPN DNS-Server hinzu:

```
103.86.96.100
```

sowie

```
103.86.99.100
```

Außerdem stellen wir das "Interface listening behavior" auf

```
Listen on all interfaces
```

um unser Netzwerk eth0 sowie unser OpenVPN Netzwerk tun0 abzudecken.

Danach scrollen wir ganz nach unten, um auf "Save" zu klicken.

### Aktivieren des PiHole DHCP-Servers

Wichtig ist es, dass wir nun, kurz bevor wir den DHCP-Server des PiHole aktivieren, den DHCP-Server unseres Routers deaktivieren, damit wir in unserem Netzwerk nurnoch einen DHCP-Server vorfinden und dieser auch von allen Geräten genutzt wird.

Da z.B. der Speedport standardmäßig über extrem lange Leasezeiten von z.B. 3 Wochen verfügt, kann es sein, dass wir, im Anschluss unserer Umstellun auf unseren Geräten explizit eine neue IP vom DHCP anfordern müssen.
Beim Iphone z.B. kann man dazu in den WLAN-Einstellungen "Lease erneuern" wählen, unter Windows kann man das Netzwerk dann kurz trennen oder auf "Diagnose" in den Adaptereinstellungen klicken, um den Lease zu erneuern.

Bevor wir dies jedoch tun, müssen wir den PiHole DHCP-Server konfigurieren und starten.

Wir setzen den Haken bei "DHCP server enabled" und legen unsere Range der zu vergebenen IP-Adressen fest. Dazu wähle ich Adressen intuitiverweise im bestehenden eth0 IP-Adressen Bereich.
Um auch zu sehen, dass der DHCP funktioniert und neue Adressen vergibt, entscheide ich mich in meinem Beispiel für einen höheren Bereich als den Bereich des Speedports:

```
From: 192.168.2.201
To: 192.168.2.251
```

Mein Router ist nun nicht mein Hardware-Router, sondern mein Raspberry, welcher ja als Tunnel-Gateway, demnach ebenso als Router fungiert:

```
Router: 192.168.2.2
```

Spätestens jetzt sollte ich den DHCP-Server des Hardware-Routers (Speedport/FritzBox) deaktivieren und im WebInterface auf Save klicken.

Gratulation. Alles ist nun konfiguriert.

### DHCP-Leases aktualisieren

Wenn ich nun die Leases erneuere, sollte mein Rechner nun eine IP-Adresse vom Raspberry PiHole DHCP-Server zugewiesen bekommen. (In meinem Beispiel eine IP-Adresse zwischen 192.168.2.201 und 192.168.2.251)

Wenn ich im PiHole Webinterface nun die Seite neu lade, sollte ich ebenfalls meinen Rechner bei DHCP leases sehen können.

Außerdem sollte ich beim Besuch der [NordVPN Website](https://nordvpn.com) nun den Status: Protected erhalten haben.

## 9. PiHole Blocklists erweitern

Als nächtes kann ich noch Urls von weitverbreiteten Blocklists unter dem Webinterface-Tab >>Blocklists
hinzufügen können.

Es eignet sich dabei z.B. die Kollektion von [Firebog](https://v.firebog.net/hosts/lists.php)

Auf der genannten Website sehe ich alle Urls Text pro Zeile und kann diese per Copy&Paste unterhalb der bestehenden Blocklists einfügen.

Ein Klick auf "Save and Update" scannt diese Listen und fügt die entsprechenden Urls hinzu. (Das kann schonmal ein paar Minuten dauern.)

## 10. Geblocktes Internet genießen

Die Konfiguration ist abgeschlossen und mein Netzwerk ist nun getunnelt und wird von PiHole überwacht.
