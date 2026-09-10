# Audit du CDC v1.0 — corrections à intégrer

Audit mené le 7 septembre 2026 contre l'instance n8n réelle et la boîte Gmail.
Le principe directeur du CDC — « Gmail qualifie, n8n exécute » — est conservé sans
changement. Les corrections ci-dessous portent sur des points d'implémentation qui
rendaient le CDC non déployable en l'état.

## État constaté de l'instance

- Instance n8n **vide** : 0 credential, 0 workflow, 0 data table.
- Le connecteur MCP **ne peut pas créer de credentials** (flux OAuth) → étape manuelle.
- **Aucun** des 4 labels Gmail du §4.1 n'existe.
- Système de tri déjà en place : `💰 Compta` (`Label_6800194937859133668`, 1 389 messages),
  `📌 Signal`, `🔇 Bruit`, `⏳ En attente`, arborescence `Aktimmo/*`.

## Corrections

### C1 — Le nœud Dropbox n'expose aucune action de lien de partage

Le §6.11 laissait le point « à vérifier au montage ». Vérifié : les ressources du nœud
`n8n-nodes-base.dropbox` sont `file` (copy/delete/download/move/upload), `folder` et
`search`. Aucune opération de partage.

Le repli HTTP Request n'est donc pas un repli, c'est le chemin normal : `HTTP Request` avec
le type de credential prédéfini `dropboxOAuth2Api`, sur
`sharing/create_shared_link_with_settings`. Le piège 409 anticipé par le CDC est réel et se
traite par un appel à `sharing/list_shared_links`.

### C2 — La colonne `id` est réservée

Les Data Tables n8n génèrent automatiquement une colonne `id` numérique. Déclarer une
colonne utilisateur `id` entre en collision. La clé `{messageId}-{indexPieceJointe}` est
donc portée par **`cle_piece`**.

### C3 — L'anti-doublon ne couvre pas le cas réel le plus fréquent

`{messageId}-{index}` protège contre le retraitement du *même mail*, pas contre la *même
facture reçue dans plusieurs mails*.

Constaté : Sellsy envoie « Votre facture Sellsy » (PDF joint), « Votre paiement a été
validé » puis « Merci pour votre paiement » (PDF joint) — 28/07 et 31/07 2026, même facture
de 1 699,20 €. Même schéma chez income le 12/05 : trois mails « Note d'honoraires » dans un
seul fil.

Sans garde-fou, Sophie reçoit la ligne deux ou trois fois.

**Correction :** lookup Data Table `rowExists` sur `fournisseur` + `numero_facture` après
extraction et avant dépôt. Si la pièce est déjà connue : poser le label `OK`, ne rien
déposer, ne rien journaliser.

> **Corrigé en recette.** Une facture et son reçu de paiement portent le même numéro : cette
> clé à deux termes aurait écarté le reçu en silence. La clé retenue est
> `fournisseur` + `numero_facture` + `type_document` — voir
> [`recette.md`](recette.md), défaut 1.

### C4 — Les transferts à Sophie seraient ré-ingérés

Le process actuel est le `Fwd:` vers `sophie@solead-gestion.fr` ; le PDF est ré-attaché
dans la copie SENT. Gmail applique un label posé à la main au fil entier, donc
`label:Justif has:attachment filename:pdf` remonte l'original **et** la copie envoyée.

**Correction :** ajouter `-in:sent -from:me` à la requête.

### C5 — L'entité obligatoire condamne EDF et Sellsy au traitement manuel

EDF (`service-facturation-ma@edf.fr`) et Sellsy (`adv@sellsy.com`) arrivent sur l'adresse
Gmail personnelle et sont adressées à « Nicolas LARET » / « Nicolas LARET-2 » — pas à
AKTIMMO, ATTRAKTION ou COMAKT. Le §6.9 et la recette 9 imposent : pas d'entité →
`A-verifier`. Les deux fournisseurs les plus réguliers tomberaient en manuel chaque mois.

**Correction :** quatrième valeur d'entité **`PERSO`**, pour les factures adressées à une
personne physique. `A-verifier` reste réservé aux vraies illisibilités.

### C6 — Risque d'ingérer les factures de vente

`notification@evoliz.com` transmet les factures qu'Attraktion et Aktimmo **émettent**
(F-20260900003 → Fimavi, loyers Castel@Work…). Le principe « Gmail qualifie » reporte tout
le risque sur les règles.

**Correction :** règles Gmail **par expéditeur uniquement**, jamais par mot-clé « facture ».

### C7 — Le Filter sur `application/pdf` ne peut pas fonctionner

Les pièces jointes arrivent en `attachment_0`, `attachment_1`… dans `$binary`, sur **un
seul item**. Un nœud `Filter` travaille sur le JSON et ne voit pas ces clés.

**Correction :** nœud `Code` qui éclate les binaires en items — un item par PDF, mimeType
filtré, binaire renommé `data` (ce que `Extract from File` lit par défaut), index conservé
pour la construction de `cle_piece`. C'est aussi ce qui fait passer le test 6 du §11.

### C8 — Gmail Send ajoute une mention automatique

`options.appendAttribution` vaut `true` par défaut et ajoute « This email was sent
automatically with n8n » aux mails à Sophie. À passer à `false`.

### C9 — Add Label attend des IDs

