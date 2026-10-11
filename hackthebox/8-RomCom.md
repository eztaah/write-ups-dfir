# RomCom

**Difficulté :** Very Easy · **Artefacts :** image `.vhdx` (`$MFT` et `$Extend\$J`)

Susan a eu plein d'erreurs en extrayant un document depuis une archive, et Defender a levé une alerte. WinRAR est exploité en ce moment par des groupes comme RomCom.

Je monte l'image sous Linux :
```bash
sudo qemu-nbd --connect=/dev/nbd0 image.vhdx
sudo mount -o ro /dev/nbd0p1 /mnt/romcom
```
Puis je parse la MFT et le USN Journal avant de les ouvrir dans Timeline Explorer :
```powershell
MFTECmd.exe -f "$MFT" --csv out --csvf mft.csv
MFTECmd.exe -f "$J"   --csv out --csvf usn.csv
```

## Questions

### 1. What is the CVE assigned to the WinRAR vulnerability exploited by the RomCom threat group in 2025?
**Réponse :** `CVE-2025-8088`

### 2. What is the nature of this vulnerability?
D'après la NVD.

**Réponse :** `Path Traversal`

### 3. What is the name of the archive file under Susan's documents folder that exploits the vulnerability upon opening the archive file?
Dans la MFT, je filtre `Extension` = `.rar` et `Parent Path` sur `Susan\Documents`.

**Réponse :** `Pathology-Department-Research-Records.rar`

### 4. When was the archive file created on the disk?
Dans le USN Journal, ligne `FileCreate` du `.rar`.

**Réponse :** `2025-09-02 08:13:50`

### 5. When was the archive file opened?
Quand on ouvre un fichier, Windows crée un `.lnk` : je regarde quand celui de l'archive a été créé.

**Réponse :** `2025-09-02 08:14:04`

### 6. What is the name of the decoy document extracted from the archive file, meant to appear legitimate and distract the user?
Dans la MFT, juste à côté de l'archive, il y a un PDF.

**Réponse :** `Genotyping_Results_B57_Positive.pdf`

### 7. What is the name and path of the actual backdoor executable dropped by the archive file?
Dans le USN Journal, je filtre sur `FileCreate` : un `.exe` est créé en même temps que le PDF. Je récupère le chemin complet dans la MFT.

**Réponse :** `C:\Users\Susan\AppData\Local\ApbxHelper.exe`

### 8. The exploit also drops a file to facilitate the persistence and execution of the backdoor. What is the path and name of this file?
Même méthode : un `.lnk` est créé au même moment dans le dossier Startup.

**Réponse :** `C:\Users\Susan\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\Display Settings.lnk`

### 9. What is the associated MITRE Technique ID discussed in the previous question?
Boot or Logon Autostart Execution → Shortcut Modification.

**Réponse :** `T1547.009`

### 10. When was the decoy document opened by the end user, thinking it to be a legitimate document?
Création du `.lnk` du PDF.

**Réponse :** `2025-09-02 08:15:05`

## Ce que j'ai retenu
- La MFT donne les chemins, le USN Journal donne l'historique (créations, suppressions…) : il faut utiliser les deux ensemble.
- Un `.lnk` créé = le fichier a été ouvert par l'utilisateur.
