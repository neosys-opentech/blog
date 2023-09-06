---
layout: post
author: pedrobsaila
date: 2023-09-06 20:00:00
title: Performance de LINQ
front_image: /assets/images/posts/2023-09-06-net-linq-perf/logo.jpg
excerpt_separator: <!--more-->
---

LINQ est un ensemble de méthodes qui traitent les collections de données fonctionnellement. En fournissant une fonction qui va transformer/filrer/aggréger/projeter un élément, LINQ est capable de propager la fonction sur l'entiereté de la collection. Ce qui permet d'aligner l'écriture des traitements itératifs sur des collections et en simplifier la complexité cyclomatique. La convenance des fonctions LINQ a un coût et le but de ce billet sera de vous présenter ce qu'on perd en performance en les privilégiant.

<!--more-->

# Application démo

Pour expliquer les différents coûts engendrés par LINQ, je vous propose cette application :

{% gist b7cef4a54bb2c204f429c44ec36634b9 %}

Elle contient un dictionnaire clé/valeur sur lequel on applique un filtre et une aggrégation. On y trouve 2 versions : une utilisant LINQ et une autre utilisant un simple foreach. Je me servirai de [BenchmarkDotnet](https://github.com/dotnet/BenchmarkDotNet) dans un 1er temps pour comparer les 2 implémentations :

{:.table}
| Method |      Mean |     Error |    StdDev |   Gen0 | Allocated |
|--------|-----------|-----------|-----------|--------|-----------|
|   Linq | 915.13 ns | 11.904 ns | 11.135 ns | 0.0162 |     208 B |
|   Loop |  87.61 ns |  0.753 ns |  0.704 ns |        |           |

On remarque 2 choses :
* La version avec foreach n'alloue quasiment pas de mémoire alors que si pour LINQ (colonne Allocated).
* La version avec LINQ est 10 fois plus lente que la version en foreach (colonne Mean, ordre de grandeur en ns).

# Analyse des allocations mémoire

Pour expliquer les derniers résultats, je vais utiliser l'outil *PerfView* dont j'ai déjà fait la présentation sur ce [billet](https://devblogs.microsoft.com/dotnet/gc-etw-events-1/) :

* Démarrons le :

{:.ns-post-img-fluid}
![alt Starting perfview]({{ '/assets/images/posts/2023-09-06-net-linq-perf/perview-menu.png' | relative_url }}){:.mx-auto}{:.d-block}

* Je choisis Collect -> Collect dans le menu :

{:.ns-post-img-fluid}
![alt Collect menu]({{ '/assets/images/posts/2023-09-06-net-linq-perf/collect-menu.png' | relative_url }}){:.mx-auto}{:.d-block}

* Je déroule *Advanced Options* et je coche *ETW .NET Alloc* : permettra d'écouter les événements d'allocation mémoire avec les types associés.

* Je lance mon application `dotnet run .\PimpMyNet.csproj -c Release`

* Quand l'exécution de mon appli se termine, j'annule la collection avec *Cancel* et j'attends que les données soient consolidés dans un fichier *.etl

* J'ouvre ce dernier avec Perfview et je double clique sur la sous arborescence *Events* :

{:.ns-post-img-fluid}
![alt Events leaf]({{ '/assets/images/posts/2023-09-06-net-linq-perf/perfview-dump.png' | relative_url }}){:.mx-auto}{:.d-block}

* Je filtre les événements en saisissant *Allocation* :

{:.ns-post-img-fluid}
![alt Extract process id]({{ '/assets/images/posts/2023-09-06-net-linq-perf/get-alloc-event.png' | relative_url }}){:.mx-auto}{:.d-block}

* Je fais clique-droit sur le type d'événement *Microsoft-Windows-DotNETRuntime/GC/AllocationTick* et choisis *Open Any Stack* et après l'onglet *CallTree* et renseigne le nom du projet `PimpMyNet` dans Find.

{:.ns-post-img-fluid}
![alt Extract process id]({{ '/assets/images/posts/2023-09-06-net-linq-perf/find-process-id.png' | relative_url }}){:.mx-auto}{:.d-block}

* Je choisis la ligne avec `--benchmarkName Program.Linq` et garde l'identifiant du process (19520 dans ce cas), reviens à l'onglet *Callers* et je rensigne l'identifiant dans le champ *IncPaths* : le but étant de filtrer les événements d'allocation mémoire qui viennent uniquement de notre application et du benchmarking avec LINQ

{:.ns-post-img-fluid}
![alt Allocations in perfview]({{ '/assets/images/posts/2023-09-06-net-linq-perf/perfview-results.png' | relative_url }}){:.mx-auto}{:.d-block}

# Interprétation des résultats

Je reprends le dernier tableau sur *PerView* pour mieux de visibilité :

{:.table}
|Name                                                   | Pourcentage d'allocation | Nombre d'objets alloués |
|-------------------------------------------------------|--------------------------|-------------------------|
|`WhereEnumerableIterator<KeyValuePair<TimeSpan, int>>` |19.3	                     |16,370                   |
|`Enumerator<TimeSpan, int>`                            |16.8	                     |14,271                   |
|`Func<KeyValuePair<System.TimeSpan, int>, bool>`	      |19.3	                     |16,376                   |
|`<>c__DisplayClass3_0`                                 |7.3	                     |6,180                    |

Pour comprendre les différents types alloués sur ce tableau, il faut savoir que la version LINQ est une abstration du code suivant :

{% gist 5998e09c34086d5b840842932c64c594 %}

* `WhereEnumerableIterator<KeyValuePair<System.TimeSpan, int>>` : correspond à l'objet `IEnumerator` retourné par la méthode `Where`.
* `Enumerator<TimeSpan, int>` : correspond à l'objet `IEnumerator` créé par le Sum pour parcourir le résultat en sortie du `Where`.
* `Func<KeyValuePair<System.TimeSpan, int>, bool>`: un delegate représentant la lambda expression en paramètre du `Where`.
* `<>c__DisplayClass3_0` :
  * c'est une classe générée par le compilateur pour stocker les delegates des expressions anonymes en paramètre du `Where` (`<>c__DisplayClass3_0`) et `Sum` (`<>c__DisplayClass3_1`).
  * La classe permet aussi d'implémenter une clotûre pour la variable `value` définit hors-scope : Quand le compilateur détecte ce genre d'inclusion dans le scope d'une lambda, il génère un champ (`public int value` dans l'exemple) qui garde une référence sur la variable comme ça le GC n'y touche pas.

Ainsi on voit clairement que LINQ consomme :

* En terme de mémoire :
  * On alloue 2 `IEnumerator` pour `Where` et `Sum` (alors que la version sans LINQ alloue un seul pour le `foreach`)
  * On alloue une clotûre
  * La clotûre empêche la variable hors-scope d'être collecté par le GC : donc reste en vie plus longtemps et peut être promu aux générations supérieurs allongeant son cycle de vie.
  * On alloue 2 delegate pour les fonctions lambda en paramètre du `Where` et `Sum`

* En terme de temps CPU :
  * On doit caster le retour du GetEnumerator car `Dictionary<TKey, TValue>` expose publiquement la version non-générique.
  * Les appels `MoveNext` et `Current` se font sur l'interface `IEnumerator` : l'environnement d'exécution perd du temps additionnel pour déterminer la classe concrête à appeler pour les 2 objets de ce type.

# Conclusion

Ne soyez pas tenté de faire une grosse refacto éliminant LINQ sauf si vous être sûrs que c'est le point de contention :grin:. Il simplifie malgré tout la compexité cyclomatique du code le rendant plus lisible et compréhensible pour les autres copains. Dans la majorité des cas, les points de contention sont les requêtes mal-optimisées, mauvaise gestion de cache, saturation de la bande passante réseau, scalabilité horizontale/verticale non appropriée, distribution de charge non-équilibré... Le but c'est de vous rendre conscient des différents impacts, comment les analyser, et vous encourger à écrire des expressions LINQ simple (sans inclusion de variables externes).

# Références

* [Discussion Github](https://github.com/dotnet/runtime/discussions/45060).
* [ETW](https://docs.microsoft.com/en-us/windows/win32/etw/about-event-tracing).
* [PerfView](https://devblogs.microsoft.com/dotnet/gc-etw-events-1/).
* [Devblogs](https://devblogs.microsoft.com/dotnet/understanding-the-cost-of-csharp-delegates/)