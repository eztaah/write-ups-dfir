# Reaper

**Sherlock :** https://app.hackthebox.com/sherlocks/Reaper

**Difficulté :** Very Easy · **Artefacts :** `ntlmrelay.pcapng`, `Security.evtx`

Le SIEM a remonté un logon où l'IP source ne correspond pas au nom de la machine. Avec la capture réseau et les logs, il faut confirmer une attaque NTLM relay.

## Questions

#### 1. What is the IP Address for Forela-Wkstn001?

Filtre Wireshark `nbns`.

**Réponse :** `172.17.79.129`

#### 2. What is the IP Address for Forela-Wkstn002?

Même filtre.

**Réponse :** `172.17.79.136`

#### 3. What is the username of the account whose hash was stolen by attacker?

Avec le filtre `ntlmssp`, on voit une IP interne sans nom d'hôte qui participe à l'authentification d'un utilisateur.

**Réponse :** `arthur.kyle`

#### 4. What is the IP Address of Unknown Device used by the attacker to intercept credentials?

C'est cette IP sans nom d'hôte.

**Réponse :** `172.17.79.135`

#### 5. What was the fileshare path which victim tried navigating to?

Filtre `smb2`, puis je cherche `BAD_NETWORK_NAME`. L'utilisateur s'est d'abord trompé de chemin, ce qui a déclenché l'attaque, puis il a tapé le bon.

**Réponse :** `\\DC01\Trip`

#### 6. What is the source port used to logon to target workstation using the compromised account?

Dans `Security.evtx`, je filtre l'ID **4624** et je cherche un logon de type 3 avec Security ID `NULL SID`, `NtLmSsp` et `NTLM`.

**Réponse :** `40252`

#### 7. What is the Logon ID for the malicious session?

Même événement.

**Réponse :** `0x64A799`

#### 8. What is the workstation name and the source IP Address from which the malicious logon occur?

Le log indique `FORELA-WKSTN002`, mais avec l'IP de l'attaquant, alors que ce poste est en `.136`. C'est cette incohérence qui a déclenché l'alerte.

**Réponse :** `FORELA-WKSTN002, 172.17.79.135`

#### 9. At what UTC time did the the malicious logon happen?

Onglet *Details* → `SystemTime`.

**Réponse :** `2024-07-31 04:55:16`

#### 10. What is the share Name accessed as part of the authentication process by the malicious tool used by the attacker?

ID **5140**, que je relie au même Logon ID.

**Réponse :** `\\*\IPC$`

## Note
- Pour vérifier un logon, je compare le nom de la machine à son IP réelle (trouvée avec NBNS).
- Le Logon ID permet de relier tous les événements d'une même session.
