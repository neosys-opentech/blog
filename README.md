# Neosys - Le blog

Bienvenue dans le journal de bord de Neosys, si vous voulez contribuer au blog, veuillez suivre le guide de démarrage.

## Guide de démarrage

Le blog utilise le service d'hébergement des pages statiques GitHub Pages. Ce dernier est compatible avec Jekyll.
Jekyll est un générateur de pages statiques orienté blog. Il est développé avec Ruby.

Pour configurer votre environnement de dév local:

+ Installez Ruby et son SDK en suivant les instructions sur le [lien](https://jekyllrb.com/docs/installation/windows/).
+ Clonez le repository.
+ Placez-vous à la racine du projet et jouer l'instruction `bundler update` en ligne de commande.
+ Démarrez le blog en local avec la ligne de commande `bundler exec jekyll serve -- force_polling`
+ Ouvrez le navigateur et allez sur le lien http://localhost:4000/blog/ (surtout n'oubliez pas le dernier /)

## Pour ajouter un nouveau billet blog

Si c'est votre 1er billet, il faudra d'abord créer un nouveau blogueur:

+ Ajoutez un fichier markdown (extension .md) dans le dossier `_authors`.
+ Remplissez toutes vos informations, inspirez-vous de ce qui est déjà existant.
+ Si tout est OK, vous allez voir que vous êtes ajouté à la liste des blogueurs sur le lien http://localhost:4000/blog/staff

Pour rédiger le billet:

+ créez un nouveau fichier markdown dans le dossier `_posts`.
+ Taper votre billet en langage markdown (pas de HTML SVP dans le code), inspirez-vous des exemples déjà réalisés.
+ Si tout est OK, votre billet s'ajoutera au menu gauche des derniers billets et dans votre espace blogueurs http://localhost:4000/blog/authors/{username}.

## Guidelines

+ Gardez à l'esprit que le blog doit rester mobile friendly.
+ Les code snippets doivent être créés sur github gist.
+ Les images des billets doivent être annotées avec `{:.ns-post-img-fluid}`.
