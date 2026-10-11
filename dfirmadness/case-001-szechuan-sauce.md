# The Stolen Szechuan Sauce

**Lab :** https://dfirmadness.com/the-stolen-szechuan-sauce/

**Difficulté :** *I'm Too Young to Die* (mode facile : disque, mémoire, Autoruns et PCAP fournis)

**Artefacts :** images disque E01, dumps mémoire et Autoruns de `CITADEL-DC01` et `DESKTOP-SDN1RPT`, capture réseau `case001.pcap`

**Outils :** Wireshark, Volatility 3, Registry Explorer, EvtxECmd, MFTECmd, Timeline Explorer, Autopsy, secretsdump.py, John the Ripper, CyberChef

## Contexte

Le FBI prévient notre client que sa recette secrète de sauce Szechuan est en vente sur le dark web. La recette était stockée sur un partage de fichiers du contrôleur de domaine. Le but est de savoir s'il y a eu intrusion, comment l'attaquant est entré, ce qu'il a fait et ce qu'il a volé.

Le réseau est petit : un contrôleur de domaine (Windows Server 2012 R2) et un poste utilisateur (Windows 10), dans le domaine `C137.local`.

## Préparation

### Monter les images disque

```bash
sudo ewfmount citadeldc01.E01 /mnt/ewf
sudo mount -o ro,loop,offset=$((718848*512)) /mnt/ewf/ewf1 /mnt/dc01
```
L'offset de la partition Windows s'obtient avec `mmls /mnt/ewf/ewf1`. Sous Windows, FTK Imager ou Autopsy font la même chose.

Ensuite j'extrais les artefacts utiles de chaque machine : ruches du registre (`SYSTEM`, `SOFTWARE`, `SAM`, `NTUSER.DAT`), `Security.evtx` et `System.evtx`, `$MFT`, `$UsnJrnl:$J`, et `ntds.dit` sur le DC.

### Le piège du fuseau horaire

C'est le point le plus important du lab, et beaucoup de writeups se trompent dessus.

- Le client est au Colorado : en septembre, l'heure locale est **UTC-6**.
- Dans le registre (`SYSTEM\ControlSet001\Control\TimeZoneInformation`), les VM sont configurées en **Pacific Standard Time**, donc mal réglées.
- Résultat : les horodatages des machines (logs, MFT, mémoire) ont **1 h d'avance** sur ceux du PCAP. Par exemple, le premier logon RDP réussi est à 03:21:46 dans `Security.evtx` mais à 02:21:47 dans le PCAP.

J'ai pris le PCAP (horloge du routeur, correcte) comme référence. **Toutes les heures de ce writeup sont en UTC, le 19/09/2020, corrigées sur le PCAP.** Pour comparer avec les artefacts des machines, il faut retirer 1 h aux valeurs brutes.

## Timeline de l'attaque

| Heure (UTC) | Machine | Événement |
|---|---|---|
| 02:19:26 | DC01 | Début du brute-force RDP depuis `194.61.24.102` |
| 02:21:47 | DC01 | Brute-force réussi sur `Administrator` |
| 02:22:36 | DC01 | Session RDP interactive de l'attaquant (~30 min) |
| 02:24:06 | DC01 | Téléchargement de `coreupdater.exe` depuis `194.61.24.102` (Internet Explorer) |
| 02:25:18 | DC01 | Première connexion Meterpreter vers le C2 `203.78.103.109` |
| 02:27:49 | DC01 | Installation du service de persistance |
| ~02:30 | DC01 | Fichiers du partage compressés dans `secret.zip` |
| ~02:31 | DC01 | `secret.zip` exfiltré puis supprimé |
| 02:32:21 | DC01 | Accès à `Szechuan Sauce.txt` |
| ~02:34 | DC01 | `Secret_Beth.txt` supprimé, remplacé par `Beth_Secret.txt` |
| 02:35:55 | DC01 → Desktop | Mouvement latéral en RDP |
| ~02:38 | DC01 | Timestomping de `Beth_Secret.txt` |
| 02:39:58 | Desktop | Téléchargement de `coreupdater.exe` (Edge) |
| 02:40:49 | Desktop | Connexion Meterpreter vers le C2 + persistance |
| ~02:46 | Desktop | Création de `loot.zip` |
| ~02:48 | Desktop | `loot.zip` exfiltré puis supprimé |
| 02:56:03 | DC01 | Dernière session RDP de l'attaquant |
| fin du PCAP | DC01 + Desktop | Sessions Meterpreter toujours actives |

