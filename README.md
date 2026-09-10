# Traitement automatisé des factures et justificatifs

Chaîne n8n qui va du mail reçu à la transmission à la comptable, sans intervention manuelle
une fois les labels posés.

**Gmail qualifie. n8n exécute.** Le workflow ne contient aucune logique de détection : il
obéit au label. Un nouveau fournisseur se traite par une règle Gmail, jamais par une
modification du workflow.

## Documentation

- [`docs/exploitation.md`](docs/exploitation.md) — les identifiants, où changer les horaires,
  comment lire un incident, comment rejouer.
- [`docs/recette.md`](docs/recette.md) — les huit défauts trouvés sur de vraies factures et ce
  qu'ils ont changé dans la chaîne.
- [`docs/audit-cdc-v1.0.md`](docs/audit-cdc-v1.0.md) — les 16 corrections apportées au cahier
  des charges v1.0 après audit de l'instance et de la boîte.
- [`docs/regles-gmail.md`](docs/regles-gmail.md) — les labels, les expéditeurs identifiés, et
  ceux qu'il ne faut surtout pas labelliser.
- [`docs/evolutions.md`](docs/evolutions.md) — le carnet des idées pour la suite. Rien de ce
  qui s'y trouve n'est décidé ni engagé.

Le code SDK des workflows n'est pas dupliqué ici : n8n conserve l'historique de versions de
chaque workflow, qui fait foi.

## Architecture

Quatre workflows séparés, chacun rejouable indépendamment : si un envoi échoue, l'ingestion
n'est pas à refaire.

| Workflow | Rythme | Rôle |
|---|---|---|
| WF1 — Ingestion | quotidien 6h | Gmail → extraction PDF → Dropbox → journal |
| WF2 — À régler | quotidien 8h | Récap `A-payer` à Sophie |
| WF3 — Justificatifs | hebdomadaire | Récap `Justif` à Sophie |
| WF4 — Erreurs | sur échec | Alerte à `nicolas@attraktion.fr` |

Les récapitulatifs partent à `sophie@solead-gestion.fr`, avec `nicolas@attraktion.fr` en copie
le temps du rodage.

Le rythme quotidien de WF2 est ce qui justifie la séparation d'avec WF3 : une facture à
payer ne doit pas attendre jusqu'à six jours.

## Modèle de données

Data Table n8n `journal_factures` (`RjKH530pzYwbIkIk`, projet personnel). Seize colonnes,
toutes en `string`. La colonne `id` n'est pas déclarée — n8n la génère.

`cle_piece` · `message_id` · `expediteur` · `sujet_mail` · `fournisseur` · `date_facture` ·
`numero_facture` · `entite` · `type_document` · `montant_ttc` · `devise` · `statut` ·
`chemin_dropbox` · `lien_dropbox` · `date_traitement` · `envoye_le`

`entite` ∈ `AKTIMMO` | `ATTRAKTION` | `COMAKT` | `PERSO` | `""`
`type_document` ∈ `facture` | `recu` | `avoir` | `relance` | `autre`

**`montant_ttc` est un confort, pas une obligation.** C'est le montant réellement dû toutes
taxes comprises, stocké en décimal simple (`5040.00`) avec sa `devise` à côté. Il n'entre pas
dans le test de complétude : une pièce dont le montant est illisible se dépose et se
journalise normalement, elle ne part pas en `A-verifier`. Il n'apparaît ni dans le chemin ni
dans le nom de fichier — uniquement au journal et dans les tableaux envoyés à Sophie.

**`envoye_le` est le mécanisme anti-doublon d'envoi.** Les workflows d'envoi filtrent sur
« pas encore envoyé », jamais sur une fenêtre de dates glissante : une fenêtre à sept jours
perdrait définitivement les pièces d'une semaine où l'exécution a échoué.

## Arborescence Dropbox

Vérifié le 7 septembre 2026 par interrogation directe de l'API.

```
/Bannette-Numérique/Justificatifs/{ENTITÉ}/{ANNÉE}/{FOURNISSEUR}/
    AAAA-MM-JJ_ENTITÉ_Fournisseur_Numero_type.pdf
/Bannette-Numérique/Justificatifs/A-classer/
```

L'**entité** est le premier niveau : trois sociétés, trois comptabilités. Elle figure aussi
dans le nom, pour que le fichier reste lisible une fois téléchargé hors de son dossier. Le
**type** (`facture`, `recu`, `avoir`, `relance`) clôt le nom : une facture et son reçu de
paiement portent le même numéro et se marcheraient dessus sans lui.

**Le dossier n'est pas à la racine du Dropbox.** L'API Dropbox raisonne en chemin relatif à
la racine du compte connecté, jamais en chemin disque. Coder `/Justificatifs/` en dur aurait
créé un second dossier vide à la racine sans jamais toucher celui-ci.

Le **statut**, lui, reste hors du chemin : un `A-payer` devient payé, il faudrait déplacer
le fichier. Il vit dans le label Gmail et dans le journal. Une entité, elle, ne change
jamais — c'est ce qui autorise à la mettre dans le chemin.

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

Faite. `journal_factures`, seize colonnes.

### Étapes 2 à 5 — Workflows

Faites. Les quatre workflows sont publiés depuis le 10 septembre 2026.

### Étape 6 — Règles Gmail

**C'est ce qui reste à faire, et c'est ce qui fait vivre la chaîne.** Sans règles, seuls les
mails labellisés à la main entrent dans la file : le système tourne mais ne reçoit rien.
Les expéditeurs relevés sur six mois sont dans [`docs/regles-gmail.md`](docs/regles-gmail.md).

## Rejeu

Chaque étage se rejoue seul.

- **Une pièce partie à tort en `A-verifier`** : retirer le label à la main, elle repasse dans
  la file au prochain tour.
- **Un dépôt Dropbox échoué** : le label `OK` n'est pas posé, le mail repasse le lendemain.
  L'ordre dépôt → lien → label → journal garantit qu'aucune ligne ne pointe vers un fichier
  inexistant.
- **Un récap non parti** : `envoye_le` est resté vide, le tour suivant rattrape.
