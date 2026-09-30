---
tags: git config sign pgp ssh verified
author: Nicolas Mugnier
description: TODO
image: TODO
locale: fr_FR
---

## Pourquoi ?

au niveau de la configuration de git dans le .gitconfig, il est possible de définit nom, prénom, email. Du coup il est possible de renseigner les informations d'une autre personne.

Afin de s'assurer de l'identité de l'auteur d'un commit, il est possible de configurer des couples de clés privées/publiques qui permettent à github de vérifier que la personne qui commit est bien la bonne.

## Création des clés

### PGP

Création des clés

https://docs.github.com/en/authentication/managing-commit-signature-verification/generating-a-new-gpg-key

```bash
gpg --full-generate-key
gpg --list-secret-keys --keyid-format=long
```

TODO passphrase

Configuration de git

```bash
git config --global user.signingkey <key-id>
```

Ajout de la clé publique dans Github

TODO : ajouter un print de l'UI

### SSH

Création des clés

https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent

TODO : passphrase

```bash
ssh-keygen -t ed25519 -C "<email>"
```

Ajout des clé au registre

```bash
ssh-add /Users/<username>/.ssh/id_ed25519
```

Configuration de git

```bash
git config --global gpg.format ssh
git config --global user.signingkey /Users/<username>/.ssh/id_ed25519.pub
``` 

Ajout de la clé publique dans Github

TODO : ajouter un print de l'UI

## Utilisation dans Codespace

TODO : Ajouter un print de l'UI

## Copie d'une clé PGP

Récupérer le hash

```bash
gpg --list-keys
```

Exporter la clé privée dans un fichier

```bash
gpg --export-secret-keys --armor <key-id> > private-key.asc
```

Copier la clé (scp)

Importer la clé sur le nouvel environnement

```bash
gpg --import private-key.asc
```

Configurer git pour utiliser la clé

```bash
git config --global user.signingkey <key-id>
```

## Usage

```bash
git commit -S -am 'feat(feature): message'
git commit -S --amend
git rebase -S -i HEAD~2
git tag -s v0.0.1
```

```bash
git config --global commit.gpgsign true
git config --global tag.gpgSign true
```

## Conserver la signature du commit lors du merge

Les options disponibles depuis l'UI de Github ne permettent pas de conserver le commit signer lors du merge dans le cas où on shouaite conserver un historique linéaire sans commit de merge.
Pour ce faire, il faut merger en cli :

```bash
git checkout main
git pull origin main
git merge --ff-only <head>
git push -u origin main
```

## Documentation

https://docs.github.com/en/authentication/managing-commit-signature-verification/signing-commits