## Questions

#### 1. What's the operating system of the server?

Dans la ruche `SOFTWARE`, clé `Microsoft\Windows NT\CurrentVersion`, valeur `ProductName`. On peut aussi le lire depuis la mémoire :
```bash
vol -f citadeldc01.mem windows.info
```

**Réponse :** `Windows Server 2012 R2 Standard Evaluation`

#### 2. What's the operating system of the desktop?

Même clé dans la ruche `SOFTWARE` du poste.

**Réponse :** `Windows 10 Enterprise Evaluation`

#### 3. What was the local time of the server?

Le registre indique `Pacific Standard Time`, mais l'entreprise est au Colorado et les heures des machines ont 1 h d'avance sur le PCAP. Les VM sont donc mal configurées (voir *Préparation*).

**Réponse :** heure du Colorado, `UTC-6` (le serveur est réglé à tort sur Pacific Time)

#### 4. Was there a breach?

Oui : brute-force RDP réussi, malware installé sur les deux machines, données exfiltrées.

**Réponse :** `Oui`

#### 5. What was the initial entry vector?

Dans le PCAP, filtre `tcp.port == 3389` : des centaines de connexions par seconde depuis `194.61.24.102` vers le DC, à partir de 02:19:26. Dans `Security.evtx` du DC :
- une rafale de **4625** (échecs de logon) sur `Administrator` ;
- puis un **4624** réussi en Logon Type 10 (RDP), avec `kali` comme nom de station.

**Réponse :** brute-force RDP sur le compte `Administrator` du DC, depuis `194.61.24.102`, réussi à 02:21:47

#### 6. Was malware used?

**Réponse :** `Oui`, un payload Meterpreter (Metasploit) nommé `coreupdater.exe`

#### 6.1. What process was malicious?

```bash
vol -f citadeldc01.mem windows.pstree
vol -f citadeldc01.mem windows.malfind
```
- `pstree` montre `coreupdater.exe` (PID 3644), dont le parent n'existe plus en mémoire.
- `malfind` montre `spoolsv.exe` (PID 3724) avec une zone mémoire en `PAGE_EXECUTE_READWRITE` qui contient un en-tête `MZ` : c'est de l'injection de code. Le dump de cette zone est détecté comme Meterpreter par ClamAV.

```bash
vol -f citadeldc01.mem -o dump/ windows.malfind --pid 3724 --dump
clamscan dump/
```
Le malware a démarré en `coreupdater.exe` puis a migré dans `spoolsv.exe`, une fonction classique de Meterpreter (`migrate`).

**Réponse :** `coreupdater.exe`, migré ensuite dans `spoolsv.exe`

#### 6.2. What IP address delivered the payload?

Dans Wireshark, *File → Export Objects → HTTP* : `coreupdater.exe` est servi par un serveur HTTP Python sur `194.61.24.102`, la même IP que le brute-force.

**Réponse :** `194.61.24.102`

#### 6.3. What IP address is the malware calling to?

```bash
vol -f citadeldc01.mem windows.netscan
```
`coreupdater.exe` a une connexion `ESTABLISHED` de `10.42.85.10` vers `203.78.103.109:443`. FLOSS sur le dump de `spoolsv.exe` donne la même IP.

**Réponse :** `203.78.103.109`

#### 6.4. Where is the malware on disk?

**Réponse :** `C:\Windows\System32\coreupdater.exe`

#### 6.5. When did it first appear?

Le PCAP montre le téléchargement à 02:24:06. La MFT confirme la création du fichier dans le dossier `Downloads` de l'Administrator juste après (horodatage corrigé de 1 h).

**Réponse :** `2020-09-19 02:24` UTC sur le DC, et `02:39:58` UTC sur le Desktop

#### 6.6. Did someone move it?

