# Twisted - HackMyVM (Easy)
 
![Twisted.png](Twisted.png)

## Übersicht

*   **VM:** Twisted
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=Twisted)
*   **Schwierigkeit:** Easy
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 9. Oktober 2022
*   **Original-Writeup:** https://alientec1908.github.io/Twisted_HackMyVM_Easy/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel der "Twisted"-Challenge war die Erlangung von User- und Root-Rechten. Der Weg begann mit Steganographie: Aus einer Bilddatei `cat-original.jpg` (deren Ursprung im Log nicht dokumentiert ist, vermutlich vom Webserver auf Port 80) wurde mittels `stegseek` und dem Passwort `westlife` eine Datei `markus.txt` extrahiert. Diese enthielt das Passwort `markuslovesbonita` für den Benutzer `markus`. Dies ermöglichte den SSH-Login auf dem nicht standardmäßigen Port 2222 als `markus`. Als `markus` wurde in einer `note.txt` ein Hinweis auf den Speicherort des privaten SSH-Schlüssels (`/var/cache/apt/id_rsa`) für den Benutzer `bonita` gefunden. Mit diesem Schlüssel gelang der SSH-Login als `bonita`. Die User-Flag wurde in deren Home-Verzeichnis gefunden. Die Privilegieneskalation zu Root erfolgte durch Ausnutzung eines SUID/SGID-Root-Binaries `/home/bonita/beroot`. Die Analyse dieses Binaries (impliziert durch Ghidra) ergab, dass die Eingabe des dezimalen Werts von `0x16f8` (also `5880`) eine Root-Shell startete.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `stegseek`
*   `cat`
*   `ssh`
*   `ls`
*   `find`
*   `tail`
*   `vi` (oder anderer Texteditor)
*   `chmod`
*   `python3 http.server`
*   `wget`
*   `ghidra` (oder anderes Reverse Engineering Tool)
*   `apt` (zur Installation von Ghidra)
*   Standard Linux-Befehle (`cd`, `id`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Twisted" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Initial Credential Discovery:**
    *   IP-Findung mit `arp-scan` (`192.168.2.125`).
    *   `nmap`-Scan identifizierte offene Ports: 80 (HTTP - Nginx 1.14.2) und 2222 (SSH - OpenSSH 7.9p1).
    *   *Der Fundort der Datei `cat-original.jpg` ist im Log nicht dokumentiert (vermutlich vom Webserver auf Port 80).*
    *   `stegseek cat-original.jpg /usr/share/wordlists/rockyou.txt` fand das Passwort `westlife` und extrahierte die Datei `markus.txt` (als `cat-original.jpg.out`).
    *   `cat cat-original.jpg.out` zeigte das Passwort `markuslovesbonita`.

2.  **Initial Access (SSH als `markus`):**
    *   Erfolgreicher SSH-Login als `markus` mit dem Passwort `markuslovesbonita` auf Port 2222 (`ssh markus@twisted.hmv -p 2222`).

3.  **Privilege Escalation (von `markus` zu `bonita`):**
    *   `sudo -l` für `markus` war nicht verfügbar/erfolgreich.
    *   `cat note.txt` im Home-Verzeichnis von `markus` enthielt den Hinweis: "Hi bonita, I have saved your id_rsa here: /var/cache/apt/id_rsa".
    *   Auslesen des privaten SSH-Schlüssels von `bonita` aus `/var/cache/apt/id_rsa`.
    *   Der Schlüssel wurde lokal gespeichert (z.B. `id`), Berechtigungen auf `600` gesetzt.
    *   Erfolgreicher SSH-Login als `bonita` mit dem extrahierten Schlüssel auf Port 2222 (`ssh -i id bonita@twisted.hmv -p 2222`).
    *   User-Flag `HMVblackcat` in `/home/bonita/user.txt` gelesen.

4.  **Privilege Escalation (von `bonita` zu `root` via SUID Binary `beroot`):**
    *   `sudo -l` für `bonita` war nicht verfügbar/erfolgreich.
    *   `find / -type f -perm -4000 ...` identifizierte ein SUID-Root und SGID-`bonita` Binary `/home/bonita/beroot` (`-rwsrws--- 1 root bonita`).
    *   Das Binary `beroot` wurde vom Zielsystem auf die Angreifer-Maschine heruntergeladen (via Python HTTP-Server) und mit Ghidra analysiert.
    *   Die Analyse ergab, dass das Programm bei Eingabe des Hex-Wertes `0x16f8` (dezimal `5880`) eine Root-Shell (`setuid(0); setgid(0); system("/bin/bash");`) startet.
    *   Ausführung von `/home/bonita/beroot` als `bonita` und Eingabe von `5880`.
    *   Erlangung einer Root-Shell.
    *   Root-Flag `HMVwhereismycat` in `/root/root.txt` gelesen.

## Wichtige Schwachstellen und Konzepte

*   **Steganographie (stegseek):** Passwort wurde in einer Bilddatei versteckt und mit einer Wortliste gefunden.
*   **Informationsleck (Passwort in Datei):** Ein weiteres Passwort (`markuslovesbonita`) wurde in der extrahierten Textdatei gefunden.
*   **Informationsleck (Pfad zu privatem SSH-Schlüssel in Notiz):** Eine Notiz im Home-Verzeichnis eines Benutzers verriet den Speicherort des SSH-Schlüssels eines anderen Benutzers.
*   **Unsichere Speicherung privater SSH-Schlüssel:** Ein privater SSH-Schlüssel war für andere Benutzer lesbar.
*   **Benutzerdefiniertes SUID/SGID-Binary mit Logikfehler (`beroot`):** Ein SUID-Root-Programm enthielt einen Mechanismus (Eingabe eines festen Codes), der zur direkten Erlangung einer Root-Shell missbraucht werden konnte.
*   **Reverse Engineering (Ghidra):** Analyse eines Binaries zur Aufdeckung der internen Logik und des Exploits.
*   **SSH auf nicht standardmäßigem Port:** Erschwerte die Entdeckung geringfügig.

## Flags

*   **User Flag (`/home/bonita/user.txt`):** `HMVblackcat`
*   **Root Flag (`/root/root.txt`):** `HMVwhereismycat`

## Tags

`HackMyVM`, `Twisted`, `Easy`, `Steganography`, `stegseek`, `SSH`, `SUID Exploitation`, `Reverse Engineering`, `Ghidra`, `Privilege Escalation`, `Linux`
