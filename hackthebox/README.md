# HackTheBox Sherlocks

Mes writeups des Sherlocks HackTheBox (labs DFIR) que j'ai faits pour m'entraîner à l'investigation numérique et à la réponse à incident.

| Lab | Sujet |
|---|---|
| [Brutus](1-Brutus.md) | Brute-force SSH et persistance sous Linux (`auth.log`, `wtmp`) |
| [Unit42](2-Unit42.md) | Malware UltraVNC analysé avec Sysmon |
| [Campfire-1](3-Campfire.md) | Kerberoasting (logs du DC, PowerShell, Prefetch) |
| [Reaper](4-Reaper.md) | NTLM relay (Wireshark + logs Windows) |
| [Campfire-2](5-Campfire2.md) | AS-REP Roasting |
| [BFT](6-BFT.md) | Analyse de la MFT |
| [MangoBleed](7-MangoBleed.md) | Exploitation de MongoDB (CVE-2025-14847) sous Linux |
| [RomCom](8-RomCom.md) | Exploitation de WinRAR (CVE-2025-8088), MFT et USN Journal |

**Outils utilisés :** Event Viewer, Wireshark, Eric Zimmerman tools (MFTECmd, PECmd, Timeline Explorer), HxD, commandes Linux (`grep`, `awk`…)
