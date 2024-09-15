---
tags: git config
author: Nicolas Mugnier
---

:bulb: Exemple de personnalisation basique mais efficace :blush:

Se placer dans le répertoire racine du user

```bash
cd ~
```

Créer le fichier `.gitconfig`

```bash
touch .gitconfig
```

Editer le fichier à l'aide de vim

```bash
vim .gitconfig
```

Ajouter le contenu suivant

```bash
[user]
	name= Lastname, Firstname
	email = firstname.lastname@github.com
[pull]
	rebase = true
[alias]
	show-files = diff-tree --no-commit-id --name-only -r
	co = checkout
	br = branch
	ci = commit
	st = status -sb
	lg = log --pretty=format:'%Cred%h%Creset -%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset'
[credential]
	helper = store
[core]
	editor = vim
```

Sauvegarder les modifications

```bash
:wq
```

Exemples

![git-lg-sample](/assets/img/git-lg-sample.png)

![git-st-sample](/assets/img/git-st-sample.png)
