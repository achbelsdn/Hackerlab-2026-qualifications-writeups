# 403 Forbidden News

**Catégorie** : WEB | **Points** : 200 | **Auteur du challenge** : mak0ra | **Statut** : ✅ Résolu

## Description

Le frontoffice du journal *La Sentinelle* (WordPress) publie une « édition spéciale » programmée. Objectif : rendre cette édition indisponible publiquement. Le backoffice est une API Uvicorn derrière HAProxy (HTTPS:8000), et un plugin WordPress « TC Backoffice Guard » bloque la corbeille tant que l'article est référencé dans le backoffice.

## Recon / Enumeration

- Frontoffice WordPress sur `:80`, backoffice API sur `:8000` (annonce du panneau), port **8001 non annoncé** = origine HTTP du backoffice, utile pour comparer proxy vs origine.
- Le formulaire public de candidature accepte un dépôt de fichier et le sert tel quel.

## Exploitation

### 1. XSS stockée par SVG actif

Un SVG peut contenir du JavaScript et, servi comme document (`image/svg+xml`, même origine, sans `Content-Disposition: attachment`), il s'exécute. On soumet au formulaire public un SVG dont le script :

1. fetch `/wp-admin/` avec `credentials: include` (le bot de relecture a une session WordPress privilégiée) ;
2. extrait le nonce REST par regex sur `wpApiSettings` / `createNonceMiddleware` ;
3. appelle `POST /wp-json/wp/v2/users/<id>` pour **réinitialiser le mot de passe de `e.vasseur`** (compte admin) vers une valeur choisie par le payload ;
4. trace chaque étape via `new Image().src` vers un listener.

### 2. Session administrateur

Connexion en `e.vasseur` avec le mot de passe imposé par le payload. On identifie le post « édition spéciale », son `_backoffice_uuid` (via export WXR) et le réglage du garde :

```
tc_backoffice_article_path=https://backoffice:8000/api/articles
```

### 3. Faiblesse logique du garde

Le garde décide d'après le **code HTTP seul** : `200` → corbeille refusée, `404` → corbeille autorisée. Il ne vérifie ni l'UUID du JSON, ni la cohérence du statut, ni la source de la réponse.

### 4. Désynchronisation HTTP CL.0 + empoisonnement du cache HAProxy

Le backend Uvicorn traite un `GET` comme une requête de longueur nulle : un `GET` externe avec un `Content-Length: N` voit son corps **reparsé comme une deuxième requête** par le backend, tandis que HAProxy le considère comme du corps. Sur une seule connexion TLS HTTP/1.1 (SNI `backoffice`, keep-alive, `Connection: close` sur la requête cible) :

```
vue HAProxy                       vue Uvicorn
GET /desync + corps      --->     GET /desync
                                  GET /api/articles/<absent>
                                  GET /api/articles/<cible>
GET /api/articles/<cible> --->
```

La réponse `200` de la cible est **décalée** sur la requête `<absent>`, laquelle renvoie donc un `404` que HAProxy met en cache (les 404 sont cachables, TTL 30 s). Le garde reçoit alors un faux 404, autorise la corbeille, et l'URL publique passe en 404. Le flag est lu dans `wp-admin/options.php`, champ `_tc_flag`.

> Validation non destructive d'abord : la primitive a été testée sur un objet bénin avant de viser l'édition spéciale.

## Artefacts

- `~/ctf/challenges/hackerlab2026/403-forbidden-news/403 FN.md` — writeup complet reproductible (1454 lignes).
- `~/scripts/poison.py` — désynchronisation CL.0 + cache poisoning TLS/HTTP1.1.
- `~/scripts/dynamic_admin_password_reset.svg`, `submit_reset_payload.py` — payload XSS et soumission.

## Flag

```
HLB2026{a15d87b3a02b883b3835178a833795ed}
```

## Lessons learned

- Comparer origine directe vs proxy (port 8001 vs 8000) révèle les divergences de parsing.
- HTTP/2 ne désynchronise pas ici : rester en HTTP/1.1 brut, une seule connexion, `Content-Length` exact.
- Attendre l'expiration d'un 200 frais avant de re-empoisonner.
- Un contrôle d'accès qui se fie au code HTTP seul se contourne par empoisonnement de cache.
