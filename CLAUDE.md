# Repères pour une session de travail

Chaîne n8n qui va du mail reçu à la transmission à la comptable. **Elle est en production
depuis le 10 septembre 2026** : quatre workflows publiés tournent à leurs horaires sur de
vraies factures, et écrivent dans un vrai Dropbox.

## À lire avant d'agir

- [`README.md`](README.md) — le principe, l'arborescence, le modèle de données
- [`docs/exploitation.md`](docs/exploitation.md) — identifiants des workflows, de la Data Table,
  des credentials, et où changer quoi
- [`docs/evolutions.md`](docs/evolutions.md) — les évolutions instruites (E1, E2, E3) et les
  corrections décidées (C1 à C4)
- [`docs/chantier.md`](docs/chantier.md) — par où commencer pour les construire
- [`docs/recette.md`](docs/recette.md) — les défauts déjà trouvés sur de vraies factures

**Le code fait foi, pas la documentation.** Lire les workflows dans n8n avant d'affirmer
comment ils marchent : quatre écarts entre la doc et le réel ont déjà été trouvés ainsi. En
signaler un nouveau plutôt que de le contourner en silence.

## Ce qui demande un accord explicite

Publier ou modifier un workflow · toucher à WF1 · supprimer quoi que ce soit sur Dropbox ·
changer un horaire · envoyer un mail. Lire n8n, Dropbox et Gmail est libre.

## Ce qui ne se casse pas

- **L'ordre du dépôt** : dépôt → lien de partage → journal → **puis seulement** marquage.
  Aucune ligne du journal ne doit pouvoir pointer vers un fichier absent.
- **Le dédoublonnage à deux étages** : l'empreinte du fichier en haut,
  `fournisseur` + `numero_facture` + `type_document` en bas.
- **La qualification vit hors de n8n** : un label Gmail, un chemin Dropbox. Jamais une règle de
  détection dans un workflow.
- **Une pièce non traitée reste à sa porte** : dans Gmail, ou dans son dossier de dépôt.
- **L'entité peut vivre dans un chemin, pas le statut** : une entité ne change jamais.
- **Le montant n'entre pas dans le test de complétude** : c'est un confort.

Les sections « Ce qui est tranché » d'`evolutions.md` portent des arbitrages rendus avec
Nicolas. Elles ne se rouvrent pas ; seules les sections « Points à trancher » sont ouvertes.

## Git

`main` porte la vérité. Chaque chantier vit sur une branche de travail éphémère, fusionnée
dans `main` puis supprimée. Les versions livrées sont marquées par des étiquettes, jamais par
des branches.

## Écriture

Documentation en français, au présent, sans jargon inutile. Une phrase dit ce qui est fait
**et pourquoi** : c'est le pourquoi qui sert dans six mois. Les messages de commit suivent la
même règle et s'écrivent sans accent.
