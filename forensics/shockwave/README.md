# Shockwave

**Catégorie** : FORENSICS | **Points** : 1550 | **Auteur du challenge** : [Sh1n](https://github.com/Sh1n) | **Statut** : 🔶 Accès root + implant identifiés (mission `tervagrid_1`, 13 questions)

## Description

Audit de routine des données télémétriques du réseau SCADA de TervaGrid : une séquence anormale apparaît « sur les ondes ». Déterminer ce qui se passe réellement sur le réseau et remonter la piste avant le black-out.

## Cible

- `scada-gw-01` — `172.28.12.2:20000` (service DNP3 `dnp3d`).

## Accès root — PWN sur le service DNP3

`dnp3d` est un ELF x86-64 **non-PIE**. Le service lit jusqu'à `0x258` octets dans un buffer heap puis traite une trame commençant par `05 64` ; la longueur (octet 2) est utilisée **sans contrôle** dans un `memcpy` vers un buffer de pile de 80 octets.

**Fuite du canari** : une trame commençant par `05 65` provoque `write(fd, rbp-0x50, 0x50)`, ce qui expose le canari de pile aux offsets 72..79.

**Exploitation** : la trame `05 64` recopie 96 octets —

```text
72 octets de remplissage | canari fuité | 8 octets rbp | adresse de retour 0x401296
```

`0x401296` enchaîne `setuid(0)`, `setgid(0)`, `execl("/bin/sh", …)` :

```text
[dnp3d] engineering console
uid=0(root) gid=0(root) groups=0(root)
```

Script : `~/ctf/.local_downloads/Others/probe_dnp3.py` (fuite du canari + trame de débordement).

## Implant « Live Wire » (Q2, PWN, 200 pts) — ✅

Une fois root sur `scada-gw-01`, la persistance est identifiée :

- `/opt/gw/.tervasync` — script shell lancé en arrière-plan par `/opt/gw/entrypoint.sh` : archive périodiquement `/opt/gw/staging`, encode en base64, découpe en blocs de 48 octets et injecte les blocs dans `/opt/gw/.radio/inject.txt` sous forme de trames `EXF`.
- `/opt/gw/radio-tx.sh` — ajoute `inject.txt` à `frames.txt`, modulé en **AFSK** et diffusé par la bande radio via `broadcast.sh` et le serveur TCP `:40000`.
- Le beacon `.radio/.beacon` et la boucle infinie maintiennent l'implant actif.

Preuves observées : processus `/bin/sh /opt/gw/.tervasync`, artefacts `inject.txt`, `frames.txt`, `.beacon`, clé `/opt/gw/staging/master_control.key`.

C'est l'exfiltration radio (AFSK) qui explique l'anomalie « sur les ondes » de l'énoncé.

```
HLB2026{2eb26db800f863c148af10c3f3217a6c}
```

## Reste de la mission — 🔶

Les 11 autres questions (`Background Noise`, `Sleeper`, `StrongBox`, `Knock Knock`, `Original Sin`, `The Ferryman`, `Moonwalking`, `Smooth Criminal`, `Deadman`, `Cold Storage`, `Pathogen`, `NightShift`) n'ont pas été documentées dans les notes archivées.

## Lessons learned

- Un service binaire non-PIE avec un `memcpy` de longueur non contrôlée : chercher d'abord une primitive de **fuite** (ici une trame de debug qui dump la pile) avant le débordement.
- Le canari de pile ne bloque rien si on peut le lire puis le réécrire à l'identique dans la même chaîne.
- Sur une passerelle SCADA, la persistance est souvent un simple script shell + canal exotique (radio/AFSK) : inspecter `entrypoint.sh`, les fichiers dotfiles et les processus `/bin/sh` en arrière-plan.
