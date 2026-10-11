# BFT

**Sherlock :** https://app.hackthebox.com/sherlocks/BFT

**Difficulté :** Very Easy · **Artefacts :** `$MFT`

Simon Stark a téléchargé un ZIP depuis un lien reçu par mail le 13 février. On a seulement la MFT pour comprendre ce qui s'est passé.

Je convertis la MFT en CSV puis je l'ouvre dans Timeline Explorer :
```powershell
MFTECmd.exe -f "C:\...\$MFT" --csv "C:\out" --csvf mft.csv
```

## Questions

#### 1. Simon Stark was targeted by attackers on February 13. He downloaded a ZIP file from a link received in an email. What was the name of the ZIP file he downloaded from the link?

Filtre `Extension` = `.zip` et `Created0x10` = 13/02/2024. J'ignore `KAPE.zip`, c'est la collecte. `invoices.zip` est à l'intérieur de l'autre archive.

**Réponse :** `Stage-20240213T093324Z-001.zip`

#### 2. Examine the Zone Identifier contents for the initially downloaded ZIP file. This field reveals the HostUrl from where the file was downloaded, serving as a valuable Indicator of Compromise (IOC) for our investigation/analysis. What is the full Host URL from where this ZIP file was downloaded?

Quand un fichier est téléchargé avec un navigateur, Windows ajoute un ADS `Zone.Identifier` qui contient l'URL. Je filtre le nom de fichier sur `stage` et je regarde la ligne `Zone.Identifier`.

**Réponse :** `https://storage.googleapis.com/drive-bulk-export-anonymous/20240213T093324.039Z/4133399871716478688/a40aecd0-1cf3-4f88-b55a-e188d5c1c04f/1/c277a8b4-afa9-4d34-b8ca-e1eb5e5f983c?authuser`

#### 3. What is the full path and name of the malicious file that executed malicious code and connected to a C2 server?

Je filtre `Parent Path` sur `stage` et je trouve un `.bat`.

**Réponse :** `C:\Users\simon.stark\Downloads\Stage-20240213T093324Z-001\Stage\invoice\invoices\invoice.bat`

#### 4. Analyze the $Created0x30 timestamp for the previously identified file. When was this file created on disk?

**Réponse :** `2024-02-13 16:38:39`

#### 5. Finding the hex offset of an MFT record is beneficial in many investigative scenarios. Find the hex offset of the stager file from Question 3.

Chaque entrée de la MFT fait 1024 octets, donc offset = numéro d'entrée × 1024 :
```
23436 × 1024 = 23998464 = 0x16E3000
```

**Réponse :** `16E3000`

#### 6. Each MFT record is 1024 bytes in size. If a file on disk has smaller size than 1024 bytes, they can be stored directly on MFT File itself. These are called MFT Resident files. Find the contents of the malicious stager identified in Question 3 and answer with the C2 IP and port.

Le `.bat` est assez petit pour être stocké directement dans la MFT. J'ouvre la MFT dans HxD, *Go To* `16E3000`, et je lis le script PowerShell.

**Réponse :** `43.204.110.203:6666`

## Note
- L'ADS `Zone.Identifier` donne l'URL d'où vient un fichier téléchargé.
- Les petits fichiers (moins de ~900 octets) sont stockés dans la MFT : on peut lire leur contenu même sans le disque.