Dans la MFT et le `$UsnJrnl:$J`, on voit le fichier créé dans `Downloads` puis renommé/déplacé (`RenameOldName` / `RenameNewName`) vers `System32`, pour passer pour un fichier système.

**Réponse :** `Oui`, de `C:\Users\Administrator\Downloads\` vers `C:\Windows\System32\`

#### 6.7. What were its capabilities?

C'est un Meterpreter : shell à distance, migration de processus, vol d'identifiants (hashdump), keylogger, captures d'écran, upload/download de fichiers, timestomping… Sur un DC, il tournait en `SYSTEM`.

**Réponse :** accès distant complet (Meterpreter)

#### 6.8. Is it easily obtained?

**Réponse :** `Oui`, il se génère en une commande avec `msfvenom` (Metasploit est gratuit et open source)

#### 6.9. Was it installed with persistence on any machine? When and where?

L'export Autoruns, filtré sur les entrées actives sans signature, montre deux mécanismes sur le DC :
- un **service Windows** qui lance `C:\Windows\System32\coreupdater.exe` au démarrage, confirmé par l'Event ID **7045** dans `System.evtx` ;
- une **clé Run** dans `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run`, qui lance `powershell.exe -w hidden` avec du Base64 lu dans `HKLM\SOFTWARE\9sEoCawv`, valeur `45SVAG2o`. Décodé dans CyberChef (*From Base64* puis *Gunzip*), c'est un loader PowerShell qui injecte du shellcode.

Le Desktop a les deux mêmes mécanismes.

**Réponse :** `Oui`, service + clé Run sur le DC (service installé à 02:27:49) et sur le Desktop (~02:41)

#### 7. What malicious IP addresses were involved?

**Réponse :** `194.61.24.102` (brute-force + livraison du payload) et `203.78.103.109` (C2)

#### 7.1. Were any IPs from known adversary infrastructure?

Sur VirusTotal et AbuseIPDB, `194.61.24.102` est signalée pour du brute-force RDP. `203.78.103.109` est géolocalisée en Thaïlande et a été brièvement liée au domaine `happydoghappycat-th.com`, mais ce lien a ensuite été écarté.

**Réponse :** `Oui`, `194.61.24.102` est connue pour du brute-force RDP

#### 7.2. Are these pieces of infrastructure involved in other attacks around the same time?

**Réponse :** `194.61.24.102` est signalée pour d'autres brute-force RDP à la même période. Pour `203.78.103.109`, aucun lien solide.

#### 8. Did the attacker access any other systems?

**Réponse :** `Oui`, le poste `DESKTOP-SDN1RPT`

#### 8.1. How?

Dans le PCAP, filtre `rdp && ip.src == 10.42.85.10` : le DC ouvre une session RDP vers `10.42.85.115`. Le nom d'utilisateur `Administrator` apparaît dans le trafic Kerberos. Dans le `Security.evtx` du Desktop, on retrouve le **4624** correspondant.

**Réponse :** en RDP depuis le DC, avec le compte `Administrator` du domaine

#### 8.2. When?

**Réponse :** `2020-09-19 02:35:55` UTC

#### 8.3. Did they steal or access any data? When?

Dans le `$UsnJrnl:$J` des deux machines, on voit la création puis la suppression d'une archive ZIP. L'exfiltration est passée par la session Meterpreter vers `203.78.103.109` (le contenu n'est pas lisible dans la capture).

**Réponse :** `Oui`, `secret.zip` sur le DC (~02:31) et `loot.zip` sur le Desktop (~02:48)

#### 9. What was the network layout of the victim network?

**Réponse :**

| Machine | IP | Rôle |
|---|---|---|
| `CITADEL-DC01` | `10.42.85.10` | Contrôleur de domaine `C137.local` + partage de fichiers |
| `DESKTOP-SDN1RPT` | `10.42.85.115` | Poste utilisateur |

Réseau `10.42.85.0/24`. Le RDP du DC est directement exposé sur Internet.

#### 10. What architecture changes should be made immediately?

**Réponse :**
- Couper l'accès RDP depuis Internet, et passer par un VPN avec MFA.
- Ne pas stocker de fichiers sensibles sur le contrôleur de domaine.
- Réinitialiser tous les mots de passe du domaine (le DC est compromis, donc `ntds.dit` aussi) et réinstaller les deux machines.
- Bloquer `194.61.24.102` et `203.78.103.109`.

#### 11. Did the attacker steal or access the Szechuan sauce? If so, what time?

Dans la MFT du DC, la date d'accès de `Szechuan Sauce.txt` sur le partage tombe pendant la session de l'attaquant, juste avant la création de `secret.zip`.

**Réponse :** `Oui`, vers `02:30-02:32` UTC (accès au fichier à 02:32:21)

#### 12. Did the attacker steal or access any other sensitive files? If so, what times?

**Réponse :**
- `Secret_Beth.txt` supprimé et remplacé par `Beth_Secret.txt` vers 02:34 ;
- les pensées de Morty volées vers 02:34 ;
- `loot.zip` exfiltré depuis le Desktop vers 02:48.

#### 13. What was the last known contact with the adversary?

La dernière session RDP de l'attaquant commence à 02:56:03, et la dernière déconnexion interactive est vers 03:00. Mais les sessions Meterpreter restent ouvertes jusqu'à la fin de la capture (~05:38 sur les deux machines).

**Réponse :** dernière activité interactive vers `03:00` UTC, mais l'attaquant a encore un accès actif à la fin du PCAP

## Questions bonus

#### Which users have actually logged onto the DC?

Event ID **4624** avec Logon Type 2 ou 10 dans le `Security.evtx` du DC, et profils présents dans `C:\Users`.

**Réponse :** `Administrator`

#### Which users have actually logged onto the Desktop?

**Réponse :** `Administrator` et `ricksanchez`

#### What are the passwords for the domain users?

J'extrais les hash NT de `ntds.dit` avec la ruche `SYSTEM` du DC, puis je les casse avec rockyou :
```bash
secretsdump.py -ntds ntds.dit -system SYSTEM LOCAL -outputfile hashes
john --format=NT --wordlist=rockyou.txt hashes.ntds
```

**Réponse :**

| Compte | Mot de passe |
|---|---|
| Administrator | `:)&Denver89` |
| jerrysmith | `!BETHEYBOO12!` |
| summersmith | `34MarvelBootySalmon$$` |
| ricksanchez | `800PortalsForMe%` |
| mortysmith | `Jessica@1` |
| bethsmith | `RedWine1!` |
| birdman | `(dimension5150)` |

#### Can you recover the original file about Beth's secrets?

Le fichier original a été supprimé mais son entrée existe encore dans la MFT. Comme il fait moins de ~900 octets, son contenu est **résident** : il est stocké directement dans l'enregistrement MFT, donc récupérable avec Autopsy (fichiers supprimés) ou en hexa à l'offset de l'entrée.

**Réponse :** nom d'origine `Secret_Beth.txt`, contenu `Earth Beth is the real Beth.`

#### Which file was time stomped?

Dans MFTECmd, je compare les dates de `$STANDARD_INFORMATION` (0x10) et de `$FILE_NAME` (0x30). Pour `Beth_Secret.txt`, la date de création 0x10 est plus ancienne que la 0x30, ce qui est impossible normalement : Meterpreter (`timestomp`) ne modifie que `$SI`. La date a été copiée sur celle de `PortalGunsPlans.txt` pour que le fichier ait l'air ancien.

**Réponse :** `Beth_Secret.txt`

#### Which controls would have prevented this breach?

**Réponse :**
- ne pas exposer le RDP sur Internet (CIS 12, gestion de l'infrastructure réseau) ;
- MFA et verrouillage de compte après plusieurs échecs (CIS 5 et 6) ;
- des mots de passe forts, `Administrator` en a un faible ;
- de la journalisation centralisée avec une alerte sur les rafales de 4625 (CIS 8).

## Note

- Toujours vérifier le fuseau horaire avant de construire la timeline. Ici, une erreur d'1 h fausse toutes les corrélations entre PCAP et machines.
- Commencer par le PCAP et la mémoire donne des pivots (IP, nom du malware, PID) qui rendent l'analyse disque beaucoup plus rapide.
- `$SI` plus ancien que `$FN` = signe de timestomping.
- Un petit fichier supprimé peut encore être lu depuis la MFT s'il était résident.
