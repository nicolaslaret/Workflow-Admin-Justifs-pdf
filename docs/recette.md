# Recette — ce que les vraies factures ont appris

Journal des essais du 7 septembre 2026, sur des pièces réelles. Chaque règle du workflow
vient d'un défaut constaté ici, pas d'une supposition.

## Pièces d'essai

| Document | Particularité |
|---|---|
| Reçu Stripe / Anthropic | Deux PDF dans un seul mail : une *Invoice* et un *Receipt* |
| Facture Vegaweb #366 | Mise en page InDesign, deux blocs d'adresse sans étiquette |

## Ce qui a été validé

- **Deux pièces jointes dans un mail donnent deux lignes distinctes** (cas 6 du §11).
- **`sharing.write` fonctionne** : les liens Dropbox sont créés.
- **Le chemin accentué passe** : l'échappement `é` de l'en-tête `Dropbox-API-Arg` tient.
- **Le lecteur PDF sort dans `$json.text`**, la documentation n8n se contredisant sur ce point.
- **La règle TVA fixe l'entité** : `ATTRAKTION` de façon stable sur les deux PDF Anthropic.

## Cinq défauts trouvés, et leur correction

### 1. Facture et reçu ne sont pas des doublons

Le premier garde-fou dédoublonnait sur `fournisseur` + `numero_facture`. Or une facture et
son reçu de paiement **portent le même numéro de facture** : le reçu aurait été supprimé en
silence. Comptablement ce sont deux pièces distinctes, et pour une facture déjà réglée le
reçu *est* le justificatif.

**Correction :** cinquième champ `type_document` (facture, recu, avoir, relance, autre), et
clé anti-doublon à trois termes.

### 2. Un caractère nul coupait le numéro de facture

Le texte extrait contient `9BF0758D<NUL>5387906` — un tiret que l'extraction PDF n'a pas su
rendre. Le modèle l'a interprété différemment à chaque passage : `9BF0758D 5387906`, puis
`9BF0758D5387906`. Même document, deux numéros, donc deux fichiers.

Première correction ratée : remplacer le caractère nul par une espace. Le modèle a alors lu
deux jetons distincts et n'a gardé que `9BF0758D`. **Correction retenue :** supprimer le
caractère nul, remplacer les autres caractères de contrôle par une espace.

### 3. L'entité n'apparaissait nulle part

Le CDC justifiait de tenir le **statut** hors du chemin — un `A-payer` devient payé, il
faudrait déplacer le fichier. L'argument est bon, mais il ne s'applique pas à l'entité :
**une entité ne change jamais**. Or c'est la partition la plus structurante, trois sociétés,
trois comptabilités.

**Correction :** `/Justificatifs/{ENTITE}/{ANNÉE}/{FOURNISSEUR}/`, et l'entité aussi dans le
nom de fichier pour qu'il reste lisible une fois téléchargé hors de son dossier.

### 4. Facture et reçu se retrouvaient avec le même nom

Même date, même entité, même fournisseur, même numéro. Dropbox a renommé le second en
`… (1).pdf` — un suffixe qui ne dit rien et qui dépend de l'ordre de traitement.

**Correction :** le type entre dans le nom : `…_facture.pdf`, `…_recu.pdf`.

### 5. Le fournisseur dérivait, puis a désigné le destinataire

Sur le même PDF, en trois passages : `Anthropic`, `Anthropic-Ireland`,
`Anthropic-Ireland-Limited`. Trois dossiers pour un fournisseur. Le §6.8 du CDC anticipait ce
risque et le renvoyait en phase 2 ; il s'est produit sur le tout premier fournisseur testé.

Pire, sur la facture Vegaweb le modèle a renvoyé **`Nicolas-Laret`** — le destinataire. Le
texte extrait explique pourquoi :

```
Nicolas Laret            ← destinataire, en premier
Attraktion
...
Sylvain Guillotte        ← émetteur, en second
N° SIRET : 49320063800028
```

