# Évolutions envisagées

Ce fichier n'est pas un plan et n'engage aucune date. C'est le carnet des idées qui
méritent d'être reprises plus tard, écrites au moment où elles viennent pour ne pas être
reperdues.

**Rien de ce qui est dans les sections `E{n}` n'est décidé.** Ce qui est décidé part dans le
CDC, dans [`exploitation.md`](exploitation.md) ou dans le code.

Une seconde partie, [Corrections à embarquer](#corrections-à-embarquer), suit une autre
règle : ce sont des défauts constatés dans le code en production, dont la correction **est**
décidée, mais qui attendent le chantier qui les portera plutôt qu'une intervention isolée sur
une chaîne qui tourne. Numérotées `C{n}` pour qu'on ne les confonde pas avec des idées.

## Comment ajouter une idée

Une idée = une section `## E{n} — Titre`, numérotée à la suite, jamais renumérotée même si
une idée est abandonnée : les numéros servent de référence dans les conversations et les
commits.

Chaque section suit le même découpage :

- **État** — `à creuser` · `prête à chiffrer` · `en cours` · `faite` · `abandonnée` (avec la
  raison, elle vaut souvent plus que l'idée).
- **L'idée** — ce qu'on veut, en clair, du point de vue de l'usage.
- **Ce que ça change** — l'impact sur l'existant, workflow par workflow.
- **Ce qui est tranché** — les arbitrages déjà rendus, pour ne pas rouvrir un débat clos.
- **Points à trancher** — les questions ouvertes. C'est la partie la plus utile : elle évite
  de redécouvrir six mois plus tard qu'un point bloquait.

## Ordre du chantier

Les trois évolutions ci-dessous se tiennent : **E2 d'abord** (le socle), **E1 ensuite** (la
seconde porte d'entrée, qui n'a de sens qu'une fois le socle extrait), **E3 en dernier** (la
surveillance, qui a besoin des deux pour avoir quelque chose à surveiller).

Cet ordre n'est pas un confort de rédaction : E1 conçu avant E2 conduisait à recopier la
chaîne de WF1 dans WF5, donc à entretenir deux fois la même logique d'extraction.

---

## E1 — Dépôt manuel par dossier Dropbox

**État :** prête à chiffrer — dépend d'E2

### L'idée

Un dossier `Dépôt manuel` dans la Bannette, dans lequel Nicolas glisse n'importe quelle pièce
— une facture reçue hors mail, un justificatif scanné, un ticket de caisse photographié. Une
tâche planifiée regarde régulièrement ce qu'il contient et fait passer chaque pièce
**exactement par le même tube** que celles qui arrivent par mail : lecture du document,
extraction des champs, renommage, classement dans le bon dossier d'entité, écriture au
journal, puis apparition dans les récapitulatifs à Sophie.

La seule différence est la porte d'entrée. Aujourd'hui c'est un mail labellisé ; là c'est un
fichier posé dans un dossier. Tout l'aval est identique — c'est précisément ce qu'E2 rend
vrai au lieu de le supposer.

Le dossier contient un sous-dossier `Archives`, **réservé au système**. Une fois la pièce
traitée avec succès — copiée, renommée et rangée à sa destination définitive — le fichier
source y est déplacé. C'est ce déplacement qui le sort de la file : au tour suivant, la
racine du dépôt manuel ne contient plus que ce qui reste à faire.

Le fichier source n'est jamais supprimé. En cas de doute sur un classement, l'original brut,
avec son nom d'origine, reste consultable.

### L'arborescence de dépôt

```
/Bannette-Numérique/Dépôt manuel/{ENTITÉ}/A-payer/
/Bannette-Numérique/Dépôt manuel/{ENTITÉ}/Justif/
/Bannette-Numérique/Dépôt manuel/Archives/
```

`{ENTITÉ}` ∈ `AKTIMMO` | `ATTRAKTION` | `COMAKT` | `PERSO` — soit huit dossiers de dépôt,
plus `Archives`.

**Le chemin qualifie deux choses à la fois, et c'est ce qui rend l'idée viable.** Un mail
porte un label, un fichier posé dans un dossier ne porte rien : il faut bien que la
qualification vienne d'ailleurs, et le principe de la chaîne veut qu'elle vive hors de n8n.
Gmail qualifie pour les mails, l'arborescence qualifie pour les dépôts.

Le statut (`A-payer` / `Justif`) était la partie attendue. L'entité est la trouvaille : **un
ticket de caisse ne porte aucune mention d'entité**, et c'est justement sur ces pièces-là que
le modèle la devine le plus mal. En la lisant dans le chemin, on retire au modèle le champ le
plus fragile exactement là où il est le plus fragile. Sur un dépôt manuel, l'entité n'est plus
extraite du tout : elle est imposée par le dossier.

Les feuilles s'écrivent `A-payer` et `Justif`, **à l'identique des labels Gmail** — même
vocabulaire sur les deux portes, et pas d'accent ni d'espace à échapper dans les appels
Dropbox.

La destination définitive, elle, ne change pas : `Justificatifs/{ENTITÉ}/{ANNÉE}/{FOURNISSEUR}/`.

### Ce que ça change

**Un cinquième workflow, WF5, et lui seul.** Il fournit une porte d'entrée : lister le
dossier, écarter ce qui n'est pas à traiter, télécharger, appeler le socle E2, et selon le
verdict rendu, déplacer la source vers `Archives` ou la laisser en place.

**Rien à changer dans WF2 et WF3.** Ils lisent le journal, pas les mails. Une pièce déposée à
la main y apparaît comme les autres, sans une ligne de code en plus. C'est le bénéfice de la
séparation en quatre workflows.

**Rien à changer dans l'extraction** — c'est tout l'objet d'E2. Si E1 était construit sans
E2, WF5 recopierait la chaîne d'extraction de WF1 et il faudrait désormais corriger chaque
défaut deux fois.

**Le déplacement vers `Archives` tient le rôle du label `OK`.** Il obéit donc au même ordre
impératif : dépôt → lien de partage → journal → **puis seulement** déplacement de la source.
Si le workflow tombe avant le déplacement, le fichier est relu au tour suivant et c'est la
clé anti-doublon qui le rattrape — comme elle rattrape déjà les mails Sellsy envoyés en trois
exemplaires.

**WF5 doit être rattaché à WF4** comme workflow d'erreur, au même titre que les autres. Un
réglage d'une ligne, facile à oublier, sans lequel un échec est muet.

### Ce qui est tranché

**La qualification vient du chemin** — entité et statut, comme décrit ci-dessus.

**La clé de pièce vient de Dropbox.** `cle_piece` vaut aujourd'hui `{messageId}-{index}`.
Sans mail, on prend l'empreinte du contenu — et sans la calculer : l'API Dropbox renvoie déjà
un `content_hash` pour chaque fichier listé. `cle_piece = dbx:{16 premiers caractères}`. Le
même fichier redéposé est écarté **avant** le téléchargement et avant l'appel au modèle. Un
re-scan du même document donne d'autres octets et retombe sur le second filet,
`fournisseur` + `numero_facture` + `type_document`.

**Le HEIC est accepté en entrée directe.** Pas de réglage à changer sur le téléphone, pas de
nœud de conversion exotique : Dropbox sait rendre un HEIC en JPEG (`/2/files/get_thumbnail_v2`,
jusqu'à 2048×1536 — déjà plus que ce que le modèle exploite), même credential et même API que
le reste de la chaîne. **La conversion appartient au socle**, pas à cette porte, et E2 dit
pourquoi. C'est le JPEG converti qui part à la destination définitive ; le HEIC d'origine reste
dans `Archives` avec son nom.

**Trois colonnes du journal perdent leur sens** : `message_id`, `expediteur`, `sujet_mail`.
Elles restent vides, et deux colonnes les remplacent plutôt que d'être détournées :
`source` (`mail` | `depot-manuel`) et `fichier_origine` (le nom d'origine du fichier, vide
pour les mails). La migration de la Data Table porte aussi `fournisseur_lu` (voir
[C1](#c1--fournisseur_lu-est-calculé-puis-jeté)) : une seule opération, 16 → 19 colonnes.

**Une pièce en échec s'écrit au journal** avec `statut = A-verifier` et sa `cle_piece`. Sans
ça, un fichier illisible laissé en place serait renvoyé au modèle **à chaque tour, pour
toujours** : le déplacement automatique tenait ce rôle, et il n'a plus lieu (voir juste
en dessous). Avec la ligne au journal, le dédoublonnage le reconnaît avant l'appel au modèle
et passe, à coût nul. Il sort de la file le jour où le fichier quitte le dossier.

**Un échec laisse le fichier à la racine de son dossier de dépôt**, sous les yeux, plutôt que
dans un dossier système. Décidé ainsi : la visibilité primait. C'est la ligne de journal
ci-dessus qui rend ce choix tenable.

**Le numéro des pièces sans numéro est fabriqué.** Un ticket de caisse n'a pas de numéro de
facture, et le test de complétude l'exige — tous les tickets partiraient en `A-verifier`,
c'est-à-dire le cas d'usage même de cette évolution, mort au dernier nœud. Quand le type est
`recu` ou `autre` et qu'aucun numéro n'est lisible, `numero_facture` vaut `T` suivi des six
premiers caractères de l'empreinte : `T3f9a2c`. Unique, stable — le même ticket redéposé
retombe sur le même nom — et la clé anti-doublon continue de fonctionner. La complétude
devient `fournisseur && date_facture && entite && (numero_facture || type ∈ {recu, autre})`.

**La fréquence est horaire, de 8h à 20h.** WF1 tourne une fois par jour, ce qui convient à
des mails ; pour un dépôt manuel, on pose un fichier et on aimerait le voir rangé dans
l'heure. Un tour à vide ne coûte qu'un appel de liste Dropbox et zéro appel au modèle : la
dépense suit le nombre de pièces, pas le nombre de tours.

**Un fichier de moins de cinq minutes est ignoré.** Dropbox ne publie un fichier qu'une fois
l'envoi validé, donc le risque de lire un fichier tronqué est plus faible qu'il n'y paraît ;
le garde-fou sur `server_modified` ne coûte rien et lève le doute.

**Les archives sont préfixées par la date de traitement** — `2026-09-12_scan.pdf` — et
`autorename` de Dropbox sert de filet. Deux fichiers `scan.pdf` déposés à deux semaines
d'écart ne se marchent plus dessus.

### Points à trancher

**Vérifier que le rendu HEIC de Dropbox répond vraiment.** Toute la solution du HEIC tient à
cet appel. Un test sur une vraie photo, avant de construire quoi que ce soit.

**Un fichier posé à la racine de `Dépôt manuel`**, hors des huit dossiers qualifiants, n'est
dans aucune file et serait ignoré en silence, indéfiniment. C'est [E3](#e3--relevé-des-pièces-en-souffrance)
qui doit le signaler ; sans E3, prévoir au minimum une alerte dans WF5.

**Le volume et son coût.** Chaque pièce déposée est un appel au modèle. Le rythme horaire ne
coûte rien à vide, mais rien n'empêche de déposer trente fichiers d'un coup : vérifier qu'un
plafond par exécution est nécessaire, comme la limite de 20 mails de WF1.

---

## E2 — Socle commun de traitement d'une pièce

**État :** prête à chiffrer

### L'idée

Aujourd'hui la compétence « lire une pièce et la ranger » vit à l'intérieur de WF1, entre son
nœud Gmail et son nœud de labellisation. Toute seconde porte d'entrée devrait la recopier.

On l'extrait dans un sous-workflow appelé par les deux : **un seul tronc, deux portes.**

```
WF1 (porte mail)              WF5 (porte Dropbox)
Gmail → éclater                lister → filtrer → télécharger
        │                              │
        └────── WF0 — Traiter une pièce ──────┘
    aiguiller et lire (PDF texte | PDF scanné | image | HEIC)
    → extraire → normaliser → complétude → dédoublonner
    → déposer → lien de partage → journal → rendre un verdict
        │                              │
    poser le label               déplacer vers Archives
```

### La règle qui décide de la frontière

> **Une porte ne regarde jamais le contenu ni le format d'une pièce. Elle produit des
> octets, un nom et une qualification. Tout ce qui consiste à ouvrir le fichier appartient
> au socle.**

Cette phrase vaut mieux qu'une liste de nœuds : elle tranche aussi les cas qu'on n'a pas
encore rencontrés. **La lecture du PDF texte est dans le socle**, au même titre que la lecture
d'une image — c'est une façon d'ouvrir un fichier parmi d'autres, pas un privilège de la porte
mail. Un PDF avec couche texte déposé à la main suit donc exactement le même chemin que le
même PDF reçu par mail, sans que WF5 ait à savoir ce qu'est une couche texte.

Appliquée aux 26 nœuds de WF1, elle donne un partage net :

| Reste à la porte | Part au socle |
|---|---|
| Le déclencheur | Lire le texte du PDF |
| Lire les mails labellisés | Texte lisible ? |
| Éclater les pièces jointes *(sans son filtre de format)* | Extraire les champs · le nœud modèle |
| La boucle par pièce | Normaliser · Complétude |
| Marquer OK | Déjà au journal ? · Doublon ? |
| Marquer OK (doublon) | Lister les dossiers · Choisir le dossier |
| Marquer A-verifier | Déposer · Lien · Consolider |
| | Préparer et déposer dans A-classer |
| | Écrire au journal |

**Trois nœuds Gmail : c'est tout ce qui est réellement propre à la porte mail.** Les vingt
autres sont du socle. Ce déséquilibre est la meilleure justification d'E2 — il dit à quel
point recopier la chaîne dans WF5 aurait été coûteux.

### Le contrat

**En entrée** : le fichier binaire, son nom d'origine et son extension réelle, `source`
(`mail` | `depot-manuel`), `statut` (`A-payer` | `Justif`), `entite_imposee` (vide pour un
mail, remplie pour un dépôt), et `reference` (l'identifiant de mail et l'index de pièce, ou
l'empreinte Dropbox).

Rien d'autre. En particulier, **la porte ne dit pas ce que le fichier contient** : elle donne
son extension parce qu'elle la lit dans un nom, pas parce qu'elle a ouvert quoi que ce soit.

**En sortie** : un verdict à trois valeurs — `ok`, `doublon`, `a-verifier` — et pour une pièce
acceptée sa `cle_piece`, son chemin et son lien de partage. Un `a-verifier` porte en plus son
motif, en clair.

Trois valeurs et non deux, parce que WF1 distingue déjà les trois cas : `Marquer OK`,
`Marquer OK (doublon)` et `Marquer A-verifier`. Fondre le doublon dans le succès ferait perdre
une distinction visible aujourd'hui en exécution. Chaque porte traduit ensuite les trois dans
son vocabulaire : un label pour la porte mail, un déplacement pour la porte Dropbox — où
`ok` et `doublon` mènent tous deux à `Archives`, la pièce étant traitée dans les deux cas.

**Une pièce, un appel.** Le socle traite une pièce et rend un verdict ; la boucle reste chez
l'appelant. Le contrat est trivial à lire, chaque pièce apparaît comme une exécution distincte
dans l'onglet *Executions*, et l'échec de l'une n'emporte pas les autres.

### L'aiguillage : la compétence de lecture, en un seul endroit

C'est le contenu neuf d'E2, et ce qui justifie de faire l'extraction maintenant plutôt que
plus tard.

| Ce qui entre | Ce que fait le socle |
|---|---|
| `.pdf` avec couche texte | extraction texte, puis extraction des champs — la chaîne actuelle, inchangée |
| `.pdf` sans couche texte | le PDF part entier au modèle, en bloc `document` |
| `.jpg` `.jpeg` `.png` `.webp` | bloc `image` |
| `.heic` | conversion en JPEG (voir ci-dessous), puis bloc `image` |
| tout le reste | `a-verifier`, avec le motif écrit noir sur blanc |

**Le nœud `Texte lisible ?` cesse d'être une porte de sortie vers l'échec pour devenir un
aiguillage.** Aujourd'hui `faux` mène à `A-classer` ; demain `faux` mène à la branche
document. C'est un fil à déplacer, pas un nœud à écrire — et c'est ce qui fait disparaître la
cause de rejet la plus fréquente de la chaîne.

Le PDF scanné part **entier** : le bloc `document` de l'API accepte un PDF en base64 jusqu'à
32 Mo et 100 pages sur un modèle à 200K de contexte. **Il n'y a aucune rastérisation page par
page à bricoler dans n8n** — c'était le coût redouté de cette évolution, il n'existe pas.

**La conversion du HEIC appartient au socle, pas à la porte.** L'API n'accepte pas ce format ;
Dropbox sait le rendre en JPEG (`/2/files/get_thumbnail_v2`), mais seulement pour un fichier
qui est déjà chez lui — vrai pour un dépôt manuel, faux pour un HEIC reçu par mail. Laisser la
conversion à la porte Dropbox donnerait au socle deux comportements selon l'appelant, ce que la
règle interdit. Le socle dépose donc lui-même tout HEIC dans un dossier de travail
(`/Bannette-Numérique/_conversion/`), demande le rendu, puis efface. Trois appels de plus,
uniquement pour du HEIC, et un seul comportement quelle que soit la porte.

C'est le JPEG converti qui part à la destination définitive : ces fichiers sont destinés à une
comptable, et un HEIC s'ouvre mal hors de l'écosystème Apple. Le nom de fichier porte donc
l'extension d'arrivée, pendant que `fichier_origine` garde le nom d'origine avec la sienne.

**Le contrôle des formats acceptés est dans le socle**, à l'entrée de ce tableau. Il y remplace
le filtre PDF que la porte mail applique aujourd'hui (voir
[C3](#c3--une-pièce-jointe-non-pdf-est-ignorée-en-silence)) : juger d'un format, c'est déjà
lire. Tant que ce filtre reste dans la porte, celle-ci détient un morceau de la compétence, et
la porte Dropbox doit le réimplémenter.

**Le montant TTC est extrait dans tous les cas**, y compris sur un ticket photographié : la
consigne d'extraction est celle du socle, une seule, partagée. Sur un ticket, le « net à
payer » est le total payé. Le montant reste un confort qui n'entre pas dans le test de
complétude — un montant illisible ne fait pas partir la pièce en `A-verifier`.

Le nœud d'extraction actuel ne sait porter ni image ni document : les branches non-texte
passent par un appel direct à l'API. C'est le seul nœud réellement nouveau du socle.

### Ce que ça change

**WF1 en production est restructuré, pas seulement complété.** C'est le coût et le risque de
cette évolution, et il faut le dire tel quel. Il se maîtrise par l'ordre : construire le
socle et l'éprouver par WF5 — qui n'est encore branché sur rien de critique — puis seulement
remplacer le milieu de WF1 par l'appel au socle. Les nœuds ne sont pas réécrits, ils sont
déplacés : ils sont déjà éprouvés.

**Deux bénéfices immédiats pour WF1, sans rapport avec le dépôt manuel :**

- une facture scannée arrêterait de partir en `A-verifier` — c'est aujourd'hui l'une des deux
  seules causes de rejet, et la plus fréquente ;
- une pièce jointe qui n'est pas un PDF arrêterait d'être **ignorée en silence** (voir
  [C3](#c3--une-pièce-jointe-non-pdf-est-ignorée-en-silence)). Un justificatif envoyé en JPEG
  n'existe pas pour la chaîne aujourd'hui.

**Les quatre corrections en attente s'appliquent ici**, dans le même geste : C1, C2, C3, C4.
C'est le chantier qui les porte.

### Points à trancher

**Le passage du binaire entre workflows.** À vérifier sur l'instance avant de s'engager : le
fichier doit traverser l'appel au sous-workflow sans être rematérialisé par un détour
Dropbox.

**Le modèle de la branche non-texte.** La branche texte tourne sur un petit modèle, largement
suffisant pour lire quatre champs dans une facture bien formée. Lire un ticket froissé est
une autre affaire. Rien n'oblige les deux branches à partager le même modèle : commencer avec
celui en place, mesurer sur de vrais tickets, et ne monter en gamme que la branche image si
l'entité ou le montant s'avèrent fragiles. L'entité, sur un dépôt manuel, est de toute façon
imposée par le dossier — c'est un champ difficile de moins.

**Le coût par pièce.** Une image coûte plus qu'un extrait de texte. À mesurer sur une
vingtaine de pièces réelles avant de fixer la fréquence d'E1.

**Faut-il croire l'extension ?** Le contrat fait passer l'extension parce que la porte la lit
dans un nom — mais un nom ment : un `.pdf` qui est en réalité un JPEG existe, et une photo
renommée à la main aussi. La règle prise au sérieux voudrait que le socle ne s'en remette pas
davantage au nom que la porte : reconnaître le format aux premiers octets du fichier (`%PDF`,
la signature JPEG, celle du PNG) et ne garder l'extension que comme indice. Quelques lignes
dans le nœud d'aiguillage, à décider au moment de l'écrire.

---

## E3 — Relevé des pièces en souffrance

**État :** prête à chiffrer

### L'idée

Un mail à `nicolas@attraktion.fr` qui dit ce qui attend une main humaine. Aujourd'hui rien ne
le dit : WF4 alerte quand un workflow **tombe**, ce qui n'est pas la même chose qu'une pièce
qui a suivi son chemin normalement et s'est rangée dans `A-classer`. Une pièce peut y dormir
des semaines sans que personne ne le sache.

**Pas de mail quand il n'y a rien.** Le silence signifie que tout est propre — c'est ce qui
rend le mail lisible le jour où il arrive.

### Ce qu'il regarde

Trois sources, parce que chacune attrape ce que les autres manquent :

- **`Justificatifs/A-classer/`** — les pièces déposées sans avoir passé le test de
  complétude. C'est l'état réel sur le disque, la source la plus fiable.
- **Le journal, `statut = A-verifier`** — ce que la chaîne a explicitement mis de côté. Ne
  devient utile qu'après [C4](#c4--une-pièce-partie-en-a-classer-ne-laisse-aucune-trace-au-journal)
  et l'écriture des échecs prévue par E1.
- **Les dossiers de dépôt manuel, `Dépôt manuel` comprise** — un fichier qui traîne dans un
  dossier qualifiant est un échec resté sur place ; un fichier posé **à la racine** n'est dans
  aucune file et serait ignoré à jamais. Ce dernier cas mérite d'être nommé à part dans le
  mail : ce n'est pas une pièce en échec, c'est une pièce que personne ne regarde.

### Ce que ça change

**Un sixième workflow, WF6, qui ne fait que lire.** Il ne déplace rien, ne corrige rien,
n'écrit pas au journal. Un relevé qui modifie l'état de ce qu'il relève est un relevé auquel
on ne peut plus se fier.

Rien à changer ailleurs.

### Points à trancher

**La fréquence et l'heure.** Quotidien tôt le matin est le réflexe, mais un relevé quotidien
d'un dossier qui bouge peu devient vite du bruit qu'on n'ouvre plus. Hebdomadaire, avec une
exception quotidienne au-delà d'un certain nombre de pièces, est peut-être plus juste.

**Un seuil d'ancienneté.** Une pièce arrivée en `A-classer` ce matin n'a pas besoin d'être
signalée. Au-delà de quelques jours, si.

**L'objet du mail.** Garder le crochet `[Admin Nico]` qui sert au tri, et y mettre le nombre
de pièces en attente pour que le mail se lise sans être ouvert.

---

# Corrections à embarquer

Quatre défauts constatés dans WF1 en production. **Leur correction est décidée** ; elle
attend [E2](#e2--socle-commun-de-traitement-dune-pièce), qui touche de toute façon aux nœuds
concernés. Intervenir maintenant, séparément, sur une chaîne qui tourne, coûterait deux fois
le même risque pour le même résultat.

Aucun des quatre ne perd de pièce ni n'écrit de fausse donnée. C'est ce qui autorise à
attendre.

## C1 — `fournisseur_lu` est calculé puis jeté

Le nœud *Choisir le dossier* calcule bien `fournisseur_lu`, le nom que le modèle avait
réellement répondu avant rapprochement avec les dossiers existants. Mais *Écrire au journal*
ne le mappe pas, et la Data Table n'a pas cette colonne. **La valeur est produite puis
perdue à chaque facture.**

`exploitation.md` affirmait le contraire — que la colonne permettait « de voir la dérive sans
la subir ». Elle ne le permet pas : on ne voit pas la dérive des noms de fournisseurs.

**Correction :** ajouter la colonne à la Data Table et la mapper. Elle voyage avec les deux
colonnes qu'E1 ajoute (`source`, `fichier_origine`) — une seule migration, 16 → 19 colonnes.

## C2 — L'extension `.pdf` est en dur dans le nom de fichier

Le nœud *Normaliser* construit `nom_fichier` en collant `.pdf`. Sans conséquence aujourd'hui,
puisque seuls des PDF entrent dans la chaîne. Dès qu'une image est acceptée (E2), **un JPEG se
retrouverait déposé sous un nom en `.pdf`** — illisible d'un double-clic.

**Correction :** prendre l'extension réelle de la pièce, transmise par le contrat d'entrée du
socle.

## C3 — Une pièce jointe non-PDF est ignorée en silence

Le nœud *Éclater les pièces jointes* ne garde que ce qui est `application/pdf` ou finit par
`.pdf`. Tout le reste est écarté sans trace : ni label, ni journal, ni alerte. **Un
justificatif envoyé en JPEG n'existe pas pour la chaîne** — et personne ne l'apprend.

**Ce n'est pas tout à fait un bug, c'est une frontière mal placée** — et c'est ce qui le rend
intéressant. Juger d'un format, c'est déjà lire : ce filtre appartient au socle, pas à la
porte. Tant qu'il reste dans WF1, la porte mail détient un morceau de la compétence de lecture
et la porte Dropbox devra le réimplémenter. Le silence n'en est que le symptôme.

**Correction :** déplacer le contrôle dans le socle, sous la forme d'une liste de formats
acceptés (E2 donne un chemin de lecture aux images), et rendre un `a-verifier` motivé pour ce
qui reste réellement inexploitable — au lieu du silence.

## C4 — Une pièce partie en `A-classer` ne laisse aucune trace au journal

La branche d'échec de WF1 dépose dans `A-classer` et pose le label `A-verifier`, mais
n'écrit pas au journal : l'écriture n'existe que sur le chemin du succès. **Le journal ne
connaît donc pas les pièces en souffrance.** Le label Gmail est la seule trace, et il n'en
reste rien pour une pièce qui ne vient pas d'un mail.

**Correction :** écrire la ligne avec `statut = A-verifier`. C'est aussi ce qui rend possible
le dédoublonnage des échecs prévu par E1 et le relevé par le journal prévu par E3.
