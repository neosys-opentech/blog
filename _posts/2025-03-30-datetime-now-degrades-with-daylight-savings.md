---
layout: post
author: pedrobsaila
date: 2025-03-30 09:00:00
title: Les merveilles du passage à l'heure d'été dans .NET
front_image: /assets/images/posts/2025-03-30-datetime-now-degrades-with-daylight-savings/logo.png
excerpt_separator: <!--more-->
---

Rappel annuel : l'heure d'été est aussi un casse noisette pour les développeurs .NET. En effet, les performances de la méthode `DateTime.Now` se dégradent à chaque passage à l'heure d'été et s'améliorent au passage à l'heure d'hiver.

<!--more-->

## Micro-benchmark de DateTime.Now

Allons voir le résultat du [micro benchmark](https://github.com/dotnet/performance/blob/main/src/benchmarks/micro/libraries/System.Runtime/Perf.DateTime.cs) de la fameuse méthode `DateTime.Now` sur Windows/Linux x64. Le micro benchmark est réalisé avec la librairie open source [benchmark-dotnet](https://github.com/dotnet/BenchmarkDotNet) et voilà un extrait de code de ce dernier :

{% gist 82feb08dc4441ce54ac2f6a3e9933016 %}
```

Pour info, la machine exécutant le test est configurée avec la culture des états unis et l'heure d'été y débute le 9 Mars et se termine le 2 Novembre. En France, l'heure d'été débute aujourd'hui le 30 mars et se termine au dernier dimanche d'Octobre.

## Windows x64

{:.ns-post-img-fluid}
![alt Dégradation DateTime.New sur Windows]({{ '/assets/images/posts/2025-03-30-datetime-now-degrades-with-daylight-savings/datetime_now_windows.png' | relative_url }}){:.mx-auto}{:.d-block}

La dégradation est de l'ordre de 25%

## Linux x64

{:.ns-post-img-fluid}
![alt Dégradation DateTime.New sur Linux]({{ '/assets/images/posts/2025-03-30-datetime-now-degrades-with-daylight-savings/datetime_now_linux.png' | relative_url }}){:.mx-auto}{:.d-block}

La dégradation est de l'ordre de 20%

## Explication

Quand l'heure d'été arrive, la méthode `DateTime.Now` doit récupérer une règle du système d'exploitation pour avoir les dates de début/fin de l'heure d'été et de combien on décale l'heure par rapport à la normale. Ce qui ajoute du temps de traitement en conséquence. La piste qui sera explorée pour résoudre ce problème est de cacher ces 2 dates+delta pour ne pas avoir à les lire à chaque fois. Ceci dit, il restera à adresser l'invalidation du cache quand un utilisateur invalide le passage à l'heure d'été sur le système d'exploitation.

## Conclusion

Si vous avez exécuté des tests de performance en fin de semaine sur un code qui utilise intensivement les dates actuelles et que vous vous réveillez Lundi avec des résultats dégradés, ne paniquez pas. Le problème est connu depuis un bon moment et il y a déjà un [ticket](https://github.com/dotnet/runtime/issues/67932) de 2022 ouvert à ce sujet. Qui sera le héro qui se sacrifiera pour nous corriger ce problème ? En tout cas, le ticket est ouvert pour contribution externe et il y a déjà dessus une piste d'amélioration proposée. Bon dév !!!!