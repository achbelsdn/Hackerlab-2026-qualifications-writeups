# Ghost Authority

**Catégorie** : FORENSICS | **Points** : 200 | **Auteur du challenge** : Non renseigné | **Statut** : ✅ Q1–Q5 résolues

Mission en 5 questions autour d'une autorité de certification fantôme (**Borealis Certification Authority**) utilisée pour fabriquer des preuves contre Varelia. Identifiants portail : Q1 `Ppuri Eomneun Ja` (misc), Q2 `Il Mandante` (crypto), Q3 `Kajiba` (forensics), Q4 `Anderswo` (osint), Q5 `Nishi` (misc).

## Q1 — Ppuri Eomneun Ja : identifier l'autorité

Croisement des certificats et des logs CT : toutes les pistes convergent vers la même émettrice.

```
HLB2026{Borealis Certification Authority}
```

## Q2 — Il Mandante : identifier le commanditaire

Analyse des attributs des certificats forgés (O=) : l'organisation derrière la flotte de clés.

```
HLB2026{VARELCORP}
```

## Q3 — Kajiba (forensics, 200 pts) — Batch-GCD sur l'export CT

Énoncé : « Toute la flotte de cette autorité dort dans un gros export CT… remonte jusqu'à un blob chiffré ramassé sur la flotte, la marque de l'atelier qui les a tous forgés. »

Fichiers : `crtsh-dump.pem` (1019 certificats CT), `blob-capture-c2node.enc` (256 octets = 1 bloc RSA-2048, haute entropie).

1. Filtrer `issuer == Borealis Certification Authority` → **19 certificats** de la flotte `*.c2.varelcorp.var`, O=VARELCORP (17 clés 2048 bits, 2 à 2047).
2. **Batch-GCD pairwise** sur les 19 modules RSA :
   - `edge-fra07` & `mq-relay2` : gcd = module entier (clé dupliquée, inutile) ;
   - `bastion3` & `ingest-eu` : **facteur premier commun 1024 bits** → génération de clés défectueuse.
3. Factorisation : `p = gcd`, `q = n/p`, `d = e⁻¹ mod φ` → clés privées de `bastion3` et `ingest-eu`.
4. Déchiffrement du blob avec la clé d'`ingest-eu`, padding **OAEP-SHA256** → texte clair propre.

```
HLB2026{keyforge-3.2-build7731}
```

(`keyforge-3.2-build7731` = la marque de l'atelier, l'outil de keygen vulnérable.)

## Q4 — Anderswo (osint, 100 pts) — alibi de Mateus Varga

Énoncé : « Où était Mateus Varga quand les certificats ont été fabriqués ? Donnez la ville et le jour. »

Artefacts : photos EXIF (Samsung S23 : 2026-03-13 18:40:11, GPS 41.2042 N / 8.8507 W ; iPhone 14 Pro : 2026-03-14 09:12:23, GPS proche), `out-of-office.eml` (absence 09/03→22/03/2026, « Assembleia Lusófona de Cibersegurança (ALC 2026) em Porto Aurin »), `programme-confsec.html` (conférence 13–15 mars, session Varga le samedi 14 mars 09:00, Auditório A), posts du profil (« 13 mar : Cheguei a Porto Aurin »).

Croisement avec la date de fabrication : le certificat de signature VARELCORP/Varga (pcap frame 4130, CN=varelcorp.var) a un `notBefore = 2026-03-14 07:10:00 UTC` — le jour même où Varga était à la conférence.

```
HLB2026{Porto Aurin, 14 mars 2026}
```

## Q5 — Nishi (misc, 200 pts) — antidatage du dossier

Énoncé : « Le dossier qui accable Varelia s'attribue une date antérieure à l'incident. À partir de quand le dossier a-t-il réellement pu exister ? » (format `HLB2026{JJ mois AAAA}`)

Artefacts : `dossier-varelcorp.pdf` (métadonnées Create/Modify = 2026-02-20, date **revendiquée**, antidatée), `dossier-varelcorp.pdf.p7s` (signature CMS détachée ; certificat de signature `O=Comite de Divulgacao Independente, CN=dossier.leaks.borealis-trust.example`, émis par Borealis CA, **notBefore = 2026-03-16 08:41:00 GMT**, pas d'attribut signingTime), `ct-log-hote.csv` (cert du host loggé avec not_before 2026-03-16), `wayback-cdx.txt` (1ʳᵉ capture 2026-03-18 11:45).

Raisonnement : **une signature ne peut pas précéder l'existence de son certificat**. Le dossier signé ne peut pas exister avant le notBefore du certificat de signature, corroboré par le log CT (16/03) et Wayback (18/03). La date revendiquée (20 février) est un antidatage.

```
HLB2026{16 mars 2026}
```

## Artefacts

- `~/ctf/challenges/hackerlab2026/kajiba/` — notes + `scripts/solve_kajiba.py`.
- `~/ctf/challenges/hackerlab2026/anderswo/notes/notes.md`, `nishi/notes/notes.md` + artefacts EXIF/CDX.
- `~/ctf/kajiba/`, `~/ctf/il_mandante/`, `~/ctf/ppuri_eomneun_ja/` — répertoires de travail.

## Lessons learned

- Le Batch-GCD pairwise reste la réponse standard à une flotte de clés RSA : chercher d'abord les clés dupliquées (gcd = n), puis les facteurs partagés.
- Un certificat ne peut pas signer avant son `notBefore` : les logs CT et les archives web bornent la date réelle d'un document signé.
- Les métadonnées d'un PDF se falsifient ; la chaîne cryptographique (certificat, CT log) ne se falsifie pas (facilement).
