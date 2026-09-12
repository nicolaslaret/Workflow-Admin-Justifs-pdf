# Démarrer le chantier E2 → E1 → E3

Ce fichier s'adresse à une session neuve, sans mémoire de la conversation qui a produit
[`evolutions.md`](evolutions.md). Il dit par où commencer, dans quel ordre, et ce qu'il ne
faut pas défaire.

## Le prompt d'amorçage

```
Nous reprenons le projet Workflow-Admin-Justifs-pdf.

Lis docs/chantier.md et applique-le à partir de l'étape 0.

Si docs/evolutions.md ne contient pas les sections E1, E2, E3, c'est que la
branche claude/repo-evolutions-xqv0d3 n'est pas fusionnée : va la chercher.

Ne publie aucun workflow et ne modifie WF1 sans mon accord explicite.
Commence par l'étape 1 et arrête-toi pour me montrer les résultats.
```

Le dernier paragraphe compte autant que le premier : la chaîne est **en production**, quatre
workflows publiés tournent à leurs horaires sur de vraies factures.

---

## Étape 0 — Lire, dans cet ordre

| Fichier | Ce qu'on y cherche |
|---|---|
| [`../README.md`](../README.md) | le principe (Gmail qualifie, n8n exécute), l'arborescence, le modèle de données |
| [`exploitation.md`](exploitation.md) | les identifiants des workflows, de la Data Table et des credentials |
| [`evolutions.md`](evolutions.md) | **la spec du chantier** : E2, E1, E3, puis C1 à C4 |
| [`recette.md`](recette.md) | les huit défauts trouvés sur de vraies factures — pour ne pas les réintroduire |

Puis lire les workflows eux-mêmes dans n8n. **Le code fait foi, pas la documentation** : c'est
en lisant WF1 nœud par nœud que trois écarts entre la doc et le réel ont été trouvés.

**Ce qui est clos ne se rouvre pas.** Les sections « Ce qui est tranché » d'E1 et d'E2 portent
des arbitrages rendus avec Nicolas, chacun motivé. Seules les sections « Points à trancher »
sont ouvertes. Reproposer un choix déjà fait fait perdre du temps aux deux.

---

## Étape 1 — Deux vérifications avant d'écrire quoi que ce soit

Des morceaux entiers de la spec en dépendent. Vingt minutes, avant la première ligne.

**1 · Dropbox rend-il un HEIC en JPEG ?** Poser une vraie photo `.heic` sur Dropbox, appeler
`/2/files/get_thumbnail_v2` dessus, vérifier qu'un JPEG lisible revient.
→ Si non : toute l'entrée HEIC d'E1 est à repenser. Le dire avant de construire, pas après.

**2 · Le binaire traverse-t-il l'appel à un sous-workflow ?** Un workflow d'essai qui reçoit un
PDF et le renvoie, appelé par un autre. Vérifier que le fichier arrive entier, sans détour par
Dropbox.
→ Si non : l'architecture d'E2 change de forme.

S'arrêter ici et montrer les résultats.

---

## Étape 2 — La Data Table, puis le socle

