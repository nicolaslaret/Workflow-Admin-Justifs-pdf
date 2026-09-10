# Évolutions envisagées

Ce fichier n'est pas un plan et n'engage aucune date. C'est le carnet des idées qui
méritent d'être reprises plus tard, écrites au moment où elles viennent pour ne pas être
reperdues.

**Rien de ce qui est ici n'est décidé.** Ce qui est décidé part dans le CDC, dans
[`exploitation.md`](exploitation.md) ou dans le code.

## Comment ajouter une idée

Une idée = une section `## E{n} — Titre`, numérotée à la suite, jamais renumérotée même si
une idée est abandonnée : les numéros servent de référence dans les conversations et les
commits.

Chaque section suit le même découpage :

- **État** — `à creuser` · `prête à chiffrer` · `en cours` · `faite` · `abandonnée` (avec la
  raison, elle vaut souvent plus que l'idée).
- **L'idée** — ce qu'on veut, en clair, du point de vue de l'usage.
- **Ce que ça change** — l'impact sur l'existant, workflow par workflow.
- **Points à trancher** — les questions ouvertes. C'est la partie la plus utile : elle évite
  de redécouvrir six mois plus tard qu'un point bloquait.

---

## E1 — Dépôt manuel par dossier Dropbox

**État :** à creuser

### L'idée

Un dossier `Dépôt manuel` dans la Bannette, dans lequel Nicolas glisse n'importe quel PDF —
une facture reçue hors mail, un justificatif scanné, un reçu photographié. Une tâche
planifiée regarde régulièrement ce qu'il contient et fait passer chaque pièce **exactement
par le même tube** que celles qui arrivent par mail : lecture du document, extraction des
champs, renommage, classement dans le bon dossier d'entité, écriture au journal, puis
apparition dans les récapitulatifs à Sophie.

La seule différence est la porte d'entrée. Aujourd'hui c'est un mail labellisé ; là ce serait
un fichier posé dans un dossier. Tout l'aval est identique et n'est pas à réécrire.

Le dossier contient un sous-dossier `Archives`, **réservé au système**. Une fois la pièce
traitée avec succès — copiée, renommée et rangée à sa destination définitive — le fichier
source y est déplacé. C'est ce déplacement qui le sort de la file : au tour suivant, la
racine du dépôt manuel ne contient plus que ce qui reste à faire.

Le fichier source n'est jamais supprimé. En cas de doute sur un classement, l'original brut,
avec son nom d'origine, reste consultable.

### Ce que ça change

**Un cinquième workflow**, WF5, et lui seul. Il remplace les quatre premiers nœuds de WF1
(Gmail → éclatement des pièces jointes) par : lister le dossier, télécharger le fichier,
entrer dans la boucle. À partir de la lecture du document, c'est la même chaîne de nœuds que
WF1, à recopier telle quelle.

**Rien à changer dans WF2 et WF3.** Ils lisent le journal, pas les mails. Une pièce déposée à
la main y apparaît comme les autres, sans une ligne de code en plus. C'est le bénéfice de la
séparation en quatre workflows.

**Le déplacement vers `Archives` tient le rôle du label `OK`.** Il doit donc obéir au même
ordre impératif : dépôt → lien de partage → journal → **puis seulement** déplacement de la
source. Si le workflow tombe avant le déplacement, le fichier est relu au tour suivant et
c'est la clé anti-doublon (`fournisseur` + `numero_facture` + `type_document`) qui le
rattrape — comme elle rattrape déjà les mails Sellsy envoyés en trois exemplaires.

### Points à trancher

**Comment le système sait-il si c'est `A-payer` ou `Justif` ?** Un mail porte un label, un
fichier posé dans un dossier ne porte rien. La réponse la plus cohérente avec le principe de
la chaîne — *la qualification vit hors de n8n* — est de faire qualifier par le dossier :
`Dépôt manuel/À payer/` et `Dépôt manuel/Justificatifs/`, chacun scruté séparément. Gmail
qualifie pour les mails, l'arborescence qualifie pour les dépôts. À valider, mais c'est la
piste à instruire en premier.

**Les images.** L'idée parle de PDF *ou d'image*, et c'est là que ça se complique : la chaîne
actuelle lit la couche texte d'un PDF. Une photo de ticket n'en a pas et partirait
directement en `A-verifier` — ce qui viderait l'idée de son intérêt, puisque le cas le plus
naturel du dépôt manuel est justement le reçu photographié. Il faut soit convertir en amont,
soit envoyer l'image au modèle en vision plutôt qu'en texte. Le second chemin est le bon,
mais il change le nœud d'extraction et mérite d'être chiffré à part. C'est le vrai coût de
cette évolution, pas le reste.

**Trois colonnes du journal n'ont plus de sens** : `message_id`, `expediteur`, `sujet_mail`.
Les laisser vides fait perdre la traçabilité qu'elles apportent ; y écrire le nom du fichier
d'origine et `depot-manuel` la conserve à moindres frais. Se poser au passage la question
d'une colonne `source` (`mail` / `depot-manuel`), qui dirait la provenance sans détourner le
sens des colonnes existantes.

**La clé de pièce.** `cle_piece` vaut aujourd'hui `{messageId}-{index}`. Sans mail, il faut
autre chose — chemin d'origine, ou empreinte du contenu. L'empreinte a un avantage : elle
détecte le même fichier redéposé deux fois, avant même l'appel au modèle.

**Le fichier en cours de copie.** Un PDF de plusieurs mégaoctets déposé depuis un téléphone
peut être vu par le workflow alors que Dropbox n'a pas fini de le synchroniser. Ne traiter
que les fichiers dont l'horodatage de modification a plus de quelques minutes évite de lire
un fichier tronqué.

**La fréquence.** WF1 tourne une fois par jour à 6h, ce qui convient à des mails. Pour un
dépôt manuel, l'attente attendue est plus courte : on pose un fichier et on aimerait le voir
rangé dans l'heure. Un rythme horaire en journée est probablement le bon compromis, à
condition de rester en dessous du volume d'appels au modèle qu'on accepte de payer.

**Les collisions de noms.** Deux fichiers `scan.pdf` déposés à deux semaines d'écart
atterriraient sur le même nom dans `Archives`. Dropbox renommerait le second en `scan (1).pdf`
— acceptable pour une archive brute, mais à constater plutôt qu'à subir.

---

## E2 — *(à venir)*