L'opération `addLabels` prend des `labelIds`, pas des noms. Les 4 labels doivent exister
avant la construction des workflows et leurs IDs être figés dans les nœuds.

### C10 — Dropbox Upload et la création d'arborescence

Le §6.10 affirme que le chemin crée l'arborescence au passage. La documentation du nœud dit
l'inverse : « The parent folder has to exist. » À trancher en recette ; en attendant,
`folder:create` en amont avec `onError: continue`.

### C11 — `newer_than:30d` n'est pas qu'un garde-fou de première exécution

Le filtre est permanent. Une panne de 35 jours fait sortir les mails de la file
définitivement — ils n'ont pas le label `OK`, mais ils ne sont plus dans la fenêtre.

**Correction :** `newer_than:90d` en régime, et `limit: 20` sur la première exécution pour
éviter d'envoyer 40 à 60 PDF d'un coup dans le modèle.

### C12 — Le numéro de facture n'est pas nettoyé

Le §6.8 nettoie le nom du fournisseur mais pas le numéro, alors que celui-ci entre dans le
nom de fichier. Les caractères `/ \ : * ? " < > |` sont interdits par Dropbox.

### C13 — Trois nœuds LLM là où un suffit

`Information Extractor` remplace `Basic LLM Chain` + `Structured Output Parser` : schéma
intégré, options de batching et de rate-limit natives.

### C14 — Diagnostic impossible sans rouvrir Gmail

Ajout de `expediteur` et `sujet_mail` au journal. Coût nul, et c'est ce qui permet de
comprendre une extraction douteuse sans repartir de la boîte.

### C15 — La racine Dropbox n'est pas celle du CDC

Le §4.3 annonce une racine `/Justificatifs/`. Vérification faite par interrogation directe de
l'API le 7 septembre 2026, le dossier réel est **`/Bannette-Numérique/Justificatifs/`**, avec
`/Bannette-Numérique/Justificatifs/A-classer/` pour le repli.

L'API Dropbox raisonne en chemin relatif à la racine du compte connecté, jamais en chemin
disque. Coder `/Justificatifs/` en dur aurait créé un second dossier vide à la racine et
déposé les factures dedans, sans jamais toucher le dossier voulu — en silence.

Deux observations du même relevé :

- Le compte est un **Dropbox Business**. La racine par défaut renvoyée par l'API est la
  bonne, donc aucun en-tête `Dropbox-API-Path-Root` n'est nécessaire.
- Le chemin contient un accent (`Numérique`). Dropbox gère l'UTF-8 sans difficulté et le
  relevé le confirme ; aucun renommage n'est utile.

### C16 — Le nœud Dropbox natif est abandonné au profit d'HTTP Request

Corrige C1. La credential Dropbox prédéfinie de n8n demande une liste de scopes figée qui ne
contient pas `sharing.write` : les fichiers seraient déposés, mais aucun lien de partage ne
pourrait être créé.

La chaîne utilise donc une credential **`oAuth2Api` générique** pointant sur les endpoints
Dropbox, avec les six scopes déclarés à la main, et `token_access_type=offline` en paramètre
de l'URL d'autorisation — sans lui, Dropbox ne délivre qu'un jeton de 4 heures.

Une seule credential couvre alors le dépôt et le partage, tous deux par HTTP Request.

## Points ouverts du §12, résolus

| # | Résolution |
|---|---|
| 12.1 | Cordier Roche : `document@obat.fr`, PDF signé « Sarl Cordier Roche », lecture OK. Dalkia : expéditeur réel **`dalkia@serensia.net`**, pas `dalkia.fr`. EDF et Sellsy : voir C5. |
| 12.2 | **Aucun** des 4 labels n'existe. Système en place documenté ci-dessus. À créer à plat, **pas** sous `💰 Compta` — emoji et espace casseraient la requête `label:`. |
| 12.4 | **Stericlean trouvé** : `invoice2go@communications.2go.com`, objet « Facture N°628 de SteriClean », vers `info@aktimmo.fr`. Le fournisseur n'apparaît pas dans le domaine expéditeur, ce qui valide l'approche « lire le PDF » du §5.2. **Bel Art : toujours introuvable** sur six mois. |
| 12.6 | income : l'entité figure dans l'objet (`- ATTRAKTION`, `- COMAKT`) depuis juin, mais **pas** sur les trois mails du 12/05, intitulés « income - Note d'honoraires » sans suffixe. L'objet n'est donc pas un repli fiable ; la distinction doit venir du PDF. |

Deux de ces points ont été tranchés depuis : le **récap hebdomadaire** part le **vendredi à
9h**, et la **dérive du nom de fournisseur** (§6.8) n'a pas attendu la phase 2 — elle s'est
produite sur le premier fournisseur testé et a été traitée en recette par rapprochement sur
les dossiers Dropbox existants.

Reste ouvert : **Bel Art**, aucun expéditeur identifié.

## Décisions actées

1. Quatrième valeur d'entité `PERSO` (C5).
2. Construction des workflows désactivés, branchement des credentials ensuite.
3. Anti-doublon par `fournisseur` + `numero_facture` (C3), porté à trois termes en recette
   avec `type_document`.
4. Liens Dropbox publics acceptés — seul mode disponible en Dropbox Basic. Les factures
   comportent IBAN et adresses ; le lien est long et non indexé, le risque est assumé.
