
# 🚀 Django ORM - Guide complet de `select_related()` et `prefetch_related()`

## Sommaire

- [1. Introduction aux optimisations ORM](#1-introduction-aux-optimisations-orm)
- [2. Qu'est-ce que `select_related()` ?](#2-quest-ce-que-select_related)
- [3. Qu'est-ce que `prefetch_related()` ?](#3-quest-ce-que-prefetch_related)
- [4. Pourquoi utiliser ces méthodes (Le problème N+1) ?](#4-pourquoi-utiliser-ces-méthodes-le-problème-n1)
- [5. Exemple sans optimisation](#5-exemple-sans-optimisation)
- [6. Exemple avec `select_related()`](#6-exemple-avec-select_related)
- [7. Exemple avec `prefetch_related()`](#7-exemple-avec-prefetch_related)
- [8. Relations compatibles](#8-relations-compatibles)
- [9. Combiner `select_related()` et `prefetch_related()`](#9-combiner-select_related-et-prefetch_related)
- [10. Différences entre `select_related()` et `prefetch_related()`](#10-différences-entre-select_related-et-prefetch_related)
- [11. Quand utiliser chaque méthode ?](#11-quand-utiliser-chaque-méthode)
- [12. Bonnes pratiques](#12-bonnes-pratiques)
- [13. Résumé pour un entretien](#13-résumé-pour-un-entretien)

---

# 1. Introduction aux optimisations ORM

Par défaut, le Django ORM est **lazy** (fainéant) : il n'exécute des requêtes SQL pour récupérer les objets liés que lorsqu'on y accède explicitement. Cela peut rapidement saturer votre base de données avec des dizaines de requêtes inutiles. `select_related()` et `prefetch_related()` sont les deux outils principaux pour résoudre ce problème.

---

# 2. Qu'est-ce que `select_related()` ?

`select_related()` est une méthode qui fonctionne en créant une **jointure SQL (JOIN)** directement lors de la requête initiale. 

- Elle récupère l'objet principal et ses objets liés en **une seule et unique requête SQL**.
- L'assemblage est entièrement géré par le serveur de base de données.
- Elle est limitée aux relations de type "single-valued" (où l'objet lié est unique).

---

# 3. Qu'est-ce que `prefetch_related()` ?

`prefetch_related()` adopte une stratégie différente. Au lieu de faire une jointure complexe, elle :

- Exécute **plusieurs requêtes SQL** indépendantes (généralement une par relation).
- Récupère toutes les données associées d'un coup.
- Relie ensuite les objets directement **en mémoire grâce à Python**.

Elle est indispensable pour les relations contenant **plusieurs objets** (collections).

---

# 4. Pourquoi utiliser ces méthodes (Le problème N+1) ?

Prenons des modèles où un `Book` possède un `Publisher` (un seul) et plusieurs `Author` (plusieurs).

Sans optimisation, si vous listez 100 livres et affichez leurs éditeurs et leurs auteurs, Django va effectuer :
- 1 requête pour récupérer les 100 livres.
- 100 requêtes pour récupérer l'éditeur de chaque livre.
- 100 requêtes pour récupérer les auteurs de chaque livre.

C'est le problème des **requêtes N+1** (ici $1 + N + N = 201$ requêtes). Grâce à nos deux méthodes, on peut réduire cela à seulement **3 requêtes SQL**.

---

# 5. Exemple sans optimisation

### Modèles

```python
from django.db import models


class Publisher(models.Model):
    name = models.CharField(max_length=100)


class Author(models.Model):
    name = models.CharField(max_length=100)


class Book(models.Model):
    title = models.CharField(max_length=100)
    publisher = models.ForeignKey(Publisher, on_delete=models.CASCADE)
    authors = models.ManyToManyField(Author)

```

### Code non optimisé

```python
books = Book.objects.all()

for book in books:
    print(
        f"{book.title} par {book.publisher.name}"
    )  # Déclenche 1 requête par livre (ForeignKey)
    for author in book.authors.all():
        print(author.name)  # Déclenche 1 autre requête par livre (ManyToMany)

```

---

# 6. Exemple avec `select_related()`

Pour optimiser l'accès à l'éditeur (`publisher`), on utilise `select_related()` car c'est une relation `ForeignKey`.

```python
# 1 seule requête SQL incluant un INNER JOIN
books = Book.objects.select_related("publisher")

for book in books:
    # L'éditeur est déjà chargé en mémoire, aucune requête SQL supplémentaire ici
    print(f"{book.title} par {book.publisher.name}")

```

### SQL exécuté en arrière-plan

```sql
SELECT book.id, book.title, book.publisher_id, publisher.id, publisher.name
FROM book
INNER JOIN publisher ON (book.publisher_id = publisher.id);

```

---

# 7. Exemple avec `prefetch_related()`

Pour optimiser l'accès aux auteurs (`authors`), on utilise `prefetch_related()` car c'est une relation `ManyToMany`.

```python
# 2 requêtes SQL distinctes
books = Book.objects.prefetch_related("authors")

for book in books:
    print(book.title)
    # Les auteurs ont été préchargés en mémoire, pas de requête SQL dans la boucle
    for author in book.authors.all():
        print(author.name)

```

### SQL exécuté en arrière-plan

```sql
-- Requête 1
SELECT * FROM book;

-- Requête 2
SELECT author.*, book_authors.book_id 
FROM author
INNER JOIN book_authors ON (author.id = book_authors.author_id)
WHERE book_authors.book_id IN (1, 2, 3, ...);

```

---

# 8. Relations compatibles

| Méthode | Relations compatibles | Interdit / Inutile avec |
| --- | --- | --- |
| **`select_related()`** | `ForeignKey``OneToOneField` | `ManyToMany`Reverse `ForeignKey` |
| **`prefetch_related()`** | `ManyToMany`Reverse `ForeignKey` (`related_name`)`GenericRelation` | *Techniquement possible sur une ForeignKey mais sous-optimal par rapport à select_related.* |

---

# 9. Combiner `select_related()` et `prefetch_related()`

Il est tout à fait possible (et recommandé) de chaîner les deux méthodes pour couvrir l'intégralité des relations d'un modèle en un minimum de requêtes.

```python
books = (
    Book.objects.select_related(
        "publisher"
    )  # Jointure SQL pour la ForeignKey
    .prefetch_related(
        "authors"
    )  # Requête SQL séparée pour le ManyToMany
)

```

**Total : 2 requêtes SQL au lieu de 201.**

---

# 10. Différences entre `select_related()` et `prefetch_related()`

| Critère | `select_related()` | `prefetch_related()` |
| --- | --- | --- |
| **Type de relation** | Liaison unique (`ForeignKey`, `OneToOne`) | Liens multiples (`ManyToMany`, Reverse FK) |
| **Nombre de requêtes** | Strictement **1** | **2 ou plus** (1 par relation préchargée) |
| **Technique SQL** | `JOIN` SQL (`INNER JOIN` / `LEFT OUTER JOIN`) | Plusieurs requêtes séparées (`WHERE IN (...)`) |
| **Assemblage** | Fait par la Base de Données | Fait en mémoire par Python |
| **Performance** | Idéal pour les tables liées uniques | Idéal pour éviter les grosses jointures coûteuses |

---

# 11. Quand utiliser chaque méthode ?

## Utiliser `select_related()`

Dès que vous devez accéder à un champ d'un objet lié via une **ForeignKey** ou un **OneToOneField** dans une boucle ou un affichage de liste.

```python
# Exemple
Profile.objects.select_related("user")

```

## Utiliser `prefetch_related()`

Dès que vous manipulez une liste d'objets et que vous devez boucler sur leurs relations **ManyToMany** ou leurs **relations inverses**.

```python
# Exemple (Un auteur a plusieurs livres -> Relation inverse)
Author.objects.prefetch_related("books")

```

---

# 12. Bonnes pratiques

✅ **Anticipez les besoins :** Si vous écrivez une boucle imbriquée, vous avez presque toujours besoin de l'une de ces deux fonctions.

❌ **N'utilisez pas `select_related` sur un ManyToMany :** Django lèvera une erreur (`FieldError`).

✅ **Soyez précis :** Ne préchargez que les relations dont vous avez réellement besoin pour ne pas surcharger inutilement la mémoire RAM de votre serveur Python.

❌ **Évitez de filtrer après un prefetch :** Si vous faites `book.authors.filter(status="active")` dans votre boucle, Django invalidera le cache mémoire du prefetch et réexécutera une requête SQL. Utilisez plutôt l'objet `Prefetch()` de Django si des filtres sont nécessaires.

---

# 13. Résumé pour un entretien

* `select_related()` et `prefetch_related()` servent à éliminer le problème de performance **N+1**.
* **`select_related()`** fait **1 seule requête** avec un **JOIN SQL** (pour les relations de type `ForeignKey` et `OneToOne`).
* **`prefetch_related()`** fait **plusieurs requêtes** et fusionne les données **en mémoire avec Python** (pour les relations de type `ManyToMany` ou reverse `ForeignKey`).
* Les deux peuvent être combinés pour obtenir une requête parfaitement optimisée.

```

```
