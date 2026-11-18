# No Story Left Behind

**Catégorie** : MISC | **Points** : 500 | **Auteur du challenge** : [Johans585](https://github.com/Johans585) | **Statut** : 🔶 Chaîne de preuve complète, flag final non obtenu

## Description

Le gouvernement de Tervalis attend une révélation, Varélia attend son démenti, et une rédaction s'apprête à fabriquer un coupable. Challenge unique : **The Last Edit** (500 pts, portail `lastedit`) — cible `http://172.28.3.2:8080` (Tervalis Herald, Flask/Werkzeug), article embargoé `/article/dossier-varga`.

## Exploitation — la chaîne de preuve

1. **EXIF** de `/static/img/dossier-varga.jpg` : commentaire `#OPSEC -- LOdT` (L'Œil de Tervalis).
2. **Stéganographie** — `steghide` avec la passphrase `oeil-de-tervalis23873` (dérivée du compte/indice) → message : `Nom de code : sherlock`.
3. **GitHub** — compte `oeil-de-tervalis23873` : le commit `787970b` **supprime** `img/photo.jpg`.
4. **Blob supprimé** — récupération du blob orphelin `e43067a` (`git cat-file` après `git fsck`), même passphrase steghide → `git-note.txt` : l'article Varga est une **fabrication**.

## Blocage — 🔶

La route protégée `/newsroom/admin` (login) ou une révision authentifiée contient le flag final. Les variantes de login `sherlock` ont échoué.

Pistes documentées : wordlist de conventions (`~/scripts/no-story-internal-words.txt`), sessions/révisions, réutiliser les conventions de mots de passe du challenge « Vigie ». Arpanges/ffuf du site archivés (`~/ctf/artifacts/no-story-13-*`, `ffuf-no-story-*`).

Aucun flag n'est revendiqué.

## Artefacts

- `~/ctf/artifacts/no-story-current/` — dumps de l'instance.
- `~/ctf/artifacts/git-note.txt`, `no-story-13-*` — preuves de la chaîne.
- `~/scripts/no-story-flask-cookie.py`, `forge-admin-cookie.py`, `extract_git_steg.py`, `no-story-hex-crypto.py`.

## Lessons learned

- Un commit qui **supprime** un fichier laisse un blob récupérable : `git fsck` + `git cat-file` avant de conclure.
- Les indices OPSEC (commentaire EXIF) donnent souvent la passphrase de stégano du même fichier.
- Quand une passphrase fonctionne sur deux artefacts, elle est probablement une convention réutilisable ailleurs (autres challenges, autre route).
