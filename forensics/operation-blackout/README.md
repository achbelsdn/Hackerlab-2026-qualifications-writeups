# Operation Blackout

**Catégorie** : FORENSICS | **Points** : 1000 | **Auteur du challenge** : [Frazer](https://github.com/Frazer) | **Statut** : 🔶 Q1–Q3 résolues sur 12

## Description

Le SOC détecte un trafic sortant anormal depuis le poste d'un agent de règlement : spear-phishing ciblé contre `j.aterbe@bct.gov.trv`, usurpant un contact de VARELCORP. Reconstituer l'attaque complète et l'attribuer. Artefacts : `courriel.eml`, `zone-dns.txt`, `capture.pcap`.

Les 12 questions observées : domaine de l'expéditeur ; IP du serveur d'envoi ; technique anti-forensique des macros ; clé de déchiffrement ; fichier déposé ; empreinte SHA-256 ; URL de téléchargement du second composant ; domaine d'exfiltration ; endpoint C2 ; jeton d'accès exfiltré ; identité du processus usurpé ; mutex.

## Q1 — Domaine de l'expéditeur — ✅

Le champ `From` visible usurpe `varelcorp.var`, mais le domaine **DKIM réellement utilisé** est :

```
d=xn--varelcrp-s4a.var
```

Ce domaine est un **homographe IDN** de VARELCORP (`xn--varelcrp-s4a.var` → `varelcörp.var`). Sa zone DNS pointe vers `203.0.113.40` avec un SPF autorisant `include:_spf.kalnet.kal`.

```
HLB2026{xn--varelcrp-s4a.var}
```

## Q2 — IP du serveur d'envoi — ✅

En-tête `Received` dans `courriel.eml` :

```
Received: from relay-mail.kalnet.kal ([203.0.113.10])
         by mx.bct.gov.trv (Postfix) ...
```

Le relais externe ayant transmis le mail au MX de la BCT est `relay-mail.kalnet.kal`.

```
HLB2026{203.0.113.10}
```

## Q3 — Technique anti-forensique des macros — ✅

La macro vérifie l'environnement avant toute exécution — **détection de sandbox / anti-VM** :

- `NumberOfLogicalProcessors < 2` → arrêt silencieux ;
- `TotalPhysicalMemory < 2147483648` (2 Gio) → arrêt silencieux ;
- contrôle de temporisation (Timer), typique des contres d'analyse.

Réponse : détection d'environnement d'analyse (sandbox evasion) basée sur les caractéristiques matérielles, avec abandon silencieux.

## Q4–Q12 — 🔶 non résolues

La suite (clé de déchiffrement, fichier déposé, SHA-256, second composant, domaine d'exfiltration, endpoint C2, jeton exfiltré, processus usurpé, mutex) exige l'analyse complète de la macro, du dropper et de `capture.pcap` ; elle n'a pas été menée à terme dans les notes archivées.

## Artefacts

- `~/ctf/.local_downloads/Others/courriel.eml`, `zone-dns.txt`, `whois-borealis.txt`.
- `~/ctf/.local_downloads/Others/writeup_spearphishing_sender_domain.txt`, `-sender_ip.txt`, `writeup_macro_antiforensics.txt`.
- `~/ctf/artifacts/capture-poste.pcap`.

## Lessons learned

- Ne jamais se fier au `From` visible : le domaine DKIM/SPF réel révèle l'homographe IDN.
- Un homographe se détecte en décodant le label `xn--` (punycode) et en comparant visuellement.
- Les macros malveillantes sondent CPU/RAM/timer : dans une VM d'analyse modeste, elles sortent silencieusement — augmenter les ressources avant analyse.
