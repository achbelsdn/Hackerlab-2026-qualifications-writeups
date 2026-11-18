# Hold The Front Page

**Catégorie** : WEB | **Points** : 300 | **Auteur du challenge** : Non renseigné | **Statut** : ✅ Résolu

## Description

Empêcher la publication automatique de l'article « Varelia démasquée : les pièces d'une cyber-offensive commanditée ». Deux services : site public (`:8000`) et Studio rédactionnel (`:8001`). Compte fourni : `pigiste_invite` / `pigiste2026` (rôle pigiste, volontairement limité).

## Recon / Enumeration

Le dashboard ne montre pas l'article cible : il est programmé dans un contexte éditorial inaccessible au rôle de pigiste. Routes utiles :

- `/dashboard.php`, `/edit.php`, `/draft.php?id=...`, `/submit.php` (soumission à la relecture), `/media.php` (téléversement), `/feed.php?id=...` (métadonnées JSON), `/schedule.php` (gestion de publication).
- Jeton CSRF présent dans chaque page : `<meta name="csrf-token" content="...">`.

## Exploitation

Le rôle de pigiste ne peut pas appeler directement l'action de publication : la chaîne passe par le **bot de relecture**, qui ouvre les brouillons soumis pendant la fenêtre temporelle active (compte à rebours).

1. **Upload SVG actif** — `media.php` accepte un fichier `.svg` servi directement sous `/media/`, sans `Content-Disposition` protecteur : le JavaScript du SVG s'exécute si le fichier est ouvert comme document.
2. **Iframe dans un brouillon** — le SVG est chargé dans une iframe injectée dans un brouillon (`edit.php`).
3. **Soumission à la relecture** — `submit.php` pendant la fenêtre active : le bot ouvre le brouillon et exécute le JS dans le contexte du Studio.
4. **Payload** — `fetch('/dashboard.php').text()` puis extraction dynamique du jeton CSRF par regex (`csrf-token`) ; appel POST à `/compte.php` pour élever le contexte éditorial (`display=Roland Solari…`) ; extraction des IDs d'articles cibles ; appel à `schedule.php` avec l'action **`hold`** pour annuler la publication.

> Respecter l'état attendu par l'application : l'action correcte est `hold` (pas `cancel`), et le bot n'évalue le payload que pendant le compte à rebours.

## Artefacts

- `~/ctf/knowledge/writeups/writeup-hold-the-front-page.md` — writeup complet.
- `~/vigie-evil.svg` — payload SVG XSS/CSRF (réutilisé pour Vigie).

## Flag

Dynamique par instance (non reproduit ici).

## Lessons learned

- Un SVG servi même-origine sans `Content-Disposition: attachment` permet l'exécution JavaScript dans le contexte de l'application.
- `fetch(...).text()` + regex pour extraire dynamiquement un CSRF est très fiable pour chaîner XSS → CSRF.
- Un bot de relecture = oracle d'exécution : lui faire faire les actions que le rôle interdit.
