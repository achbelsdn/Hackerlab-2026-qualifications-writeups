# Inception

**Catégorie** : BOOT2ROOT | **Points** : 1940 | **Auteur du challenge** : [W1z4rd](https://github.com/W1z4rd) | **Statut** : ✅ Q1, Q2, Q8, Q9 · 🔶 Q3–Q7 non résolues

## Description

Après avoir identifié l'IP du serveur C2 du groupe APT Lynx Varelien, pénétrer les couches les plus profondes de son infrastructure : un relais de collecte (SSH:2222 + Loot API:8080), le C2 interne, puis le domaine `CORP.C2`.

## Cibles

- Instance : `172.28.8.2` (SSH relay `:2222`, HTTP Loot API `:8080`) — instances successives observées : `172.28.4.2`, `172.28.11.2`, `172.28.22.2`.
- Réseau interne : `10.96.11.0/24` (relais), C2 MongoDB `10.96.23.2`, DC `10.96.16.2` (SMB `:445`).

## Q1 — Le Rêve Lucide (200 pts) — ✅

La Loot API `/api/drop` accepte un `filename` non filtré : injection de chemin `../../home/collector/.ssh/authorized_keys` pour déposer une clé SSH (paire conservée dans `~/ctf/artifacts/inception-172.28.4.2-q1-key*`) et obtenir le shell `collector`.

```
HLB2026{m1_path_traversal_ssh_key_injection_kissme}
```

## Q2 — L'Extracteur (70 pts) — ✅

`collector` est UID 1000, mais `/opt/loot-processor/.mcp.json` est `root:root` mode **666**. `/usr/local/sbin/entrypoint.sh` relance `processor.py` en root toutes les 60 secondes : hijack de la configuration MCP (payload `artifacts/root_mcp.json`) → exécution root périodique.

```
HLB2026{m1_root_mcp_config_hijack_pmsla}
```

## Q3–Q7 — C2 interne — 🔶 non résolues

Une **NoSQL injection** sur le C2 `10.96.23.2` permet de dumper les collections MongoDB (payload `dump_users.py`) : profils `jsmith` (reset token) et `admin` (données de `toma`). Les questions Gingembre (500), The Projection (200), Mr. Charles (100), The Architect (70) et Kick (200) n'ont pas été bouclées avant la fin — l'instance de relais est devenue instable (SSH `MaxStartups`/fail2ban, `/api/drop` en 500) en milieu d'épreuve.

## Q8 — The Totem (100 pts) — ✅

Le fichier `toma.txt` contient un hash AS-REP Kerberos :

```text
$krb5asrep$17$toma@CORP.C2:...
```

Le compte est `toma` (AS-REP Roasting). Cassage du hash → mot de passe du totem :

```
HLB2026{T0ma#Str0ngP@ss&2Sntz}
```

## Q9 — Limbo (500 pts) — ✅

Le réseau interne découvert depuis le relais contient le DC `10.96.16.2`. Le partage SMB `killswitch` est visible en anonyme mais son ouverture renvoie `STATUS_ACCESS_DENIED`. L'authentification `toma` / mot de passe Q8 ouvre le partage :

```python
from impacket.smbconnection import SMBConnection
smb = SMBConnection(host, host, sess_port=445, timeout=8)
smb.login("toma", "T0ma#Str0ngP@ss&2Sntz")
smb.getFile("killswitch", "killswitch.flag", data.extend)
```

```
HLB2026{d57520b0642aaf967541b959fb7338c1}
```

## Artefacts

- `~/ctf/challenges/hackerlab2026/inception/` — `writeup.md`, notes, artefacts (`internal_scan.txt`, `q3_dump.txt`).
- `~/ctf/challenges/hackerlab2026/ctf-inception/` — notes Q1/Q2, `toma_asrep17.txt`.
- `~/ctf/artifacts/inception-172.28.4.2-q1-key*` — clé SSH déposée.

## Lessons learned

- `/api/drop` + `filename` non filtré = écriture de fichier arbitraire → dépôt de clé SSH.
- Un fichier de config monde-lisible (`666`) consommé par un service root périodique = privesc trivial.
- Un hash `$krb5asrep$17$` identifie immédiatement le compte cible et l'attaque (AS-REP Roasting).
- Les instances éphémères meurent en milieu d'épreuve : faire le offline (cassage de hash, analyse d'artefacts) dès que possible.
