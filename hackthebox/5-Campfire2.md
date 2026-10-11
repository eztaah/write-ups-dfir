# Campfire-2

**Sherlock :** https://app.hackthebox.com/sherlocks/Campfire-2

**Difficulté :** Very Easy · **Artefacts :** `Security.evtx` du DC

Un vieux compte admin qui n'est plus utilisé a demandé un ticket au DC. On suspecte un AS-REP Roasting : l'attaque vise les comptes qui n'ont pas la pré-authentification Kerberos.

## Questions

#### 1. When did the ASREP Roasting attack occur, and when did the attacker request the Kerberos ticket for the vulnerable user?

Je filtre l'ID **4768** (demande de TGT) et je cherche un événement avec :
- `Pre-Authentication Type` = `0`
- `Ticket Encryption Type` = `0x17` (RC4)
- `Service Name` = `krbtgt`

Une recherche sur `0x17` suffit à le trouver. L'heure UTC est dans l'onglet *Details*.

**Réponse :** `2024-05-29 06:36:40`

#### 2. Please confirm the User Account that was targeted by the attacker.

**Réponse :** `arthur.kyle`

#### 3. What was the SID of the account?

**Réponse :** `S-1-5-21-3239415629-1862073780-2394361899-1601`

#### 4. It is crucial to identify the compromised user account and the workstation responsible for this attack. Please list the internal IP address of the compromised asset to assist our threat-hunting team.

**Réponse :** `172.17.79.129`

#### 5. We do not have any artifacts from the source machine yet. Using the same DC Security logs, can you confirm the user account used to perform the ASREP Roasting attack so we can contain the compromised account/s?

`arthur.kyle` est la victime. Pour trouver le compte de l'attaquant, je cherche l'IP source dans les logs : l'événement 4769 juste après montre quel compte est utilisé sur cette machine.

**Réponse :** `happy.grunwald`

## Note
- AS-REP Roasting = 4768 avec pré-auth à 0 et RC4. Kerberoasting = 4769.
- Ne pas confondre le compte victime et le compte utilisé par l'attaquant.