Aucune étiquette « Facturé à », juste deux blocs qui se suivent, et la mise en page place le
destinataire d'abord.

**Deux corrections :**

- Le workflow **liste les dossiers déjà présents** dans l'entité avant chaque dépôt et
  réutilise le plus proche (préfixe commun d'au moins quatre caractères). `Anthropic-Ireland`
  retombe dans `Anthropic`. La colonne `fournisseur_lu` garde ce que le modèle avait dit.
- La consigne pose un point de départ déterministe : **cette chaîne ne traite que des pièces
  reçues**, donc Nicolas Laret et ses trois sociétés sont toujours le destinataire, jamais
  l'émetteur. Et sur une facture française, le bloc portant le SIRET est celui de l'émetteur.

### 6. `isEmpty` ne reconnaît pas une chaîne vide

WF2 ne remontait aucune ligne alors que le journal en contenait une, `A-payer`, avec
`envoye_le` vide. Le filtre `envoye_le isEmpty` des Data Tables n8n ne matche qu'une valeur
**nulle** — or l'insertion écrit une chaîne vide. Le même filtre sur `eq ""` remonte bien la
ligne, ce qui a confirmé le diagnostic.

Basculer sur `eq ""` aurait déplacé le problème : la procédure de rejeu documentée consiste à
**vider `envoye_le` à la main** dans la Data Table, ce qui produit vraisemblablement une
valeur nulle, et le filtre serait retombé en panne dans l'autre sens.

**Correction :** le tri sur `envoye_le` sort du filtre SQL et passe dans le nœud Code, qui
traite chaîne vide et valeur nulle de la même façon. Le filtre de la Data Table ne porte plus
que sur `statut`. Au volume attendu — quelques dizaines de lignes par mois — lire toutes les
lignes d'un statut à chaque tour ne coûte rien.

## Deuxième session d'essais — WF2 et WF3

Envois dirigés vers `nicolas@attraktion.fr` le temps des tests, puis rendus à Sophie.

| Test | Résultat |
|---|---|
| WF2 avec une ligne `A-payer` | Mail reçu, tableau à six colonnes, lien fonctionnel |
| WF2 relancé aussitôt | Aucun second mail — `envoye_le` fait son travail |
| WF3 avec deux lignes `Justif` | Mail reçu, facture et reçu distingués par la colonne Type |
| WF3 relancé aussitôt | Mail court « rien de neuf », aucune ligne remarquée |
| Fuseau horaire | `envoye_le` écrit en `+02:00`, l'heure de Paris est bien appliquée |

### 7. Deux récapitulatifs, un seul objet

Le CDC figeait l'objet des envois : `[Admin Nico] À régler` et `[Admin Nico] Justificatifs`.
Pendant la recette, deux mails WF3 sont partis à 27 secondes d'intervalle avec le même titre —
celui contenant les pièces, puis celui du test anti-doublon, vide. Le vide est arrivé en
dernier, donc en haut de la boîte, et c'est celui qu'on a ouvert.

En production, Sophie recevrait 52 mails par an intitulés `Justificatifs` et 250 intitulés
`À régler`, sans moyen de distinguer un envoi d'un autre ni de retrouver le bon par recherche.

**Correction :** l'objet porte la date et le nombre de pièces —
`[Admin Nico] Justificatifs du 10/09 — 2 pièces`, `[Admin Nico] À régler du 10/09 — 1 facture`,
et `— rien de neuf` quand la semaine est vide. Le crochet `[Admin Nico]` reste en tête, c'est
lui qui sert au tri chez Sophie.

## Reste à éprouver

- **Un PDF scanné sans couche texte** — la branche `A-verifier` n'a jamais été empruntée.
- **Le rattrapage 409** sur un lien de partage déjà existant.
- **Une facture AKTIMMO ou COMAKT** — seule ATTRAKTION a été rencontrée.
- **Le décalage d'année** — une facture de décembre reçue en janvier.