**La migration d'abord** : trois colonnes à ajouter à `journal_factures`, 16 → 19 —
`source`, `fichier_origine`, `fournisseur_lu` ([C1](evolutions.md#corrections-à-embarquer)).
Sans conséquence pour les workflows en production, qui ne les remplissent simplement pas.

**Puis WF0 — Traiter une pièce**, en suivant E2 : le contrat d'entrée et de sortie,
l'aiguillage de lecture, les trois verdicts. **Laissé non publié.**

Les vingt nœuds qui viennent de WF1 sont **recopiés tels quels**, pas réécrits : ils sont
éprouvés sur de vraies factures. Ce qui est neuf, c'est l'aiguillage, la branche non-texte, et
les corrections C2, C3, C4.

---

## Étape 3 — La porte Dropbox

Créer à la main les neuf dossiers de dépôt (huit `{ENTITÉ}/{A-payer|Justif}` plus `Archives`),
et le dossier de travail `_conversion`.

Puis **WF5**, selon E1 : lister, filtrer, pré-contrôler l'empreinte au journal, télécharger,
appeler le socle, déplacer ou laisser. Rattaché à WF4 comme workflow d'erreur. Non publié.

C'est ici qu'on éprouve le socle, sur une porte qui n'est encore branchée sur rien de
critique — jamais sur WF1.

---

## Étape 4 — La recette, sur de vraies pièces

Sept cas, et pas un de moins. Consigner ce qu'ils révèlent dans [`recette.md`](recette.md).

1. Un PDF avec couche texte → doit passer comme avant
2. Un PDF scanné sans couche texte → doit passer par la branche document
3. Un JPEG → branche image
4. Un HEIC → converti puis lu, JPEG à destination, original en `Archives`
5. Un ticket de caisse sans numéro → numéro fabriqué, pièce acceptée
6. Une pièce déjà traitée, redéposée → écartée avant l'appel au modèle
7. Une pièce illisible → reste en place, ligne au journal motivée, **et rien au tour suivant**

Le septième est le plus important : c'est lui qui prouve qu'un échec ne coûte rien en boucle.

---

## Étape 5 — Basculer WF1

**L'étape à risque, et la seule.** Remplacer le milieu de WF1 par l'appel au socle : le
déclencheur, la lecture Gmail, l'éclatement des pièces jointes et les trois nœuds de
labellisation restent ; tout le reste part.

Ne s'y engager qu'une fois l'étape 4 verte. Le filtre PDF de `Éclater les pièces jointes` part
au socle avec le reste ([C3](evolutions.md#corrections-à-embarquer)).

---

## Étape 6 — Le relevé, puis le ménage

**WF6** selon E3. Ses quatre sources, la déduplication sur `cle_piece`, le silence quand il n'y
a rien à dire.

Puis seulement : vider et supprimer `Justificatifs/A-classer/`, et retirer ses mentions du
README et d'`exploitation.md`. Le premier relevé aura listé son contenu par le label Gmail.

Mettre enfin à jour l'état d'E1, E2, E3 en `faite`, et documenter les nouveaux réglages dans
`exploitation.md` — horaires, dossiers, seuils.

---

## Les règles qu'on ne casse pas

**L'ordre du dépôt** : dépôt → lien de partage → journal → **puis seulement** marquage (label
ou déplacement). C'est ce qui garantit qu'aucune ligne du journal ne pointe vers un fichier
absent, et qu'un échec en cours de route se rattrape au tour suivant.

**Le dédoublonnage à deux étages** : l'empreinte en haut, `fournisseur` + `numero_facture` +
`type_document` en bas. Les deux, pas l'un ou l'autre.

**La qualification vit hors de n8n.** Un label Gmail, un chemin Dropbox. Jamais une règle de
détection dans un workflow : un nouveau fournisseur se traite par une règle Gmail.

**Une pièce non traitée reste à sa porte.** Elle ne se recopie nulle part.

**L'entité peut vivre dans un chemin, pas le statut.** Une entité ne change jamais ; un
`A-payer` devient payé.

**Le montant n'entre pas dans le test de complétude.** C'est un confort.

---

## Ce qui se décide seul, ce qui se demande

**Seul** : les noms de nœuds, la disposition, la façon d'écrire un bout de code, l'ordre des
tests, tout ce que les sections « Ce qui est tranché » ont déjà réglé.

**On demande** : publier un workflow, modifier WF1, supprimer quoi que ce soit sur Dropbox,
changer un horaire, écrire à Sophie. Et tout point ouvert d'E1, E2 ou E3 — ils sont ouverts
parce qu'ils demandent un arbitrage, pas parce qu'ils sont difficiles.

**On signale sans attendre** un écart entre la documentation et le code réel. Il y en a eu
quatre ; il en reste peut-être.
