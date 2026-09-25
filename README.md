# NetBird + Hetzner: SSH nur über das private Netz

Dieses Begleitmaterial zur NetBird/Hetzner-Folge zeigt einen kleinen Hetzner-VPS mit **OpenSSH über NetBird**. Auf der öffentlichen Serveradresse ist nach der Einrichtung kein SSH-Port erreichbar. Das Beispiel enthält keine Zugangsdaten und richtet keine Anwendung oder Reverse Proxy ein.

```text
Admin-Rechner ── NetBird / WireGuard ──> VPS: TCP 22
Internet      ── Hetzner Firewall ──X──> VPS: TCP 22
```

NetBird koordiniert die Geräte, verteilt deren öffentliche WireGuard-Schlüssel und setzt Zugriffsregeln um. WireGuard verschlüsselt den Verkehr zwischen berechtigten Peers. NetBird kann bei fehlender direkter Verbindung einen Relay nutzen; „privat“ bedeutet hier kontrollierter Zugang über das Overlay, keine Garantie für eine direkte Peer-Verbindung. NetBird Cloud ist in diesem Beispiel die Management-Ebene. **NetBird ersetzt weder SSH-Schlüssel noch Betriebssystem-Updates oder Backups.**

## Was du brauchst

- Ein eigenes Hetzner-Cloud-Projekt und einen eigenen NetBird-Account.
- Einen kleinen Ubuntu-VPS. Prüfe Architektur, Standort und den **angezeigten Gesamtpreis** vor dem Bestellen. Eine öffentliche IPv4 kann extra kosten. Ein vorhandenes freies Primary-IP-Objekt kannst du im passenden Standort wiederverwenden.
- Einen SSH-Schlüssel auf deinem Rechner. Optional liegt der private Schlüssel auf einem YubiKey; der **öffentliche** Schlüssel wird bei Hetzner hinterlegt. Der YubiKey ist kein Ersatz für die NetBird-Zugriffsregel.
- Deine aktuelle öffentliche IPv4 als `/32` für den **vorübergehenden** SSH-Bootstrap. Ermittle sie selbst; übernimm nie die IP eines Tutorials.

## I · Server mit engem Bootstrap-Zugang

1. Erstelle in Hetzner eine Firewall und weise sie dem neuen VPS **beim Anlegen** zu. Anfangs nur eingehend `TCP 22` von **deiner** öffentlichen `IPv4/32` erlauben. Falls du über IPv6 zugreifst, ergänze nur deine eigene `IPv6/128`. Keine Regel für `0.0.0.0/0` oder `::/0`. Keine weiteren eingehenden Ports.
2. Wähle Ubuntu, einen passenden Standort und deinen vorhandenen **öffentlichen** SSH-Schlüssel. Backups, zusätzliche Volumes und andere kostenpflichtige Optionen sind für dieses Netzwerkbeispiel nicht erforderlich.
3. Vergleiche den beim ersten SSH-Login angezeigten Host-Fingerabdruck über einen unabhängigen Weg mit der Hetzner-Konsole. Halte diese Bootstrap-Verbindung offen, bis ein **neuer** privater Login funktioniert.

Hetzner Cloud Firewalls verwerfen neuen eingehenden Verkehr ohne passende Allow-Regel. Ohne ausgehende Regeln bleibt ausgehender Verkehr erlaubt. Eine Hetzner Firewall schützt die **öffentliche** Serververbindung; NetBird regelt den Verkehr **im Overlay**. Beides ist nötig.

## II · Administrator und SSH absichern

Die folgenden Befehle laufen **auf dem VPS**. Ersetze den öffentlichen Schlüssel durch deinen eigenen, ohne den privaten Schlüssel auf den Server zu kopieren:

```bash
adduser ops
usermod -aG sudo ops
install -d -m 700 -o ops -g ops /home/ops/.ssh
printf '%s\n' 'DEIN_OEFFENTLICHER_SSH_SCHLUESSEL' > /home/ops/.ssh/authorized_keys
chown ops:ops /home/ops/.ssh/authorized_keys
chmod 600 /home/ops/.ssh/authorized_keys
```

Öffne **ein zweites Terminal** und prüfe den neuen `ops`-Login und `sudo`. Schließe die erste Root-Verbindung noch nicht. Übertrage dann [`ssh/00-private-admin.conf`](ssh/00-private-admin.conf) nach `/etc/ssh/sshd_config.d/00-private-admin.conf`, prüfe `sshd -t` und die effektiven Werte mit `sshd -T`, und lade SSH erst danach neu. `AllowUsers ops` ist ein Beispiel für **einen** Administrator. Lege einen zweiten Wiederherstellungsweg fest, bevor du weitere Konten sperrst.

```bash
sudo sshd -t
sudo sshd -T | grep -E '^(permitrootlogin|passwordauthentication|kbdinteractiveauthentication|authenticationmethods|allowusers|allowtcpforwarding) '
sudo systemctl reload ssh
```

Ein YubiKey mit FIDO2-SSH-Schlüssel (`ed25519-sk` oder `ecdsa-sk`) kann die Freigabe des privaten Schlüssels an die Berührung des Sticks binden. Unterstützte Hardware und Clientsoftware sind Voraussetzung. Ein normaler Ed25519-Schlüssel funktioniert für dieses Beispiel ebenfalls.

## III · NetBird verbinden und eng freigeben

