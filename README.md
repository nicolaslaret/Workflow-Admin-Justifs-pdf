# Traitement automatisé des factures et justificatifs

Chaîne n8n qui va du mail reçu à la transmission à la comptable, sans intervention manuelle
une fois les labels posés.

**Gmail qualifie. n8n exécute.** Le workflow ne contient aucune logique de détection : il
obéit au label. Un nouveau fournisseur se traite par une règle Gmail, jamais par une
modification du workflow.

## Documentation

- [`docs/audit-cdc-v1.0.md`](docs/audit-cdc-v1.0.md) — les 14 corrections apportées au cahier
  des charges v1.0 après audit de l'instance et de la boîte.
- [`docs/regles-gmail.md`](docs/regles-gmail.md) — les labels, les expéditeurs identifiés, et
  ceux qu'il ne faut surtout pas labelliser.

## Architecture

Quatre workflows séparés, chacun rejouable indépendamment : si un envoi échoue, l'ingestion
n'est pas à refaire.

| Workflow | Rythme | Rôle |
|---|---|---|
| WF1 — Ingestion | quotidien 6h | Gmail → extraction PDF → Dropbox → journal |
| WF2 — À régler | quotidien 8h | Récap `A-payer` à Sophie |
| WF3 — Justificatifs | hebdomadaire | Récap `Justif` à Sophie |
| WF4 — Erreurs | sur échec | Alerte à `nicolas@attraktion.fr` |

Le rythme quotidien de WF2 est ce qui justifie la séparation d'avec WF3 : une facture à
payer ne doit pas attendre jusqu'à six jours.

## Modèle de données

Data Table n8n `journal_factures` (`RjKH530pzYwbIkIk`, projet personnel). Treize colonnes,
toutes en `string`. La colonne `id` n'est pas déclarée — n8n la génère.

`cle_piece` · `message_id` · `expediteur` · `sujet_mail` · `fournisseur` · `date_facture` ·
`numero_facture` · `entite` · `statut` · `chemin_dropbox` · `lien_dropbox` ·
`date_traitement` · `envoye_le`

`entite` ∈ `AKTIMMO` | `ATTRAKTION` | `COMAKT` | `PERSO` | `""`

**`envoye_le` est le mécanisme anti-doublon d'envoi.** Les workflows d'envoi filtrent sur
« pas encore envoyé », jamais sur une fenêtre de dates glissante : une fenêtre à sept jours
perdrait définitivement les pièces d'une semaine où l'exécution a échoué.

## Arborescence Dropbox

Vérifié le 7 septembre 2026 par interrogation directe de l'API.

```
/Bannette-Numérique/Justificatifs/{ANNÉE}/{FOURNISSEUR}/AAAA-MM-JJ_Fournisseur_Numero.pdf
/Bannette-Numérique/Justificatifs/A-classer/
```

**Le dossier n'est pas à la racine du Dropbox.** L'API Dropbox raisonne en chemin relatif à
la racine du compte connecté, jamais en chemin disque. Coder `/Justificatifs/` en dur aurait
créé un second dossier vide à la racine sans jamais toucher celui-ci.

Un seul arbre pour les deux statuts : `Justif` et `A-payer` se rangent au même endroit. Le
statut vit dans le label Gmail et dans le journal, jamais dans le chemin — sinon il faudrait
déplacer les fichiers.

L'année vient de la **date de facture**, jamais de la date de réception du mail. C'est ce qui
protège des décalages de fin d'année, où une facture de décembre arrive en janvier.

## Ordre de déploiement

### Étape 0 — Manuel, préalable à tout

1. Créer les quatre labels Gmail à plat : `Justif`, `A-payer`, `OK`, `A-verifier`.
2. Créer trois credentials dans l'UI n8n : Gmail OAuth2, Dropbox OAuth2, Anthropic API.
   Le connecteur MCP ne peut pas les créer — les flux OAuth passent par le navigateur.
3. Créer sur Dropbox `/Bannette-Numérique/Justificatifs/` et
   `/Bannette-Numérique/Justificatifs/A-classer/`.

### Étape 1 — Data Table

Faite. `journal_factures`, treize colonnes.

### Étapes 2 à 5 — Workflows

Construits désactivés, credentials à sélectionner ensuite sur chaque nœud.

### Étape 6 — Règles Gmail

Voir [`docs/regles-gmail.md`](docs/regles-gmail.md).

## Rejeu

Chaque étage se rejoue seul.

- **Une pièce partie à tort en `A-verifier`** : retirer le label à la main, elle repasse dans
  la file au prochain tour.
- **Un dépôt Dropbox échoué** : le label `OK` n'est pas posé, le mail repasse le lendemain.
  L'ordre dépôt → lien → label → journal garantit qu'aucune ligne ne pointe vers un fichier
  inexistant.
- **Un récap non parti** : `envoye_le` est resté vide, le tour suivant rattrape.
