# CAHIER DES CHARGES — Traitement automatisé des factures et justificatifs

**Version 1.0 — 7 septembre 2026**
**Destination :** implémentation dans n8n via Claude Code et le connecteur MCP n8n.

---

## 1 · OBJECTIF

Automatiser la chaîne qui va du mail reçu jusqu'à la transmission à la comptable, sans intervention manuelle une fois les labels posés.

Deux livrables :

1. **Une archive Dropbox** rangée et nommée de façon homogène, qui sert de mémoire longue.
2. **Deux mails récapitulatifs à Sophie**, avec les liens de téléchargement des originaux.

## 2 · PRINCIPE DIRECTEUR

> **Gmail qualifie. n8n exécute.**

Le workflow ne contient aucune logique de détection. Il ne cherche pas à savoir si un mail est une facture : il obéit au label. La qualification vit dans les règles Gmail et dans les labels posés à la main.

Conséquence opérationnelle : un nouveau fournisseur se traite par une règle Gmail, jamais par une modification du workflow.

## 3 · PÉRIMÈTRE

**Dans le périmètre**

- Récupération des pièces jointes PDF des mails labellisés
- Extraction du fournisseur, de la date de facture et du numéro de facture
- Dépôt Dropbox avec arborescence et nommage normalisés
- Marquage Gmail du traitement
- Envoi quotidien des factures à régler
- Envoi hebdomadaire des justificatifs

**Hors périmètre — décisions actées**

- **Le rapprochement des confirmations de virement.** Écarté : virements groupés, règlements partiels et écarts de montant rendent l'appariement automatique trop peu fiable. Un rapprochement faux et silencieux est pire que pas de rapprochement.
- **Le montant.** Ni dans le nom de fichier, ni dans le journal. Il n'a plus d'usage depuis l'abandon du rapprochement.
- **La récupération sur les espaces clients par navigateur.** Hors sujet en phase 1. Pour les fournisseurs qui n'envoient rien par mail, la solution est d'activer l'envoi par courriel dans leur espace client.
- **La TVA (`e.dasilva@income.ec`).** Ce n'est ni un justificatif ni une facture à payer, mais une validation à donner. Ne doit recevoir aucun des deux labels.

---

## 4 · PRÉREQUIS

### 4.1 Labels Gmail

Quatre labels. **Sans accent ni espace**, pour éviter tout échappement dans les requêtes de recherche.

| Label | Posé par | Rôle |
|---|---|---|
| `Justif` | règle Gmail ou manuel | Facture payée ou prélevée. Sophie archive. |
| `A-payer` | règle Gmail ou manuel | Sophie doit faire le virement. |
| `OK` | le workflow | Traitement réussi et déposé sur Dropbox. |
| `A-verifier` | le workflow | Lecture du PDF incomplète, à reprendre à la main. |

**Vérifier d'abord les labels existants** avant de créer : des labels de tri sont déjà posés sur ces mails.

### 4.2 Boîtes couvertes

Les factures arrivent sur trois adresses, toutes agrégées dans la même boîte Gmail :

- `nicolas@attraktion.fr`
- l'adresse Gmail personnelle (EDF, Sellsy)
- `info@aktimmo.fr` (Dalkia)

Les règles Gmail doivent couvrir les trois.

### 4.3 Dropbox

Racine : `/Justificatifs/`
Arborescence : `/Justificatifs/{ANNÉE}/{FOURNISSEUR}/`
Dossier de repli : `/Justificatifs/A-classer/`

**Un seul arbre pour les deux statuts.** `Justif` et `A-payer` se rangent au même endroit : le statut vit dans le label Gmail et dans le journal, jamais dans le chemin — sinon il faudrait déplacer les fichiers.

L'année vient de la **date de facture**, jamais de la date de réception du mail. C'est ce qui protège des décalages de fin d'année, où une facture de décembre arrive en janvier.

### 4.4 Identifiants à configurer

- Gmail OAuth2 (lecture, écriture de labels, envoi)
- Dropbox OAuth2 (écriture et création de liens partagés)
- Anthropic API (nœud LLM)

---

## 5 · MODÈLE DE DONNÉES

Une seule Data Table n8n. Choix acté : pas de Google Sheets, pour ne pas ajouter au système un outil non utilisé par ailleurs. Le volume attendu — quelques dizaines de lignes par mois — est très en deçà de la limite de 200 Mio par instance.

### 5.1 Table `journal_factures`

C'est la colonne vertébrale. Le workflow d'ingestion écrit, les deux workflows d'envoi lisent et marquent.

