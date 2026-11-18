# Overwrite

**Catégorie** : WEB | **Points** : 300 | **Auteur du challenge** : [5c0r7](https://github.com/5c0r7) | **Statut** : 🔶 Chaîne identifiée, non conclue

## Description

Application de rapports (Flask, `web:8000`) avec un moteur de rendu interne (`render:9000`, « spectre-report-render ») qui évalue des templates Jinja2 sous `/srv/attach`. Compte fourni : `a.rhone` / `analyst2026` (rôle Analyste, UID 1). Objectif : récupérer la clé opérateur (`OPK-…`) pour déverrouiller le moteur via SSRF et exécuter un template SSTI lisant le flag.

## Recon / Enumeration

- Session Flask signée (`{"uid":1}`) : brute-force de la clé secrète échoué (rockyou 14M, wordlists Flask) — piste abandonnée.
- Mass assignment et HTTP override (`PUT`/`PATCH`, `X-HTTP-Method-Override`) : rejetés.
- **Cartographie SSRF** via `POST /generator/fetch` : découverte de `http://render:9000/` ; `GET /render` → `clé opérateur requise` ; `?key=test` → `clé opérateur invalide` (paramètre confirmé). Le tasking n°4 confirme que `render:9000` rend des templates Jinja2 sous `/srv/attach` (`template_path`).
- Payload SSTI prêt : `{{ cycler.__init__.__globals__.os.popen('cat /flag* …').read() }}`.

## Exploitation — RPO (Relative Path Overwrite) vers la clé opérateur

La clé opérateur (`data-k`) n'est injectée dans le DOM (`<span id="sig" data-k="OPK-…">`) que pour une session opérateur ; pour les autres elle est masquée (`OPK-xxxxxxxxxxxxxxxx`).

Vecteur identifié :

- Les pages d'erreur (403/404) incluent leur CSS avec un **chemin relatif** : `<link rel="stylesheet" href="console.css">`, sans `X-Content-Type-Options: nosniff`.
- L'upload accepte les `.css`, déposés dans `/srv/attach/a.rhone/` et servis sur `/attach/a.rhone/<filename>`.
- Un CSS malveillant cible `#sig[data-k^=…]` / `#sig[data-k$=…]` pour exfiltrer la clé caractère par caractère vers un listener.

**Preuve d'exécution** : le bot relecteur (HeadlessChrome/151) a bien chargé le CSS et émis les requêtes de fond — logs du listener :

```text
[14:19:27] GET /s16-x        from 172.28.4.2  UA=HeadlessChrome/151
[14:24:04] GET /MASKED-exact from 172.28.4.2  UA=HeadlessChrome/151
```

## Point de blocage

Le bot matchait la règle `#sig[data-k="OPK-xxxxxxxxxxxxxxxx"]` : la clé restait **masquée** dans son contexte. Causes analysées :

1. Le bot navigue avec sa session sur `web:8000` (console interne) ; sur `172.28.4.2` c'est une autre origine, ses cookies ne partent pas → session non authentifiée → clé masquée.
2. Le contrôle d'accès de `/attach/<username>/...` compare le `<username>` de l'URL à la session : le bot (opérateur) vu via l'URL `a.rhone` est un tiers → clé masquée.
3. `/config` (page qui expose la clé aux opérateurs) renvoie 403 pour un analyste.

Pistes restées ouvertes : faire viser au bot une `source_url` sur `http://web:8000/attach/…` (résolution interne), tester le RPO depuis le chemin de l'opérateur, ou faire charger `/config` par le bot. Le challenge n'a pas été conclu avant la fin de l'épreuve.

Plan d'exécution prévu dès le premier caractère fuité : SSRF `POST /generator/fetch` avec `url=http://render:9000/render?key=OPK_<leaked>&template_path=a.rhone/payload.txt` → lecture du flag.

## Artefacts

- `~/ctf/knowledge/writeups/writeup-overwrite-partial.md`, `~/ctf/knowledge/overwrite-172.28.14.2.md`.
- `~/ctf/challenges/hackerlab2026/overwrite/` — 70+ scripts d'exploration (`bypass_*.py`, `test_rpo*`, `gen_css.py`, `listener.py`), notes, `REDEPLOY.md`.

## Flag

Non obtenu.

## Lessons learned

- Le RPO est une attaque **client-side** : elle ne vaut que si la victime partage l'origine de la page vulnérable — vérifier les cookies/origines du bot avant d'itérer.
- Un masquage applicatif (`OPK-xxxx…`) peut être cohérent côté serveur : la fuite dépend du contexte de session de la victime, pas du payload.
- Documenter les faux positifs : une « RCE confirmée » peut n'être qu'une expansion locale `$(...)` par le shell d'attaque (double-quoting).
