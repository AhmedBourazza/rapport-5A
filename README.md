# Rapport de PFE — Ahmed BOURAZZA (ENSIM × VESI)

## Compilation sur Overleaf
1. Compresser ce dossier en `.zip` et l'importer sur Overleaf (**New Project → Upload Project**).
2. Menu **Compiler → pdfLaTeX**.
3. Bibliographie : Biber (Overleaf le lance automatiquement).

> Remarque : le fichier utilise `newtx` (Times New Roman) et `babel` français, tous deux présents sur Overleaf. Une compilation locale peut échouer si ces paquets ne sont pas installés — ce n'est pas un problème du projet.

## Structure
- `main.tex` — fichier maître (ne pas surcharger, il ne fait qu'inclure).
- `config/preambule.tex` — police Times, marges 2 cm, styles, packages.
- `frontmatter/` — page de garde, résumés, remerciements.
- `chapitres/` — un fichier par chapitre (on rédige l'un après l'autre).
- `backmatter/` — `references.bib`, glossaire, annexes.
- `images/` — y déposer logos et figures. Logos attendus : `logo_ensim.png`, `logo_vinci.png`.
- `code/` — extraits de code à insérer via `listings` au chapitre 4.

## À faire (rappels dans les fichiers, balisés `>>> ... <<<`)
- Définir le **titre** (page de garde).
- Ajouter les **logos** (décommenter dans `page_de_garde.tex`).
- Fournir/valider les **chiffres groupe** (chapitre 1) et les **références** (chapitre 3).
- Rédiger chaque chapitre après validation, dans l'ordre convenu.