| Colonne | Type | Contenu |
|---|---|---|
| `id` | string | Clé unique : `{messageId}-{indexPieceJointe}` |
| `message_id` | string | Identifiant Gmail, pour remonter au mail d'origine |
| `fournisseur` | string | Nom normalisé |
| `date_facture` | string | `AAAA-MM-JJ` |
| `numero_facture` | string | Tel qu'il figure sur la facture |
| `entite` | string | `AKTIMMO`, `ATTRAKTION` ou `COMAKT` |
| `statut` | string | `Justif` ou `A-payer` |
| `chemin_dropbox` | string | Chemin complet du fichier déposé |
| `lien_dropbox` | string | Lien de partage |
| `date_traitement` | string | Date d'exécution, ISO |
| `envoye_le` | string | **Vide à la création.** Rempli après envoi du récap. |

**La colonne `envoye_le` est le mécanisme anti-doublon.** Les workflows d'envoi filtrent sur « pas encore envoyé », jamais sur une fenêtre de dates glissante.

C'est un choix de conception délibéré : une fenêtre à sept jours perdrait définitivement les pièces d'une semaine où l'exécution a échoué. Le filtre sur `envoye_le` vide rattrape tout seul au tour suivant.

### 5.2 Pas de table de correspondance fournisseurs — décision actée

Une seconde table associant chaque fournisseur à son entité a été envisagée, puis écartée.

**Raison :** le workflow ouvre déjà le PDF pour en extraire la date et le numéro. L'entité destinataire y figure nécessairement, puisque c'est elle qui est facturée. L'information est donc lue à la source, dans la même passe.

**Ce que ça évite :** une table à tenir à jour, et surtout un fournisseur inconnu qui échouerait tant que sa ligne n'a pas été créée. Ici, un nouveau prestataire fonctionne dès le premier envoi, sans aucune intervention préalable.

C'est la couche intelligente et adaptative du workflow : la plomberie reste déterministe, la lecture s'adapte.

**Cas qui valide l'approche :** Cordier Roche envoie depuis `document@obat.fr`, l'adresse de son logiciel de facturation. Une table indexée sur l'expéditeur aurait manqué le fournisseur. La lecture du PDF le trouve sans difficulté.

---

## 6 · WORKFLOW 1 — INGESTION QUOTIDIENNE

Quatre workflows séparés plutôt qu'un seul. Chacun est rejouable indépendamment : si l'envoi échoue, l'ingestion n'est pas à refaire.

### Nœuds

**1 · Schedule Trigger** — une fois par jour, heure fixe.

**2 · Gmail — Get Many Messages**

Requête :

```
(label:Justif OR label:A-payer) -label:OK -label:A-verifier has:attachment filename:pdf newer_than:30d
```

L'exclusion de `A-verifier` évite qu'un PDF illisible soit retraité tous les jours indéfiniment. Il ressort de la file dès que le label est retiré à la main.

Le `newer_than:30d` borne la première exécution, qui remonterait sinon tout l'historique.

**3 · Loop Over Items** — traitement message par message.

Indispensable : sans boucle, un échec sur une facture fait tomber l'exécution entière et les suivantes ne sont jamais traitées.

**4 · Gmail — Get Message**, option de téléchargement des pièces jointes activée.

C'est ce nœud qui produit le binaire. Le fichier reste un objet binaire dans le flux n8n et ne transite jamais par le contexte du modèle.

**5 · Filter** — ne conserver que les pièces jointes dont le type est `application/pdf`.

Élimine signatures, logos et images intégrées, qui représentent l'essentiel du bruit.

**6 · Extract from File** — conversion PDF vers texte, en local.

**Choix de conception majeur :** le modèle reçoit du texte, pas un PDF encodé. Moins coûteux, plus rapide, et aucune limite de contexte à gérer.

Effet de bord assumé : un PDF scanné sans couche texte ressort vide et part naturellement en `A-verifier`. C'est le comportement voulu.

**7 · Basic LLM Chain** avec **Structured Output Parser**

Modèle Claude. Sortie strictement contrainte à quatre champs.

Consigne :

> Tu extrais quatre informations d'une facture. Réponds uniquement avec les champs demandés, sans commentaire.
> — `fournisseur` : la raison sociale de l'émetteur de la facture, pas celle du destinataire. Donne la forme commerciale courte, sans forme juridique : « Dalkia », pas « DALKIA SA ».
> — `date_facture` : la date d'émission de la facture, au format AAAA-MM-JJ. Pas la date d'échéance, pas la date de règlement.
> — `numero_facture` : le numéro de la facture tel qu'il est écrit.
> — `entite` : l'entité destinataire de la facture, celle qui est facturée. Trois valeurs possibles uniquement : `AKTIMMO`, `ATTRAKTION`, `COMAKT`. Une facture adressée à « AKTIMMO 1 » ou à une autre déclinaison est rattachée à `AKTIMMO`. Si le destinataire ne correspond à aucune des trois, renvoie une chaîne vide.
>
> Si une information est absente ou illisible, renvoie une chaîne vide pour ce champ. N'invente jamais une valeur.

