# Règles Gmail — qualification des factures

La qualification vit ici, pas dans les workflows n8n. Un nouveau fournisseur se traite en
ajoutant une ligne à ce tableau et la règle correspondante, jamais en modifiant un workflow.

## Principe

**Une règle Gmail cible un expéditeur, jamais un mot-clé.**

Un filtre sur « facture » attraperait `notification@evoliz.com`, qui transmet les factures
qu'Attraktion et Aktimmo **émettent** (F-20260900003 vers Fimavi, les loyers Castel@Work).
Ingérer ses propres factures de vente comme des factures fournisseur corromprait le journal
en silence.

## Labels

Quatre labels, **à plat**, sans accent ni espace.

Ne pas les nester sous `💰 Compta` : ce label contient un emoji et une espace, la requête
deviendrait `label:"💰 Compta/Justif"` — exactement l'échappement que le CDC cherche à
éviter.

| Label | Posé par | Rôle |
|---|---|---|
| `Justif` | règle Gmail ou manuel | Facture payée ou prélevée. Sophie archive. |
| `A-payer` | règle Gmail ou manuel | Sophie doit faire le virement. |
| `OK` | le workflow | Traitement réussi, fichier déposé sur Dropbox. |
| `A-verifier` | le workflow | Lecture du PDF incomplète, à reprendre à la main. |

## Expéditeurs identifiés

Relevé sur les six derniers mois de la boîte.

| Expéditeur | Fournisseur | Destinataire | Label |
|---|---|---|---|
| `dalkia@serensia.net` | Dalkia | `info@aktimmo.fr` | `Justif` |
| `service-facturation-ma@edf.fr` | EDF | gmail perso | `Justif` |
| `adv@sellsy.com` | Sellsy | gmail perso | `Justif` |
| `payments-noreply@google.com` | Google Workspace | `nicolas@attraktion.fr` | `Justif` |
| `invoice2go@communications.2go.com` | SteriClean | `info@aktimmo.fr` | `A-payer` |
| `document@obat.fr` | Cordier Roche | `nicolas@attraktion.fr` | `A-payer` |
| `compta@income.ec` | income | `nicolas@attraktion.fr` | `A-payer` |

### Sans label — volontairement

| Expéditeur | Raison |
|---|---|
| `e.dasilva@income.ec` | TVA. Ni justificatif ni facture à payer, mais une validation à donner. Hors périmètre (§3 du CDC). |
| `notification@evoliz.com` | Factures **émises** par Attraktion et Aktimmo, pas reçues. |

### À identifier

- **Bel Art** — aucun mail retrouvé sur six mois. Adresse d'expédition inconnue, aucune
  règle possible pour l'instant.

## Points d'attention

**Sellsy et income envoient la même facture dans plusieurs mails.** Sellsy : « Votre facture
Sellsy », « Votre paiement a été validé », « Merci pour votre paiement » — deux d'entre eux
portent le PDF. income : trois mails le 12/05. Le workflow dédoublonne sur
`fournisseur` + `numero_facture`, mais il est inutile de labelliser les trois.

**income** envoie une paire mensuelle, une par entité. L'objet porte le suffixe
`- ATTRAKTION` ou `- COMAKT` depuis juin 2026, mais pas avant. La distinction repose sur la
lecture du PDF, pas sur l'objet.

**SteriClean** n'apparaît nulle part dans le domaine expéditeur
(`communications.2go.com`). C'est le cas qui valide l'approche « lire le PDF » plutôt qu'une
table de correspondance indexée sur l'expéditeur.
