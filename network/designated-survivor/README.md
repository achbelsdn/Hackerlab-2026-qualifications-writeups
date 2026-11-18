# Designated Survivor

**Catégorie** : NETWORK | **Points** : 200 | **Auteur du challenge** : The Chall Makers | **Statut** : ✅ Résolu (question unique `infra`)

## Description

Six affaires, six dossiers instruits séparément par le CERT-Tervalis : une centrale d'eau, une banque, un hôpital, un opérateur électrique, un fournisseur d'eau urbain, un mail de phishing. Le TCDA exige un **rapport provisoire d'incidents** : une synthèse de ce que l'équipe a réellement établi sur ces six dossiers, et une conclusion sur le responsable.

## Rapport provisoire — synthèse des six dossiers

Chaque dossier correspond à une autre mission du portail ; la valeur du challenge est la **corrélation** :

| Dossier | Mission liée | Ce qui a été établi |
|---|---|---|
| Centrale d'eau | Eaux Troubles (infection) | Compromission du poste d'un administrateur après exécution d'un fichier (fenêtres cmd fugaces) : infection machine, pas faute interne. |
| Banque | Framed (fraud) | Virement atypique sur BCT Ops attribué à l'administrateur `Marc de Lacroix` ; l'analyse des journaux doit départager sabotage vs complicité — le tableau accuse, la technique disculpe. |
| Hôpital | Ransom Me (ransomme) | Ransomware sur les cinq plateformes du HCT ; chaîne reconstituée depuis snapshot + logs pré-chiffrement, acteur identifié. |
| Opérateur électrique | Shockwave (tervagrid_1) | Root sur la passerelle SCADA `scada-gw-01` (débordement DNP3) et implant d'exfiltration radio AFSK (« Live Wire ») : l'anomalie « sur les ondes » est une exfiltration, pas un défaut de télémétrie. |
| Fournisseur d'eau urbain | Shutter Island (empreinte) | Fuite de données chez AquaTerv (réseau urbain d'Averis) : exfiltration documentée, origine et ampleur à chiffrer. |
| Mail de phishing | Operation Blackout (ombre) | Spear-phishing contre `j.aterbe@bct.gov.trv` : domaine DKIM homographe IDN de VARELCORP (`xn--varelcrp-s4a.var`), relais d'envoi `relay-mail.kalnet.kal` (203.0.113.10), macro anti-VM. |

## Conclusion d'attribution

Le fil conducteur est l'**empreinte VARELCORP** : homographe de domaine dans le phishing, certificats forgés par l'autorité fantôme Borealis CA au nom de VARELCORP (mission Ghost Authority), campagnes de désinformation coordonnées (The Great Pretender). Les six incidents ne sont pas six attaquants : c'est une campagne unique, instrumentée par l'écosystème VARELCORP/Varélia, dont chaque dossier est un vecteur différent (phishing, ransomware, intrusion SCADA, insider framé).

## Lessons learned

- Un « rapport provisoire » se note **pendant** l'épreuve : les conclusions par dossier doivent rester des faits établis, pas des hypothèses.
- L'attribution se fait sur les recoupements techniques (domaines, certificats, infrastructures partagées), pas sur le narratif.
- Garder une trace par dossier (IP, domaines, hashes) rend la synthèse finale triviale.