Schéma de sortie :

```json
{
  "fournisseur": "string",
  "date_facture": "string",
  "numero_facture": "string",
  "entite": "string"
}
```

**L'entité est un champ obligatoire au même titre que les trois autres.** Sans elle, Sophie ne sait pas quel compte débiter : une ligne sans entité n'a pas d'intérêt pour elle. Elle entre donc dans le test de complétude du nœud 9.

**8 · Code — normalisation**

C'est ici que vit la convention. Ce nœud :

- normalise la date en `AAAA-MM-JJ` (accepter les formats `01/07/2026`, `1er juillet 2026`, `2026-07-01`)
- nettoie le nom de fournisseur : sans accent, espaces remplacés par des tirets, formes juridiques retirées (`SA`, `SAS`, `SARL`, `SASU`)
- construit le chemin et le nom de fichier
- construit l'`id` unique

**Point de vigilance connu :** le nom de fournisseur venant du modèle, une variation de mise en page peut produire `Dalkia` un mois et `Dalkia-Energie` le suivant, donc deux dossiers Dropbox pour un même fournisseur. Le nettoyage déterministe limite le risque sans l'éliminer.

La dérive est visible et se corrige à la main dans Dropbox en fusionnant les dossiers. Si elle devenait fréquente, la parade serait de lister les dossiers existants avant le dépôt et de rapprocher le nom par similarité — non retenu en phase 1 pour ne pas alourdir la chaîne.

**Nommage du fichier :**

```
AAAA-MM-JJ_Fournisseur_NumeroFacture.pdf
```

Le tri alphabétique devient chronologique, et la recherche par fournisseur reste immédiate.

**Chemin complet :**

```
/Justificatifs/{année de date_facture}/{Fournisseur}/{nom de fichier}
```

**9 · IF — les trois champs sont-ils renseignés ?**

*Branche NON :*
- Dropbox Upload vers `/Justificatifs/A-classer/`
- Gmail Add Label `A-verifier`
- Aucune écriture dans le journal
- Fin de l'itération

*Branche OUI :* on continue.

**10 · Dropbox — Upload File**

Le chemin crée l'arborescence au passage : aucun nœud de création de dossier n'est nécessaire.

**11 · Génération du lien de partage**

À vérifier au montage : le nœud Dropbox natif n'expose peut-être pas cette action. Repli par HTTP Request sur l'API Dropbox.

**Piège connu à traiter :** `sharing/create_shared_link_with_settings` renvoie une erreur 409 si un lien existe déjà pour ce fichier. Le workflow doit alors appeler `sharing/list_shared_links` pour récupérer le lien existant, plutôt que d'échouer.

**12 · Gmail — Add Label `OK`**

**Uniquement après confirmation du dépôt Dropbox.** Jamais avant.

Si Dropbox échoue, le mail reste sans `OK` et repasse le lendemain. Rien ne se perd silencieusement, et le coup d'œil dans la boîte Gmail suffit à voir ce qui a fonctionné.

**13 · Data Table — Insert Row** dans `journal_factures`, `envoye_le` laissé vide.

**Ordre impératif :** dépôt Dropbox, puis lien, puis label, puis journal. Si l'écriture du journal précédait le dépôt, une panne Dropbox laisserait une ligne pointant vers un fichier inexistant, et le récap enverrait un lien mort à Sophie.

---

## 7 · WORKFLOW 2 — ENVOI QUOTIDIEN « À RÉGLER »

Rythme quotidien, et c'est le point qui justifie la séparation des deux flux : une facture à payer ne doit pas attendre jusqu'à six jours avant que Sophie la voie. Un fournisseur avec une échéance courte passerait juste.

**1 · Schedule Trigger** — quotidien, décalé après le workflow 1.

**2 · Data Table — Get Rows**
Filtre : `statut = A-payer` **ET** `envoye_le` vide.

**3 · IF** — y a-t-il des lignes ? Si non, fin sans envoi. Pas de mail vide sur ce flux : il est actionnable, un mail sans action n'a pas de sens.

**4 · Code** — assemblage du tableau HTML.

**5 · Gmail — Send Message**

Objet : `[Admin Nico] À régler`

Corps, registre habituel avec Sophie — informel, tutoiement, « Coucou ma So » en ouverture, « Biz » en clôture.

Colonnes du tableau : **Fournisseur · Date de facture · Numéro · Entité · Lien de téléchargement**

