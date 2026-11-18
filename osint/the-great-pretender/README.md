# The Great Pretender

**Catégorie** : OSINT / AI | **Points** : 500 | **Auteur du challenge** : Non renseigné | **Statut** : ✅ Q1–Q4 résolues · Q5 bloquée · Q6 ✅

Mission `diplomatie` du portail : 6 questions — Toxic, Legion, Ymir, Vestigium, Griot, Bohemian Rhapsody.

## Contexte

Une campagne de désinformation ciblant Varélia est diffusée sur une instance Mastodon (`ownsocial.duckdns.org`, version 4.7.0) dont **l'API est publique sans authentification**. Le même écosystème héberge un chat IA (« Griot ») qui génère la campagne.

## Q1–Q4 — Campagne Mastodon (résolues)

Méthode : tout passer par l'API publique (`/api/v1/tags`, timelines paginées, `/api/v1/admin/accounts`).

1. **Hashtag de campagne dominant** — `/api/v1/tags` + historique hebdomadaire : le tag `VareliaMenace` domine avec un pic anormal d'usage → `HLB2026{VareliaMenace}`.
2. **Nombre de comptes relais** — pagination **complète** du feed du tag (1 278 posts), déduplication : **82 comptes**, tous `bot=true` → `HLB2026{82}`.
3. **Patient zéro** — tri par `created_at` : `patriote1er` (2026-08-19T19:10:37Z) est le premier compte du réseau → `HLB2026{patriote1er}`.
4. **IP source commune** — `robots.txt` de l'instance exposait `/backups/ops_notify.sh.bak` contenant un **bearer token admin**. Ce token débloque `/api/v1/admin/accounts`, qui renvoie les champs `ip`/`ips` : les bots partagent la même IP source `198.51.100.77` → `HLB2026{198.51.100.77}`.

## Q5 — Griot (AI/ML, 100 pts) — 🔶 non résolue

Cible : UI de chat `POST /chat` JSON `{message:...}`. Refus de campagne sur toute demande directe de secret. La **répétition** a leaké la première phrase du prompt système (`Tu es Griot, le moteur de generation de la campagne.`) et une extraction **positionnelle** (par numéros de ligne) a produit 26 hex — mais le candidat `HLB2026{...}` correspondant a été rejeté par la plateforme.

Pistes restantes documentées : autres gadgets d'extraction (traduction, résumé, encodage base64/hex du système, faux marqueurs de fin de conversation), vérification du format réel du flag. Harness : `~/scripts/griot_probe.py`.

## Q6 — Bohemian Rhapsody (OSINT, 200 pts) — ✅ résolue

Chaîne onion, via Tor SOCKS 9150 (`curl --socks5-hostname 127.0.0.1:9150`) :

1. **Charon** (moteur de recherche onion) — recherche « Varelia » → forum **The Masquerade** + compte `hellynxthebuilder`.
2. **LeakZone** — thread `showthread.php?tid=11` : table `users` du forum Masquerade en clair, hash MD5 `434acf…b8c0`.
3. **Cassage** — `john --format=Raw-MD5` + rockyou → `ph4nt0m` en quelques secondes.
4. **Login Masquerade** → DM privé `/messages/l3_r3lais` (compte ciblé trouvé par recoupement des threads `/t/92`) → clé de campagne.

Flag : `HLB2026{78cd8c43a91f54cea5ee8c4f4c2e35ba}` — sert ensuite de payload `?start=` du bot Telegram Brainiac (sans accolades : les accolades sont refusées dans un deep-link).

## Artefacts

- `~/ctf/artifacts/great-pretender-{evidence,stage02,stage04}.json` — preuves API.
- `~/ctf/artifacts/masquerade-*`, `leakzone-*`, `charon-*` — chaîne onion.
- `~/scripts/great-pretender-*.py`, `masquerade-live-*.py`.

## Lessons learned

- L'API Mastodon publique (tags, timelines, instances) suffit à toute la phase de cartographie — pas de compte nécessaire.
- Paginer **avant** de compter ; dédupliquer les comptes.
- `robots.txt` et fichiers `.bak` restent des pépites : un token admin dans un script de backup donne accès aux données privées (IP des comptes).
- Les dumps de forum exposent souvent les tables users ; MD5 brut = rockyou en secondes.