1. Installiere den [offiziellen NetBird-Client](https://docs.netbird.io/get-started/install) auf deinem Rechner und auf dem VPS. Melde beide **im selben privaten Account** an. Für den Server eignet sich ein kurzlebiger, einmaliger Setup Key, der nur der Servergruppe zugewiesen ist. Er gehört nie ins Repo, Video oder Shell-History. Prüfe anschließend `netbird status` auf **beiden** Geräten.
2. Lege die Gruppen `video-admin` (**nur dein Rechner**) und `video-vps` (**nur dieser Server**) an. Prüfe alle weiteren Gruppenmitgliedschaften und vorhandenen Policies. Ein breites „All to All“ würde die enge Regel unterlaufen.
3. Lege **eine gerichtete** Allow-Policy an: Quelle `video-admin`, Ziel `video-vps`, Protokoll `TCP`, Port `22`. Nutze für dieses Beispiel **OpenSSH**; NetBirds optionaler eigener SSH-Server bleibt deaktiviert.
4. Lies die echte NetBird-IP und den Peer-DNS-Namen des VPS im Dashboard ab. Teste vom Admin-Rechner einen **neuen** SSH-Login über diesen Namen oder die Overlay-IP. Prüfe `hostname`, `whoami` und `sudo -v` (mit deinem `ops`-Passwort). DNS-Auflösung allein beweist noch keinen erlaubten Netzwerkzugriff.

## IV · Öffentlichen SSH-Zugang schließen

**Erst nach dem neuen privaten Login:** Entferne in der zugewiesenen Hetzner Firewall die vorübergehende eingehende TCP-22-Regel. Die Firewall bleibt dem Server zugewiesen und hat danach **null eingehende Allow-Regeln**. Öffne einen weiteren frischen SSH-Login über NetBird. Prüfe von einem externen Netz, dass `TCP 22` an öffentlicher IPv4 **und, falls erreichbar, IPv6** nicht verbindet. Ein Timeout vom eigenen Rechner ohne IPv6-Route beweist keine IPv6-Sperre.

Die Firewall ist die Grenze am öffentlichen Netz. Der SSH-Daemon kann technisch auf mehreren lokalen Adressen lauschen; die Aussage „SSH nur über NetBird“ beschreibt in diesem Aufbau den **erreichbaren Pfad**. Die NetBird-Policy begrenzt dabei die berechtigten Overlay-Peers, und der SSH-Schlüssel authentifiziert den Linux-Administrator. Prüfe nach einem Reboot den frischen privaten Login erneut. Bewahre den Hetzner-Konsolenzugang als Wiederherstellungsweg auf.

## Sicherheitscheck

| Kontrolle | Sollzustand |
|---|---|
| Hetzner Firewall | Am VPS zugewiesen, null eingehende Regeln nach Cutover |
| NetBird | Nur `video-admin → video-vps`, TCP 22; keine breite überlappende Policy |
| OpenSSH | `PermitRootLogin no`, keine Passwortanmeldung, nur `ops` mit Schlüssel |
| Zugriff | Frischer privater SSH-Login gelingt; öffentliche IPv4/IPv6 scheitert bei einem tatsächlich routbaren Test |
| Neustart | NetBird und SSH starten; frischer privater Login gelingt erneut |

Wenn der private Login scheitert, **lasse die öffentliche Bootstrap-Regel vorerst aktiv** und prüfe Peer-Status, Gruppen, Policy, DNS und `sshd` über die bestehende Verbindung. Nach dem Cutover kannst du über die Hetzner-Konsole eine eng auf deine aktuelle Quelladresse begrenzte SSH-Regel für die Reparatur **temporär** wiederherstellen.

## Einordnung der Hosting-Wahl

Ein lokaler Rechner ist zum Testen einfacher und verursacht keine VPS-Rechnung. Für Zugriff unterwegs muss er aber eingeschaltet und erreichbar sein; bei einem Heimserver kommen Strom, Anschluss und eigener Betrieb hinzu. Ein VPS trennt den Dienst von persönlichen Dateien und bleibt bei ausgeschaltetem Laptop erreichbar, kostet dafür laufend Geld und braucht Updates und Backups. **NetBird funktioniert auch mit einem Heimserver und mit anderen VPS-Anbietern.**

Hostinger bietet ebenfalls VPS und Firewallfunktionen. Seine Angebots- und Verlängerungspreise hängen von der Laufzeit ab; die Pläne werden laut Anbieter im Voraus bezahlt. Hetzner passt hier zu einem einzelnen, stundenweise berechneten Lernserver. Das ist eine **Betriebsentscheidung**, kein Leistungs- oder Sicherheitstest zwischen Anbietern. Prüfe vor jeder Bestellung aktuelle Preise und Konditionen.

## Quellen und Geltungsbereich

Diese Links dokumentieren das Verhalten; Versionsstände und Preise können sich ändern:

- [NetBird: Funktionsweise](https://docs.netbird.io/about-netbird/how-netbird-works), [Zugriffsregeln](https://docs.netbird.io/manage/access-control/manage-network-access), [Client installieren](https://docs.netbird.io/get-started/install)
- [Hetzner: Cloud Firewall](https://docs.hetzner.com/cloud/firewalls/faq/), [Preisanpassung Juni 2026](https://docs.hetzner.com/de/general/infrastructure-and-availability/price-adjustment/)
- [Hostinger: deutsche VPS-Angebote und Verlängerungspreise](https://www.hostinger.com/de/vps)
- [WireGuard: Protokoll und Design](https://www.wireguard.com/)

Dieses Repo enthält bewusst weder echte IPs, Peer-Namen und Schlüssel noch einen automatischen Installer. Die Schritte müssen mit **deinen** Konten und den sichtbaren Prüfergebnissen abgeglichen werden.