La colonne entité est indispensable : c'est elle qui indique quel compte débiter.

**6 · Data Table — Update Row** sur les lignes envoyées, `envoye_le` renseigné.

**Ce nœud ne s'exécute qu'après le succès de l'envoi.** C'est lui qui garantit l'absence de doublon au tour suivant.

---

## 8 · WORKFLOW 3 — RÉCAP HEBDOMADAIRE « JUSTIFICATIFS »

**1 · Schedule Trigger** — hebdomadaire, jour à définir.

**2 · Data Table — Get Rows**
Filtre : `statut = Justif` **ET** `envoye_le` vide.

**3 · IF** — lignes présentes ou non.

**Cas vide : envoyer quand même un mail court.** L'absence de nouveauté est une information — elle confirme à Sophie que le système tourne. C'est la différence avec le flux « à régler ».

**4 · Code** — assemblage, tri par fournisseur.

**5 · Gmail — Send Message**

Objet : `[Admin Nico] Justificatifs`

Même structure de tableau, même registre.

**6 · Data Table — Update Row**, `envoye_le` renseigné.

---

## 9 · WORKFLOW 4 — GESTION DES ERREURS

**Error Trigger** rattaché aux trois workflows précédents, qui envoie un mail à `nicolas@attraktion.fr` en cas d'échec d'exécution.

**Sans ce workflow, un automatisme qui cesse de tourner ne se remarque qu'au bout de trois semaines.** C'est le mode de panne le plus fréquent et le plus coûteux de ce type de chaîne.

---

## 10 · TRI CÔTÉ SOPHIE

Le crochet `[Admin Nico]` est présent dans l'objet des deux récapitulatifs, qui sont des mails composés par le workflow.

Si des transferts de mails d'origine sont ajoutés plus tard, noter que **l'objet d'un transfert n'est pas modifiable** par les outils disponibles : il reprend celui du mail d'origine. Le marqueur devrait alors être placé en première ligne du commentaire de transfert, et le tri chez Sophie reposer sur un filtre par expéditeur.

---

## 11 · RECETTE

Tests à passer avant mise en production.

1. **Cas nominal** — un mail `Justif` avec un PDF lisible : fichier au bon chemin, nom conforme, label `OK` posé, ligne au journal avec `envoye_le` vide.
2. **Cas illisible** — un PDF scanné : fichier dans `A-classer`, label `A-verifier`, aucune ligne au journal.
3. **Anti-doublon** — relancer le workflow 1 immédiatement : aucun retraitement.
4. **Anti-doublon envoi** — relancer le workflow 2 après un envoi : aucun second mail.
5. **Décalage d'année** — une facture datée de décembre reçue en janvier : classée dans l'année de la facture.
6. **Deux pièces jointes** — un mail portant deux PDF : deux lignes distinctes au journal, deux fichiers déposés.
7. **Lien existant** — redéposer un fichier déjà partagé : le lien est récupéré, pas d'erreur 409 bloquante.
8. **Échec Dropbox simulé** — le label `OK` ne doit pas être posé.
9. **Entité illisible** — une facture dont le destinataire n'est pas identifiable : doit partir en `A-verifier`, jamais être rattachée à une entité par défaut.
10. **Entité déclinée** — une facture adressée à « AKTIMMO 1 » : doit ressortir en `AKTIMMO`.

---

## 12 · POINTS OUVERTS

1. **Fiabilité de la lecture d'entité** — à valider sur un échantillon réel, en particulier EDF, Sellsy et Cordier Roche, dont on ne sait pas encore si le destinataire est clairement identifiable sur la facture.
2. **Labels existants** à inventorier avant création des quatre nouveaux.
3. **Règles Gmail** à écrire pour poser `Justif` et `A-payer` automatiquement sur les expéditeurs récurrents.
4. **Bel Art et Stericlean** — adresses d'expédition à identifier pour leurs règles Gmail. Elles ne sont pas remontées sur les six derniers mois.
5. **Jour d'envoi** du récap hebdomadaire.
6. **Cas income** — deux notes d'honoraires arrivent systématiquement par paire, avec l'entité dans l'objet. À vérifier que la lecture du PDF suffit à les distinguer.

---

## 13 · ÉVOLUTIONS ENVISAGÉES, NON RETENUES EN PHASE 1

- **Vue de l'encours** — la table `journal_factures` permettrait d'afficher ce qui reste en `A-payer` depuis plus de N jours. À considérer quand la chaîne sera stabilisée.
- **Récupération sur espaces clients** — pour les fournisseurs sans envoi par mail. Nécessiterait un agent navigateur avec profil authentifié persistant, donc un projet technique distinct.
