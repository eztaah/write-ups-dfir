# Campfire-1

**Difficulté :** Very Easy · **Artefacts :** Security.evtx du DC, PowerShell-Operational.evtx et Prefetch du poste

Un utilisateur a remarqué des fichiers bizarres sur son PC, on suspecte un Kerberoasting. Il faut le confirmer avec les logs du DC, puis remonter jusqu'au poste.

## Questions

### 1. Analyzing Domain Controller Security Logs, can you confirm the date & time when the kerberoasting activity occurred?
Sur le DC, je filtre l'Event ID **4769** (demande de ticket de service) et je cherche un événement avec :
- `Ticket Encryption Type` = `0x17` (RC4)
- un service qui n'est ni `krbtgt` ni un compte machine (`...$`)

**Réponse :** `2024-05-21 03:18:09`

### 2. What is the Service Name that was targeted?
Dans le même événement.

**Réponse :** `MSSQLService`

### 3. It is really important to identify the Workstation from which this activity occurred. What is the IP Address of the workstation?
Champ `Client Address`.

**Réponse :** `172.17.79.129`

### 4. What is the name of the file used to Enumerate Active directory objects and possibly find Kerberoastable accounts in the network?
Dans les logs PowerShell du poste, je filtre l'ID **4104** (Script Block Logging). Deux minutes avant l'événement du DC, on voit un bypass de l'ExecutionPolicy puis le contenu de PowerView.

**Réponse :** `powerview.ps1`

### 5. When was this script executed?
Premier événement 4104 (en UTC, l'Event Viewer affiche l'heure locale).

**Réponse :** `2024-05-21 03:16:32`

### 6. What is the full path of the tool used to perform the actual kerberoasting attack?
Je parse le Prefetch :
```powershell
PECmd.exe -d "C:\...\prefetch" --csv . --csvf prefetch.csv
```
Dans Timeline Explorer, je filtre sur la date et je garde les programmes lancés pour la première fois (`Previous Run` vide). On trouve `RUBEUS.EXE`, et le chemin complet est dans `Files Loaded`.

**Réponse :** `C:\Users\Alonzo.spire\Downloads\Rubeus.exe`

### 7. When was the tool executed to dump credentials?
Colonne `Last Run`, une seconde avant l'événement sur le DC.

**Réponse :** `2024-05-21 03:18:08`

## Ce que j'ai retenu
- Kerberoasting = 4769 avec RC4 (`0x17`) sur un compte de service.
- Le Prefetch prouve qu'un programme a été lancé, et donne quand et depuis où.
