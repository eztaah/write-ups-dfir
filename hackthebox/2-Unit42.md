# Unit42

**Sherlock :** https://app.hackthebox.com/sherlocks/Unit42

**Difficulté :** Very Easy · **Artefacts :** `Microsoft-Windows-Sysmon-Operational.evtx`

Lab basé sur une campagne analysée par Unit42 (Palo Alto) : une version backdoorée d'UltraVNC. On a seulement les logs Sysmon du poste infecté, que j'ai ouverts dans Event Viewer.

Les Event IDs Sysmon utiles ici : 1 (processus créé), 2 (date de création modifiée), 3 (connexion réseau), 5 (processus terminé), 11 (fichier créé), 22 (requête DNS).

## Questions

#### 1. How many Event logs are there with Event ID 11?

*Filter Current Log* → Event ID `11`.

**Réponse :** `56`

#### 2. What is the malicious process which infected the victim's system?

En filtrant sur l'ID 1, il n'y a que 6 événements. L'un d'eux sort du lot : un `.exe.exe` lancé depuis `Downloads`. Le hash est détecté comme un RAT UltraVNC sur VirusTotal.

**Réponse :** `C:\Users\CyberJunkie\Downloads\Preventivo24.02.14.exe.exe`

#### 3. Which Cloud drive was used to distribute the malware?

En filtrant sur les IDs 22 et 11, on voit une requête DNS vers Dropbox juste avant que Firefox crée le fichier.

**Réponse :** `dropbox`

#### 4. The initial malicious file time stomped many files it created on disk. What was the timestamp changed to for a PDF file?

Filtre sur l'ID 2, on cherche le fichier `.pdf`.

**Réponse :** `2024-01-14 08:10:06`

#### 5. The malicious file dropped a few files on disk. Where was "once.cmd" created on disk?

Filtre sur l'ID 11 puis *Find* `once.cmd`. Il y a deux résultats, on garde celui créé par Preventivo.

**Réponse :** `C:\Users\CyberJunkie\AppData\Roaming\Photo and Fax Vn\Photo and vn 1.1.2\install\F97891C\WindowsVolume\Games\once.cmd`

#### 6. The malicious file attempted to reach a dummy domain, most likely to check the internet connection status. What domain name did it try to connect to?

ID 22 avec `Image` = Preventivo.

**Réponse :** `www.example.com`

#### 7. Which IP address did the malicious process try to reach out to?

ID 3.

**Réponse :** `93.184.216.34`

#### 8. The malicious process terminated itself after infecting the PC with a backdoored variant of UltraVNC. When did the process terminate itself?

ID 5.

**Réponse :** `2024-02-14 03:41:58`

## Note
- Avec Sysmon on peut suivre tout le parcours du malware : téléchargement (22 + 11), exécution (1), réseau (3), fin (5).
- L'heure affichée dans Event Viewer est en heure locale : la vraie valeur UTC est dans `UtcTime`.
