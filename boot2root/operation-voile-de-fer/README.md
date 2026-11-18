# Operation Voile de Fer

**Catégorie** : BOOT2ROOT | **Points** : 970 | **Auteur du challenge** : Non renseigné | **Statut** : ✅ Résolu (chaîne multi-questions)

## Description

Opération de contre-espionnage sur l'infrastructure FNS (Front Norveth de Sécurité) : à partir d'un dump de fuite publique, remonter jusqu'au contrôleur de domaine du domaine `fns.nv` et prouver la compromission à chaque étage (ws01, SRV-JUMP01, lnx-app01, DC01).

## Cartographie

| Élément | Adresse / port | Rôle |
|---|---|---|
| Portail webmail | `172.28.13.2:443` | Point d'entrée FNS |
| SSH frontal | `172.28.13.2:22` | Premier saut SSH |
| `front01` interne | `10.96.14.4:22,25,143,443,993` | Relais / messagerie |
| `ws01` | `10.96.14.6:22` | Poste de traitement |
| `SRV-JUMP01` | `10.96.14.3:22,1433` | MSSQL et serveur de rebond |
| `lnx-app01` | `10.96.14.5:22` | Hôte de la preuve |
| `DC01` | `10.96.14.2:22,53,88,389,445,636` | Contrôleur de domaine |

Réseau interne `10.96.14.0/24`, routé depuis le VPN.

## Exploitation

### 1. Compte initial depuis le dump

Le lien de l'énoncé mène à une archive contenant `nexus_breach_2026.csv` (dump « RisePro »). L'entrée **active** de `j.durant@fns.nv` (pas les anciennes rotations de mot de passe) ouvre le webmail et le SSH frontal.

### 2. Root sur ws01 — PwnKit / GConv

`/usr/bin/pkexec` est SUID root sur `ws01`. On compile un module GConv qui lit `/root/proof.txt`, `/root/mission.txt` et `/etc/hosts`, un lanceur PwnKit (`GCONV_PATH=.`, `CHARSET=WS01`), on les téléverse et on exécute :

```text
EUID 0  →  lecture de /root/proof.txt sur ws01
```

### 3. Pivot MSSQL vers SRV-JUMP01

Depuis `ws01`, le port `1433` accepte l'authentification du domaine. Via `impacket.tds.MSSQL` à travers le tunnel SSH, on active `xp_cmdshell` et on inventorie : un interpréteur **`python3-report` root-owned avec `cap_setuid=ep`**, protégé par un wrapper AppArmor qui bloque les noms connus (`python3`, `python3-report`…).

**Contournement** : le compte SQL contrôle `/opt/mssql/data` ; un lien symbolique court (`pr`) vers l'interpréteur échappe au filtre de noms et exécute `os.setuid(0)` puis la lecture de `/root/.secrets/ssh.conf` et `id_ecdsa` — la clé ECDSA de `m.blanc@fns.nv`.

### 4. lnx-app01 — sudo tar comme primitive de lecture

La clé ECDSA ouvre `lnx-app01` (via double tunnel `front01 → ws01 → lnx-app01`). Le compte appartient au groupe `LNX-APP01 Admins` :

```text
m.blanc@fns.nv may run: (root) NOPASSWD: /usr/bin/tar
```

Lecture sans shell root :

```bash
sudo -n /usr/bin/tar -cf - /root/proof.txt | /usr/bin/tar -xOf - root/proof.txt
```

### 5. DC01 — LDAP proxy et su administrator

Les creds `svc_backup` extraits de la chaîne permettent d'accéder à `dc01` ; `/opt/setup/ldap-proxy.py` contient `ADMIN_DN`/`ADMIN_PW`, puis `su - administrator` donne la preuve finale.

## Artefacts

- `~/ctf/knowledge/writeups/writeup-voile-fer-mission.md` (+ `-04.md`, `-05.md`) — writeups détaillés.
- `~/scripts/voile-fer-ws.py`, `voile-fer-mssql.py`, `voile-fer-mblanc-ssh.py` — pivots paramiko (1 script par saut).
- `ws01-gconv.c`, `ws01-pkexec.c` — PwnKit/GConv prêt à compiler.

## Flags

```text
ws01        : HLB2026{D1QZn5XlRaeqoCHkZIAKmwfsWg5mFPPOeeQnUR3r3G6likwnBw2fvWGJslyMVuuO}
SRV-JUMP01  : HLB2026{moIPEwCHvBgLATEgPNqdpSr85WSKq7qOdXEaNQRtKHFqHCxCnE5BfCPYLh5uVVJB}
lnx-app01   : HLB2026{j44hHVIuQ0CPwtwo30NoEFKxkfmJMEFxZ3ZLuwNMqIRAKA36Oi70d0SYCdXWRCnT}
mission Q01 (DC01) : HLB2026{d1e555807530d3a33397c809029c618d}
```

> Les flags « proof » d'une machine ne sont pas forcément les flags des questions de mission : vérifier chaque question séparément.

## Lessons learned

- Pivot SSH : un script Python par saut, `open_channel("direct-tcpip")` chaînés, secrets jamais codés en dur.
- Énumération root systématique : `getcap -r /`, SUID, `/root/.secrets`.
- Un binaire capabilisé reste exploitable via un **chemin d'accès alternatif** (symlink) quand un filtre de nom bloque.
- `sudo tar` = primitive de lecture de fichier arbitraire.
- Les dumps de fuite contiennent des rotations de mots de passe : seule l'entrée « active » fonctionne.
