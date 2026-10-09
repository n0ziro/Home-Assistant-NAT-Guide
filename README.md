[README.md](https://github.com/user-attachments/files/33241094/README.md)
# Home Assistant unter Proxmox auf einem Raspberry Pi über WLAN: internes Netzwerk und NAT (nftables)

> **Getestet mit:** Raspberry Pi 4 · Debian 13 (Trixie) · Proxmox VE 9.2.10 · Home Assistant OS 18.3

**Ziel:** Eine virtuelle Maschine (VM) mit Home Assistant OS auf einem Raspberry Pi, der **über WLAN** mit dem Netzwerk verbunden ist, Zugang zum Internet hat und über einen Browser von anderen Geräten im selben WLAN-Netzwerk aus aufgerufen werden kann.

**Für wen:** Für alle, die bereits Raspberry Pi OS, Proxmox VE und eine VM mit Home Assistant OS installiert haben und nun den Pi von einer Kabelverbindung auf WLAN umstellen.

> Alle Adressen in dieser Anleitung sind Beispiele. Ersetze sie vor dem Start durch deine eigenen Werte.

## Inhaltsverzeichnis

- [1. Problem und Lösung](#1-problem-und-lösung)
- [2. Ausgangssituation und Variablen](#2-ausgangssituation-und-variablen)
- [3. Begriffe](#3-begriffe)
- [4. Schritt-für-Schritt-Einrichtung](#4-schritt-für-schritt-einrichtung)
  - [Schritt 0: Systeminformationen sammeln](#schritt-0-systeminformationen-sammeln)
  - [Schritt 1: Subnetz auswählen und Sicherungskopien erstellen](#schritt-1-subnetz-auswählen-und-sicherungskopien-erstellen)
  - [Schritt 2: vmbr0 in eine interne Brücke umwandeln](#schritt-2-vmbr0-in-eine-interne-brücke-umwandeln)
  - [Schritt 3: Die VM an die interne Bridge anschließen](#schritt-3-die-vm-an-die-interne-bridge-anschließen)
  - [Schritt 4: Adresse innerhalb der VM festlegen](#schritt-4-adresse-innerhalb-der-vm-festlegen)
  - [Schritt 5: Dem Pi das Weiterleiten von Paketen erlauben](#schritt-5-dem-pi-das-weiterleiten-von-paketen-erlauben)
  - [Schritt 6: Firewall-Regeln (nftables) erstellen](#schritt-6-firewall-regeln-nftables-erstellen)
  - [Schritt 7: Regeln anwenden und den automatischen Start aktivieren](#schritt-7-regeln-anwenden-und-den-automatischen-start-aktivieren)
  - [Schritt 8: Überprüfung nach dem Neustart](#schritt-8-überprüfung-nach-dem-neustart)
- [5. Rückgängig machen](#5-rückgängig-machen)

---

---

## 1. Problem und Lösung

### Warum man eine VM nicht einfach über WLAN „neben dem Pi“ aufsetzen kann

Im Standardaufbau ist die VM über eine virtuelle Netzwerkkarte mit der Netzwerkbrücke (dem virtuellen Switch vmbr0) verbunden. Die Brücke nutzt den Kabelanschluss eth0 als Port. Für das Netzwerk erscheint die VM daher als weiteres Gerät mit einer eigenen MAC-Adresse.
Ein WLAN-Access-Point (AP) arbeitet anders: Er akzeptiert von einem verbundenen Client nur Frames mit genau der MAC-Adresse, mit der sich der Client angemeldet hat. Frames der VM mit einer fremden MAC-Adresse verwirft er. Deshalb kann wlan0 nicht als Port zur Brücke hinzugefügt werden.

### Lösung

1. Für die VM wird **ein eigenes geschlossenes Netzwerk** auf einer Bridge ohne physische Ports eingerichtet.
2. Der Pi fungiert für die VM als **Router** und ist Teilnehmer in beiden Netzwerken: im externen WLAN und im internen Netz.
3. **NAT** erfüllt zwei Aufgaben:
   - **Masquerade** ersetzt bei Paketen von der VM nach außen die Absenderadresse durch die WLAN-Adresse des Pi;
   - **DNAT** ersetzt bei Anfragen aus dem WLAN die Zieladresse durch die Adresse der VM.

### Netzwerkschema

```text
Andere Geräte im WLAN
        |
        | http://<WLAN-IP-des-Pi>:80
        v
Raspberry Pi (wlan0)
        |
        | DNAT / Portweiterleitung
        v
Home Assistant OS (10.10.10.10:80)
        |
        | Standardgateway: 10.10.10.1
        v
Raspberry Pi (vmbr0) -- Masquerade --> Internet
```

Im Browser wird die Adresse des Raspberry Pi (z. B. `http://192.168.11.14`) geöffnet, nicht die Adresse der VM. Die Adresse `10.10.10.10` existiert nur im internen Netzwerk des Pi; von anderen Geräten ist sie nicht direkt erreichbar.

---

## 2. Ausgangssituation und Variablen

**Was bereits vorhanden sein sollte:**

- Ein Raspberry Pi mit Raspberry Pi OS und installiertem Proxmox VE.
- Eine laufende VM mit Home Assistant OS.
- Der Pi ist mit dem WLAN verbunden und verfügt über eine IP-Adresse im WLAN-Netzwerk.
- Zugriff auf den Pi per SSH **über WLAN** und Root-Rechte.

**Was muss in die Befehle eingegeben werden?**

| Variable | Beispiel in dieser Anleitung | So ermittelst du den Wert |
|---|---|---|
| Name der WLAN-Schnittstelle | `wlan0` | `ip -br a` |
| Adresse des Pi im WLAN-Netzwerk | `192.168.11.14` | `ip -br a show wlan0` |
| ID der virtuellen Maschine | `100` | `qm list` |
| Internes Subnetz | `10.10.10.0/24` | Frei wählbar; darf sich nicht mit WLAN, VPN oder Firmennetz überschneiden |
| Adresse des Pi im internen Netzwerk (Gateway) | `10.10.10.1` | Erste nutzbare Adresse des gewählten Subnetzes |
| VM-Adresse | `10.10.10.10` | Freie Adresse im internen Subnetz |
| Name der Netzwerkkarte innerhalb der VM | `enp0s18` | `ha network info` in der VM-Konsole |
| Port der Home-Assistant-Weboberfläche | `80` (in dieser Anleitung getestet) | `ha > core info` (Feld `port:`) |

- **Tipp:** Die WLAN-Adresse des Pi kommt per DHCP vom Router und kann sich ändern. Weise dem Pi im Router eine feste Adresse zu (DHCP-Reservierung).

---

---

## 3. Begriffe

- **IP-Adresse**: Die Adresse des Geräts im Netzwerk, zum Beispiel 10.10.10.10.
- **Subnetz** und Maske /24: Die Angabe 10.10.10.0/24 bedeutet: Die ersten 24 Bits (10.10.10) bilden den Netzanteil und sind bei allen Geräten des Subnetzes gleich. Der letzte Teil (Hostanteil, nutzbar 1–254) unterscheidet die Geräte. Geräte im selben Subnetz kommunizieren direkt miteinander.
- **Standardgateway**: Die Adresse des „Tors nach außen“. Pakete, die an Adressen außerhalb des eigenen Subnetzes gerichtet sind, sendet das Gerät an das Gateway. Für VMs ist das Gateway der Pi.
- **Private Adressen**: die Bereiche 10.x.x.x, 172.16–31.x.x, 192.168.x.x. Diese werden im Internet nicht weitergeleitet und können daher in internen Netzwerken verwendet werden.
- **Netzwerkbrücke**: Ein virtueller Switch unter Linux, an den die virtuellen Netzwerkkarten der VM angeschlossen werden.
- **NAT, Netzwerkadressübersetzung**: Ersetzen von Adressen in Paket-Headern. Wird die Absenderadresse ersetzt, spricht man von **Quell-NAT (SNAT, Masquerade)**; wird die Zieladresse ersetzt, von **Ziel-NAT (DNAT)**. **DNAT** für einen bestimmten Port nennt man **Portweiterleitung** (Port Forwarding).
- **Paketfilter**: Ein Programm im Kernel, das jedes Paket anhand von Regeln überprüft. In modernen Linux-Systemen ist dies **nftables**.
- **Verbindungsverfolgung, conntrack**: Eine Tabelle im Kernel, die alle aktuellen Verbindungen speichert. Dadurch erkennt der Kernel Antwortpakete und macht die Adressersetzung bei ihnen automatisch rückgängig.
- **DHCP**: Ein Dienst, der IP-Adressen automatisch vergibt. Im internen Netzwerk dieses Leitfadens gibt es keinen DHCP-Server, daher wird die Adresse der VM manuell festgelegt (**statische Adresse**).
- **MAC-Adresse:** Die feste Hardware-Adresse einer Netzwerkkarte (Schicht 2). Sie gilt nur im lokalen Netz; die IP-Adresse dagegen kann über Netzwerke hinweg geroutet werden.
- **Frame**: Die Dateneinheit auf Schicht 2 (Ethernet/WLAN). Ein Frame enthält MAC-Adressen, ein Paket (Schicht 3) enthält IP-Adressen.
- **Hook**: Ein Einhängepunkt auf dem Weg eines Pakets durch den Kernel, an dem nftables-Ketten ausgeführt werden (z. B. prerouting, forward, postrouting).
- **Policy**: Das Standardverhalten einer Kette (accept oder drop) für Pakete, auf die keine Regel zutrifft.

---

## 4. Schritt-für-Schritt-Einrichtung

### Schritt 0. Systeminformationen sammeln

```
ip -br a                    # alle Schnittstellen und ihre Adressen
ip route                    # Routen; die Standardroute sollte über wlan0 verlaufen
qm list                     # Liste der VMs und ihrer IDs
qm config 100 | grep ^net   # Netzwerkkarte der VM und ihre MAC-Adresse
```

**Was wir uns ansehen:**

- wlan0 befindet sich im Status UP und hat eine Adresse. Falls nicht, verbinde den Pi zunächst mit dem WLAN.
- In ip route gibt es die Zeile default via ... dev wlan0. Über diese Route wird die VM eine Verbindung zum Internet herstellen.
- In der Zeile net0 der VM ist die MAC-Adresse angegeben (im Format virtio=BC:24:11:..:..:..). **Notiere sie dir**, du wirst sie später benötigen.

### Schritt 1. Subnetz auswählen und Sicherungskopien erstellen

**Auswahl des Subnetzes.** Das interne Netzwerk darf sich weder mit dem WLAN-Netzwerk noch mit dem VPN noch mit Unternehmensnetzwerken überschneiden, auf die der Pi zugreifen kann. Wenn zwei Netzwerke denselben Adressbereich haben, weiß der Kernel nicht, wohin er das Paket senden soll. In dieser Anleitung wird 10.10.10.0/24 verwendet.

**Sicherungskopien:**

```
cp /etc/network/interfaces /etc/network/interfaces.bak
cp /etc/nftables.conf /etc/nftables.conf.bak
```

Sollte etwas schiefgehen, können die Originaldateien wiederhergestellt werden.

### Schritt 2. vmbr0 in eine interne Brücke umwandeln

Öffne die Datei:

```
nano /etc/network/interfaces
```

Der Abschnitt für vmbr0 und eth0 sollte wie folgt aussehen:

```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet manual

auto vmbr0
iface vmbr0 inet static
    address 10.10.10.1/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0
```

**Was bedeutet jede Zeile:**

| Zeile | Bedeutung |
|---|---|
| `iface eth0 inet manual` | Dem Kabelanschluss wird keine Adresse zugewiesen; Linux konfiguriert ihn nicht mit einer IP-Adresse. |
| `auto vmbr0` | Die Brücke wird beim Systemstart aktiviert. |
| `inet static` | Die IPv4-Adresse wird manuell festgelegt. |
| `address 10.10.10.1/24` | Adresse des Pi im internen Netzwerk, gleichzeitig das Gateway für die VM. |
| `bridge-ports none` | Es ist kein physischer Port an die Bridge angeschlossen. Das Netzwerk ist geschlossen und nur für VMs sichtbar. |
| `bridge-stp off` | Das **Spanning-Tree-Protokoll** schützt vor Schleifen zwischen Switches; hier können keine Schleifen entstehen. |
| `bridge-fd 0` | **Forward Delay** 0 Sekunden: Der Port wird sofort aktiv. |

> **Achtung:** Die SSH-Sitzung muss über WLAN laufen. Wenn du über ein Kabel verbunden bist, wird die Verbindung durch das Ändern von eth0 unterbrochen.

Anwenden:

```
ifreload -a
```

ifreload -a liest die Datei ein und passt die Schnittstellen entsprechend an. Falls ein solcher Befehl nicht vorhanden ist: ifdown vmbr0; ifup vmbr0.
**Überprüfung:**

```
ip -br a show vmbr0
```

Es sollte die Adresse 10.10.10.1/24 angezeigt werden. Der Status UP wird angezeigt, wenn eine aktive VM mit der Bridge verbunden ist. Wenn die VM angehalten ist, kann der Status UNKNOWN oder NO-CARRIER lauten, was normal ist: Die Adresse ist in diesem Fall dennoch zugewiesen.

### Schritt 3. Die VM an die interne Bridge anschließen

```
qm config 100 | grep net0
qm set 100 --net0 virtio=<MAC-Adresse aus dem vorherigen Befehl>,bridge=vmbr0
qm reboot 100
```

**Was bedeuten die einzelnen Teile des Befehls:**

- qm – Verwaltung der VM in Proxmox, 100 – Nummer der VM.
- net0 – erste virtuelle Netzwerkkarte (Nummerierung beginnt bei Null).
- virtio – schneller Typ einer virtuellen Karte, der „weiß“, dass er virtuell ist.
- bridge=vmbr0 – die Karte an eine Bridge anschließen.
- Die MAC-Adresse muss explizit angegeben werden, da der Befehl --net0 die Karte **als Ganzes** beschreibt. Ohne MAC-Adresse erstellt Proxmox eine neue, und MAC-gebundene Zuordnungen (z. B. DHCP-Reservierung) funktionieren dann nicht mehr.
-

Das lässt sich auch über die Weboberfläche erledigen: VM → **Hardware** → **Netzwerkgerät (net0)** → **Bearbeiten** → den Bridge-Eintrag vmbr0 auswählen.

### Schritt 4. Adresse innerhalb der VM festlegen

Öffne im Proxmox-Webinterface „VM“ → **Console**. Nach dem Laden erscheint die Eingabeaufforderung ha >.

```
ha network info
```

Suche in der Ausgabe den Namen der Netzwerkkarte (z. B. enp0s18). Du kannst das überprüfen: Die MAC-Adresse mac: in dieser Ausgabe stimmt mit der MAC-Adresse aus qm config 100 überein.
**So setzt sich der Name** enp0s18 zusammen: en – Ethernet, p0 – PCI-Bus Nummer 0, s18 – Steckplatz 18. Proxmox schließt die erste virtuelle Netzwerkkarte an Steckplatz 18 an, daher lautet der Name in der Regel so.
Lege die statische Adresse **mit einem einzigen Befehl** fest:

```
ha network update enp0s18 --ipv4-method static --ipv4-address 10.10.10.10/24 --ipv4-gateway 10.10.10.1 --ipv4-nameserver 1.1.1.1
```

| Parameter | Bedeutung |
|---|---|
| `--ipv4-method static` | Adresse manuell festlegen, da es im internen Netzwerk keinen DHCP-Server gibt. |
| `--ipv4-address 10.10.10.10/24` | Adresse der VM und Subnetzmaske. |
| `--ipv4-gateway 10.10.10.1` | Gateway – Adresse des Pi. |
| `--ipv4-nameserver 1.1.1.1` | **DNS-Server**, der Website-Namen in IP-Adressen umwandelt. |

> Alle Parameter werden mit **einem einzigen Befehl** festgelegt. Wenn nur DNS übergeben wird, gibt das System die Fehlermeldung method 'manual' requires at least an address aus.

**Überprüfung:** ha network info sollte folgende Werte anzeigen: address: 10.10.10.10/24, gateway: 10.10.10.1, method: static. Überprüfe die Verbindung zwischen Pi und VM (Führe den folgenden Befehl auf dem Pi aus, nicht in der VM-Konsole):

```
ping -c 3 10.10.10.10
```

Die Antworten bedeuten, dass das interne Netzwerk funktioniert. Der Internetzugang funktioniert noch nicht, da NAT noch nicht konfiguriert ist.

### Schritt 5. Dem Pi das Weiterleiten von Paketen erlauben

Standardmäßig verwirft der Linux-Kernel Pakete, die nicht für ihn bestimmt sind. Der Parameter ip_forward verwandelt den Pi in einen Router. Ohne ihn funktioniert keine der NAT-Regeln.

```
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-ipforward.conf
sysctl --system
sysctl net.ipv4.ip_forward
```

Der letzte Befehl sollte net.ipv4.ip_forward = 1 zurückgeben. Die Dateien im Verzeichnis /etc/sysctl.d/ werden bei jedem Systemstart gelesen, sodass die Einstellung beibehalten wird.

### Schritt 6. Firewall-Regeln (nftables) erstellen

#### Wie nftables aufgebaut ist

Die Regeln sind in drei Ebenen gegliedert:

- **Tabelle**: Container. Den Namen legst du selbst fest. Die Tabelle hat eine **Adressfamilie**: ip – nur IPv4, inet – IPv4 und IPv6 zusammen.
- **Kette**: Eine Liste von Regeln, die an einen Punkt im Paketpfad durch den Kernel (**Einhängepunkt**) angehängt ist.
- **Regel**: Bedingungen und Aktion. Eine Regel wird nur ausgelöst, wenn **alle** Bedingungen gleichzeitig erfüllt sind. Die Regeln in der Kette werden von oben nach unten geprüft.

#### Die von uns verwendeten Hooks

| Hook | Wann wird er ausgelöst? | Wozu dient er? |
|---|---|---|
| `prerouting` | Das Paket ist gerade eingetroffen; die Route wurde noch nicht gewählt. | DNAT |
| `forward` | Das Paket ist nicht für den Pi selbst bestimmt und wird durchgeleitet. | Filterung |
| `postrouting` | Das Paket verlässt gleich den Pi. | Masquerade |

DNAT wird im prerouting ausgeführt, da die neue Empfängeradresse bestimmt, an welche Schnittstelle das Paket weitergeleitet wird, während die Route erst später ausgewählt wird. Masquerade wird im postrouting ausgeführt, da erst ganz am Ende feststeht, über welche Schnittstelle das Paket versendet wird.

#### Prioritäten

**Priorität** legt die Reihenfolge der Ketten an einem Hook fest: Die niedrigere Zahl wird zuerst ausgeführt. Die Schlüsselwörter entsprechen den Zahlen aus dem Kernel:

| Begriff in nft | Zahl | Wofür |
|---|---:|---|
| `dstnat` | `-100` | Ersetzen der Empfängeradresse |
| `filter` | `0` | Filterung |
| `srcnat` | `100` | Ersetzen der Absenderadresse |

Die Zahlen in einer laufenden Konfiguration können mit dem Befehl nft -y list ruleset angezeigt werden.

#### Konfigurationsdatei

Öffne /etc/nftables.conf und **ersetze den gesamten Inhalt**:

```
#!/usr/sbin/nft -f

# Wir löschen nur unsere eigenen Tabellen. Fremde Regeln (Docker, Proxmox) lassen wir unberührt.
table ip haos_nat
flush table ip haos_nat
table inet haos_filter
flush table inet haos_filter

table ip haos_nat {
    chain prerouting {
        type nat hook prerouting priority dstnat; policy accept;
        iifname "wlan0" tcp dport 80 counter dnat to 10.10.10.10:80
    }

    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        ip saddr 10.10.10.0/24 oifname "wlan0" counter masquerade
    }
}

table inet haos_filter {
    chain forward {
        type filter hook forward priority filter; policy drop;

        # Ungültige Pakete sofort verwerfen
        ct state invalid counter drop

        # Antworten auf bestehende Verbindungen (beide Richtungen)
        ct state established,related counter accept

        # VM -> Internet (neue Verbindungen)
        iifname "vmbr0" oifname "wlan0" ip saddr 10.10.10.0/24 ct state new counter accept

        # WLAN -> VM: nur die Weboberfläche (Zieladresse gilt NACH dem DNAT)
        iifname "wlan0" oifname "vmbr0" ip daddr 10.10.10.10 tcp dport 80 ct state new counter accept
    }
}
```

> Setze deine eigenen Werte ein: Name der WLAN-Schnittstelle (wlan0), Subnetz, VM-Adresse und Home Assistant-Port.

> Die Namen der Tabellen (haos_nat, haos_filter) können frei gewählt werden. Es sind eindeutige Namen erforderlich, um Überschneidungen mit Tabellen zu vermeiden, die von anderen Programmen erstellt werden. Falls auf dem Pi bereits Tabellen aus früheren Experimenten vorhanden sind, lösche diese: nft delete table ip nat.

#### Erläuterung der einzelnen Regeln

**Ausschnitt** table ip haos_nat – **Adressersetzung**

| Element | Funktion |
|---|---|
| `table ip haos_nat` | IPv4-Tabelle mit dem Namen `haos_nat`. |
| `type nat` | Die Kette kann Adressen ersetzen. |
| `hook prerouting` / `hook postrouting` | Punkt auf dem Paketweg, an dem die Kette ausgelöst wird. |
| `priority dstnat` / `srcnat` | Ausführungsreihenfolge unter den Ketten am selben Hook. |
| `policy accept` | **Standardverhalten:** Ein Paket, auf das keine Regel zutrifft, wird durchgelassen. |

**DNAT-Regel:** iifname "wlan0" tcp dport 80 counter dnat to 10.10.10.10:80

| Teil | Bedeutung |
|---|---|
| `iifname "wlan0"` | **Eingangsschnittstelle:** Das Paket kam über WLAN. |
| `tcp dport 80` | TCP-Protokoll, **Zielport** 80. |
| `counter` | **Zähler:** zählt Pakete und Bytes und hilft bei der Diagnose. |
| `dnat to 10.10.10.10:80` | Ersetzt die Empfängeradresse und den Port durch Adresse und Port der VM. |

Bedeutung: Eine Anfrage an Port 80 des Pi wird an die VM weitergeleitet.

**Masquerade-Regel:** ip saddr 10.10.10.0/24 oifname "wlan0" counter masquerade

| Teil | Bedeutung |
|---|---|
| `ip saddr 10.10.10.0/24` | **Quelladresse** aus dem internen Netzwerk. |
| `oifname "wlan0"` | **Ausgangsschnittstelle:** Das Paket wird über WLAN gesendet. |
| `masquerade` | Ersetzt die Absenderadresse durch die aktuelle Adresse von `wlan0`. |

Ohne Masquerading würde das externe Netzwerk ein Paket von einer unbekannten privaten Adresse 10.10.10.10 sehen und könnte nicht darauf antworten. masquerade unterscheidet sich von snat to \<Adresse> dadurch, dass es die aktuelle Adresse der Schnittstelle übernimmt. Das ist wichtig, da das WLAN seine Adresse über DHCP bezieht und sich diese ändern kann.

**Ausschnitt aus `table inet haos_filter` – Filterung**
Die Kette forward verarbeitet nur Pakete, die **durch** den Pi laufen. type filter bedeutet „durchlassen oder nicht“, nicht das Ersetzen von Adressen.

| Nr. | Regel | Bedeutung |
|---:|---|---|
| 1 | `ct state invalid … drop` | Pakete, die zu keiner bekannten Verbindung passen (beschädigt oder fremd), werden sofort verworfen. |
| 2 | `ct state established,related … accept` | Antworten auf bereits begonnene Verbindungen dürfen in beide Richtungen passieren. |
| 3 | `vmbr0 → wlan0`, Absender aus dem internen Netz, `ct state new, accept` | Die VM darf neue Verbindungen nach außen aufbauen (Updates, Integrationen). |
| 4 | `wlan0 → vmbr0`, Ziel `10.10.10.10`, TCP-Zielport `80`, `ct state new, accept` | Ausnahme: neue Verbindungen zur Weboberfläche. Alles andere von außen wird durch `policy drop` verworfen. |

**Was bedeutet „ct state established,related“?:** „ct“ steht für **Connection Tracking**. Der Status established bedeutet, dass das Paket zu einer bereits bestehenden Verbindung gehört. Der Status related bedeutet, dass das Paket mit einer solchen Verbindung in Verbindung steht (z. B. eine Fehlermeldung).

Die Kette forward hat policy drop: Alles, was keine Regel ausdrücklich erlaubt, wird verworfen („Verboten ist, was nicht erlaubt ist“).

### Schritt 7. Regeln anwenden und den automatischen Start aktivieren

```
nft -c -f /etc/nftables.conf     # Syntaxprüfung, es werden keine Regeln angewendet
nft -f /etc/nftables.conf        # Regeln sofort laden
systemctl enable nftables        # Datei bei jedem Systemstart laden
systemctl is-enabled nftables    # Sollte „enabled“ zurückgeben
```

- -c (von *check*) überprüft die Datei lediglich. Wenn keine Ausgabe erfolgt, liegen keine Fehler vor. Bei einem Fehler zeigt nft die Zeilennummer an.
- Die nft-Befehle ändern die Regeln **im Kernel-Speicher**. Nach einem Neustart wird der Speicher gelöscht und die Regeln gehen verloren. Der Dienst nftables liest beim Systemstart die Datei /etc/nftables.conf, daher müssen die Regeln in dieser Datei gespeichert werden.

Geladene Regeln anzeigen:

```
nft list ruleset
```

### Schritt 8. Überprüfung nach dem Neustart

```
reboot
```

Nach einer Minute:

```
nft list ruleset
sysctl net.ipv4.ip_forward
ip -br a show vmbr0
```

Erwartetes Ergebnis:

- nft list ruleset zeigt die Tabellen haos_nat und haos_filter.
- sysctl net.ipv4.ip_forward gibt net.ipv4.ip_forward = 1 zurück.
- ip -br a show vmbr0 zeigt 10.10.10.1/24.

Funktionstest: Öffne auf einem anderen Gerät im WLAN http://192.168.11.14. Die Oberfläche von Home Assistant muss erscheinen. Danach zeigt nft list ruleset bei den Regeln, die gegriffen haben, steigende counter-Werte.
Wenn du alles zurücksetzen möchtest, lies den nächsten Abschnitt.

---

## 5. Rückgängig machen

So stellst du den vorherigen Zustand wieder her:

```
cp /etc/network/interfaces.bak /etc/network/interfaces
cp /etc/nftables.conf.bak /etc/nftables.conf
rm /etc/sysctl.d/99-ipforward.conf
ifreload -a
nft flush ruleset
nft -f /etc/nftables.conf
sysctl -w net.ipv4.ip_forward=0
```

Verbinde anschließend die VM wieder mit der bisherigen Bridge (Schritt 3) und aktiviere die automatische Adressvergabe: ha network update enp0s18 --ipv4-method auto.

---

**Version:** 1.0 · **Stand:** 08.10.2026
