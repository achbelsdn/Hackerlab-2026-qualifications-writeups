# Shutter Island

**Catégorie** : OSINT / WEB / DFIR | **Points** : 2029 | **Auteur du challenge** : [W1z4rd](https://github.com/W1z4rd) | **Statut** : 🔶 Partiel (mission `empreinte`, 16 questions)

## Description

Fuite de données chez AquaTerv, notamment des données sensibles du réseau urbain de la ville d'Averis. Identifier les données divulguées, le mode d'exfiltration, et l'ensemble des éléments permettant de comprendre l'origine et l'ampleur de la fuite.

Questions portail observées : The prestige (179), Genesis, Under The Root (500), Unseen (300), The Man Who Wasn't There, Is falling down my fair lady, Mr Sam, Prelude, **Dead Drop (200)**, Tamper root, Oh My DNS, AquaWatch is down, ExFil, Conn, Upsurge, Ctrl2.

## Chaîne « exfiltration » — comms chiffrées de l'agent (3 questions résolues)

> Attribution : ces trois questions sont libellées « mission exfiltration » dans les notes locales ; le portail place `Dead Drop` dans cette mission (q1). L'attribution des deux autres est probable mais non confirmée.

### Dead Drop — réutilisation de nonce AES-GCM — ✅

**Vulnérabilité** : « NorvethSecure v3 » = AES-256-GCM avec nonce déterministe (`nonce = HKDF(key, counter)`). La spec impose « Counter state MUST be persisted across reconnections » — mais lors d'une reconnexion (même `session_id`/`master_secret`), le compteur repart de 0 : **réutilisation de nonce** ⇒ keystream AES-CTR identique.

**Données** : 90 frames CBOR `[sid, kid, nonce, ct, tag]`. Session `461f78d8…` = session_01 (sans reconnexion) puis session_02/03 (reconnexion). key_id `0x03` (Priority) : session_01 idx20-26 (compteurs 0-6) et session_03 idx13-19 (compteurs remis à 0) réutilisent les **7 mêmes nonces**. Message connu : session_01 idx20 = HEARTBEAT seq20, pad = 100×'A'.

**Méthode** :
1. Parser CBOR maison, groupement par nonce → 7 réutilisations ;
2. Les heartbeats session_01 idx20-26 suivent un template fixe (seuls seq/timestamp varient, intervalle régulier) ; brute-force d'un intervalle global unique → **120 s** rend toutes les frames session_03 en JSON propre ;
3. `keystream_k = ct_s01 ⊕ heartbeat_k` puis `plaintext_s03 = ct_s03 ⊕ keystream_k`.

Messages récupérés (agent TIMS, canal Priority, jour de l'arrestation 2024-11-03) : ALERT / STATUS (plaque VLD-74-OPS) / DISTRESS (« FBI Valdoria identified, being detained ») / FINAL.

```
HLB2026{TIMS_SESSION_GCM_NONCE_REUSE_CHANNEL_INTEGRITY_ALERT}
```

### Broken Handshake — attribution de certificat — ✅

Protocole d'attribution : le certificat légitime est celui dont l'extension OID `1.3.6.1.4.1.99999.1` … (dérivation complète dans `~/ctf/challenges/hackerlab2026/broken-handshake/notes/notes.md`, script `scripts/solve.py`).

### Ratchet Failure — rollback de Double Ratchet — ✅

**Vulnérabilité** : Double Ratchet (X25519/HKDF/HMAC chain/ChaCha20-Poly1305). Crash + restauration depuis un backup antérieur ⇒ **rollback de la chaîne d'envoi**. Trois snapshots d'état :

- `state_000.cbor.enc` — chiffré par PIN (PBKDF2-HMAC-SHA256, 1000 iters, salt = `SHA256(device_id)[:16]`, AES-256-CBC, IV préfixé), Ns=5 ;
- `state_001.cbor` — Ns=12, CKs corrompu (erreurs de bits aux offsets 4, 17, 28) ;
- `state_002.cbor` — Ns=17, intact (heartbeats 17-24 seulement).

**Méthode** :
1. `salt = SHA256("NVTH-TERM-7742")[:16]` ;
2. brute-force du PIN 6 chiffres en parallèle (12 cœurs) → **PIN = 847291** (validé par padding PKCS7 + CBOR) ;
3. depuis state_000 : `kdf_ck` (MK = HMAC(CK,1), CK = HMAC(CK,2)) pour dériver MK5..MK24 ;
4. déchiffrement ChaCha20-Poly1305 (nonce = N en little-endian 4 octets + 8 zéros, AAD = header CBOR).

Messages récupérés : heartbeats 5-11, STATUS (activité inhabituelle près de l'hôtel), REPORT (SUV noir, plaque VLD-74), ALERT (VLD-74-OPS, plusieurs occupants), **ALERT HIGH (équipe d'interpellation, gear tactique) + FLAG** (msg 15), STATUS « Going dark ».

```
HLB2026{RATCHET_STATE_ROLLBACK_NS12_MK15_COORDINATES_FOUND}
```

Objectif secondaire : `device_dump.raw` contient une section `NVTH_AUDIO_LOG_V3` — extraction d'un WAV RIFF 16 kHz 16-bit mono (8,2 s, ambiance de l'arrestation à 21:43:22Z).

## Reste de la mission — 🔶

Les 13 autres questions n'ont pas été documentées dans les notes archivées.

## Artefacts

- `~/ctf/challenges/hackerlab2026/dead-drop/` — notes + `scripts/extract.py`, `solve2.py`.
- `~/ctf/challenges/hackerlab2026/broken-handshake/`, `ratchet-failure/` — notes + scripts (`pin_par.py`).
- `~/ctf/.local_downloads/Others/toma.txt` — hash AS-REP associé au fil Inception.

## Lessons learned

- Un nonce déterministe + compteur réinitialisé = réutilisation de nonce : le two-time pad rend le XOR trivial dès qu'un template de message connu existe.
- Brute-forcer un **paramètre global** (l'intervalle des heartbeats) plutôt que chaque message individuellement.
- Un rollback d'état de ratchet réutilise les message keys : la « sécurité forward » ne protège pas contre une restauration de backup.
- Valider un PIN par la structure déchiffrée (padding PKCS7 + CBOR valide), pas par la longueur.
