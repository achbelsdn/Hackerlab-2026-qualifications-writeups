# Vantage

**Catégorie** : WEB | **Points** : 200 | **Auteur du challenge** : Non renseigné | **Statut** : ✅ Résolu

## Description

Portail web d'une société écran utilisée par VARELCORP. Objectif : déface la page d'accueil (`slug='home'`) du CMS interne, ce qui exige d'obtenir une exécution de code sur le serveur PostgreSQL backend.

## Recon / Enumeration

La page d'accueil recèle des **commentaires HTML** décrivant l'architecture : un cache edge frontal, une application PHP (`account.php`, `admin/review.php`), un PostgreSQL 16 en backend et un flux de relecture par un compte interne. Ce triage préalable évite tout fuzzing inutile.

## Exploitation

La chaîne complète comporte cinq maillons :

### 1. Cache edge par extension — fuite du jeton d'intake

Le cache est indexé **par extension**, pas par chemin complet : une requête vers
`/account.php/bootstrap-<nonce>.css` est servie depuis le cache alors que le backend génère le HTML de `account.php`. En récupérant la réponse mise en cache, on obtient le HTML administrateur — dont le jeton d'intake — sans jamais s'authentifier.

> Leçon : établir la clé réelle du cache (Host, chemin, extension, query, cookies) avant toute attaque ; ici l'extension seule déterminait la clé.

### 2. XSS reviewer via CDN autorisé par la CSP

La CSP autorise `cdn.jsdelivr.net`. On charge donc **Angular 1.8.2** depuis le CDN et on utilise `ng-include` pour faire fetcher `/admin/review.php?id=2` par la victime (le reviewer), puis extraire `<meta name="vantage-key">` et l'exfiltrer vers un écouteur sur le VPN.

> Une CSP permissive vers un CDN est un gadget XSS prêt à l'emploi : le framework permet de lire des pages même-origine sans écrire une ligne de JS custom.

### 3. SQLi UNION sur l'API admin

`/admin/search.php?format=json&api_key=...` concatène le paramètre de recherche dans une requête PostgreSQL. Une injection **UNION** (3 colonnes) donne une lecture arbitraire de la base.

### 4. PostgreSQL large objects = canal de dépôt de fichiers

Via la SQLi, on utilise `lo_from_bytea` / `lo_put` / `lo_export` pour écrire un objet binaire (un `.so`) sur le disque du serveur, envoyé en blocs hexadécimaux.

### 5. RCE par `session_preload_libraries`

Le module C exporte `Pg_magic_func` avec l'ABI exacte attendue par PostgreSQL, plus `_PG_init`. Chargé via `session_preload_libraries`, il s'exécute avec les droits du backend au prochain démarrage de session. On remplace alors le contenu du `slug='home'` par la page `seized.html`.

## Artefacts

- `~/ctf/knowledge/writeups/writeup-vantage.md` — writeup complet (431 lignes).
- `~/ctf/artifacts/seized.html` — page de déface.
- archive locale `vantage-writeup.zip` — source `vantage_preload.c` (module C).

## Flag

```
HLB2026{6467cd00113667c399da7e84be83afac}
```

## Lessons learned

- Lire les commentaires HTML d'abord ; ils décrivent souvent toute l'architecture.
- Le cache suit l'extension, pas le chemin : tester des paires chemin/extension croisées.
- Une CSP qui autorise un CDN = gadget XSS gratuit (Angular `ng-include`).
- Une SQLi PostgreSQL est une chaîne RCE complète : large objects pour déposer, `pg_read_binary_file` pour lire, `session_preload_libraries` pour exécuter.
- Un module `.so` PostgreSQL doit exporter `Pg_magic_func` avec l'ABI exacte, sinon le backend refuse de démarrer.
