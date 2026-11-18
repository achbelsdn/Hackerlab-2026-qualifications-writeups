# Brainiac

**Catégorie** : BRAINFUCK / OSINT & FORENSIC | **Points** : 400 | **Auteur du challenge** : [W1z4rd](https://github.com/W1z4rd) | **Statut** : 🔶 Chaîne amont résolue, contact bot bloqué

## Description

`l3_r3lais` a mis sur pied une plateforme de diffusion de sa campagne de désinformation, gérée par un bot Telegram. Objectif : entrer en contact avec ce bot, récupérer le lien vers la plateforme et saisir celle-ci.

## Recon / Enumeration

Le bot visé est `t.me/TrvPunisher_bot`. Son deep-link d'activation attend un payload `?start=`.

## Exploitation — la chaîne amont (Bohemian Rhapsody)

Le payload du bot n'est pas devinable : il provient de la mission **The Great Pretender / diplomatie Q6** :

1. **Charon** (moteur onion, via Tor SOCKS 9150) → forum **The Masquerade** + compte `hellynxthebuilder`.
2. **LeakZone** — thread `showthread.php?tid=11` : table users du forum en clair → hash MD5 cracké avec rockyou (`ph4nt0m`).
3. **Login Masquerade** → DM privé de `l3_r3lais` → clé de campagne `HLB2026{78cd8c43a91f54cea5ee8c4f4c2e35ba}`.

Ce flag, **sans accolades** (les accolades sont refusées dans un deep-link Telegram), forme le lien d'activation :

```
https://t.me/TrvPunisher_bot?start=78cd8c43a91f54cea5ee8c4f4c2e35ba
```

## Blocage — 🔶

- `/start` simple → « recruitment phase is over » : le flux start-payload exige un **compte Telegram frais non restreint** ; le compte de session était gelé à ce moment.
- Récit vérifié par recoupement : le thread Charon `/t/92` relie `l3_r3lais`, `hellynxthebuilder`, `dark_ech0`, `pal3_daem0n`, `phantom_ciph3r13` (« Relay has it. Don't ask in the open again. »).
- La plateforme et les dumps sociaux ont été archivés (`brainiac-ownsocial-*.json`, `brainiac-telegram-*.html`) mais la suite du flux n'a pas pu être validée avant la fin de l'épreuve.

Aucun flag n'est revendiqué pour ce challenge.

## Artefacts

- `~/ctf/artifacts/brainiac-*` — pages bot Telegram, dumps sociaux.
- `~/ctf/artifacts/masquerade-*`, `leakzone-*`, `charon-*` — chaîne amont.
- `~/scripts/brainiac-telegram-extract.py`, `brainiac-ownsocial-*.py`.

## Lessons learned

- Les deep-links Telegram refusent certains caractères : tester le payload sans accolades.
- Un challenge « hub » (bot) dépend parfois d'un flag produit par une autre mission : résoudre les missions connexes d'abord.
- Prévoir le blocage infrastructure (compte gelé) : documenter la chaîne et l'état exact plutôt que deviner.
