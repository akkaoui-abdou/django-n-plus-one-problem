# 🚀 Django ORM - Guide complet de `prefetch_related()`

## Sommaire

- [1. Qu'est-ce que `prefetch_related()` ?](#1-quest-ce-que-prefetch_related)
- [2. Pourquoi utiliser `prefetch_related()` ?](#2-pourquoi-utiliser-prefetch_related)
- [3. Exemple sans `prefetch_related()`](#3-exemple-sans-prefetch_related)
- [4. Exemple avec `prefetch_related()`](#4-exemple-avec-prefetch_related)
- [5. Fonctionnement interne](#5-fonctionnement-interne)
- [6. Relations compatibles](#6-relations-compatibles)
- [7. Exemple avec une relation ManyToMany](#7-exemple-avec-une-relation-manytomany)
- [8. Exemple avec une relation inverse (Reverse ForeignKey)](#8-exemple-avec-une-relation-inverse-reverse-foreignkey)
- [9. Précharger plusieurs relations](#9-précharger-plusieurs-relations)
- [10. Combiner `select_related()` et `prefetch_related()`](#10-combiner-select_related-et-prefetch_related)
- [11. Différences entre `select_related()` et `prefetch_related()`](#11-différences-entre-select_related-et-prefetch_related)
- [12. Quand utiliser chaque méthode ?](#12-quand-utiliser-chaque-méthode)
- [13. Bonnes pratiques](#13-bonnes-pratiques)
- [14. Résumé pour un entretien](#14-résumé-pour-un-entretien)

---

# 1. Qu'est-ce que `prefetch_related()` ?

`prefetch_related()` est une méthode du **Django ORM** permettant d'optimiser les performances des requêtes lorsque l'on manipule des relations contenant **plusieurs objets**.

Contrairement à `select_related()`, qui effectue des **JOIN SQL**, `prefetch_related()` :

- exécute plusieurs requêtes SQL,
- récupère les données associées,
- relie ensuite les objets directement en mémoire grâce à Python.

Son objectif principal est d'éviter le problème des **requêtes N+1**.

---

# 2. Pourquoi utiliser `prefetch_related()` ?

Prenons un exemple :

Un livre possède plusieurs auteurs.

```python
Book
    ├── Author 1
    ├── Author 2
    └── Author 3
```

Sans optimisation, Django effectue une requête supplémentaire pour récupérer les auteurs de chaque livre.

Avec 100 livres :

```
1 requête pour les livres

+100 requêtes pour les auteurs

=101 requêtes SQL
```

`prefetch_related()` réduit cela à seulement **2 requêtes SQL**.

---

# 3. Exemple sans `prefetch_related()`

## Modèles

```python
from django.db import models


class Author(models.Model):
    name = models.CharField(max_length=100)


class Book(models.Model):
    title = models.CharField(max_length=100)
    authors = models.ManyToManyField(Author)
```

Requête

```python
books = Book.objects.all()

for book in books:
    print(book.title)

    for author in book.authors.all():
        print(author.name)
```

### SQL exécuté

Première requête

```sql
SELECT *
FROM book;
```

Puis pour chaque livre

```sql
SELECT *
FROM author
INNER JOIN book_authors
ON ...
```

Pour 100 livres :

```
1 + 100 = 101 requêtes SQL
```

---

# 4. Exemple avec `prefetch_related()`

```python
books = Book.objects.prefetch_related("authors")

for book in books:
    print(book.title)

    for author in book.authors.all():
        print(author.name)
```

Cette fois Django effectue seulement :

```sql
SELECT *
FROM book;
```

Puis

```sql
SELECT *
FROM author
INNER JOIN book_authors
ON ...
```

Nombre total :

```
2 requêtes SQL
```

---

# 5. Fonctionnement interne

Django réalise les étapes suivantes :

```
Étape 1

SELECT Books

↓

Étape 2

SELECT Authors

↓

Étape 3

Association des objets en mémoire
```

Aucun `JOIN` n'est utilisé.

---

# 6. Relations compatibles

`prefetch_related()` fonctionne avec :

- ✅ ManyToMany
- ✅ Reverse ForeignKey (`related_name`)
- ✅ GenericRelation

Il ne doit **pas** être utilisé pour :

- ForeignKey
- OneToOneField

Dans ces cas, utilisez plutôt :

```python
select_related()
```

---

# 7. Exemple avec une relation ManyToMany

```python
class Author(models.Model):
    name = models.CharField(max_length=100)


class Category(models.Model):
    name = models.CharField(max_length=100)


class Book(models.Model):
    title = models.CharField(max_length=100)

    authors = models.ManyToManyField(Author)
    categories = models.ManyToManyField(Category)
```

Optimisation :

```python
books = Book.objects.prefetch_related(
    "authors",
    "categories"
)
```

Seulement trois requêtes SQL seront exécutées :

```
SELECT Books

SELECT Authors

SELECT Categories
```

---

# 8. Exemple avec une relation inverse (Reverse ForeignKey)

```python
class Author(models.Model):
    name = models.CharField(max_length=100)


class Book(models.Model):
    title = models.CharField(max_length=100)

    author = models.ForeignKey(
        Author,
        on_delete=models.CASCADE,
        related_name="books"
    )
```

Sans optimisation

```python
authors = Author.objects.all()

for author in authors:
    print(author.name)

    for book in author.books.all():
        print(book.title)
```

Chaque appel à

```python
author.books.all()
```

déclenche une requête SQL.

Optimisation :

```python
authors = Author.objects.prefetch_related("books")
```

Django effectue uniquement deux requêtes.

---

# 9. Précharger plusieurs relations

Il est possible de précharger plusieurs collections.

```python
books = Book.objects.prefetch_related(
    "authors",
    "categories",
    "reviews",
    "tags",
)
```

Toutes les relations seront récupérées de manière optimisée.

---

# 10. Combiner `select_related()` et `prefetch_related()`

Prenons le modèle suivant.

```python
class Publisher(models.Model):
    name = models.CharField(max_length=100)


class Author(models.Model):
    name = models.CharField(max_length=100)


class Review(models.Model):
    rating = models.IntegerField()


class Book(models.Model):
    title = models.CharField(max_length=100)

    publisher = models.ForeignKey(
        Publisher,
        on_delete=models.CASCADE
    )

    authors = models.ManyToManyField(Author)

    reviews = models.ManyToManyField(Review)
```

La meilleure requête est :

```python
books = (
    Book.objects
    .select_related("publisher")
    .prefetch_related(
        "authors",
        "reviews",
    )
)
```

Pourquoi ?

- `publisher` est une **ForeignKey** → `select_related()`
- `authors` est une **ManyToMany** → `prefetch_related()`
- `reviews` est une **ManyToMany** → `prefetch_related()`

---

# 11. Différences entre `select_related()` et `prefetch_related()`

| Critère | `select_related()` | `prefetch_related()` |
|----------|--------------------|----------------------|
| Type de relation | ForeignKey, OneToOne | ManyToMany, Reverse FK |
| Nombre de requêtes | 1 | 2 ou plus |
| Technique | JOIN SQL | Plusieurs requêtes SQL |
| Assemblage | SQL | Python |
| Performance | Excellente pour une relation simple | Excellente pour plusieurs relations |

---

# 12. Quand utiliser chaque méthode ?

## Utiliser `select_related()`

Lorsqu'il existe une relation :

- ForeignKey
- OneToOneField

Exemple

```python
Book.objects.select_related("publisher")
```

---

## Utiliser `prefetch_related()`

Lorsqu'il existe :

- ManyToMany
- Reverse ForeignKey

Exemple

```python
Book.objects.prefetch_related("authors")
```

---

# 13. Bonnes pratiques

✅ Toujours utiliser `prefetch_related()` lorsqu'une boucle accède à une relation **ManyToMany**.

```python
for book in books:
    for author in book.authors.all():
        ...
```

---

✅ Précharger plusieurs relations si elles sont utilisées.

```python
Book.objects.prefetch_related(
    "authors",
    "reviews",
    "categories",
)
```

---

✅ Combiner avec `select_related()`.

```python
Book.objects.select_related(
    "publisher"
).prefetch_related(
    "authors",
    "reviews",
)
```

---

❌ Éviter :

```python
for author in authors:
    books = author.books.all()
```

sans `prefetch_related()`.

---

# 14. Résumé pour un entretien

- `prefetch_related()` permet d'optimiser les performances du Django ORM.
- Il évite le problème des **requêtes N+1**.
- Il est utilisé avec les relations :
  - ManyToMany
  - Reverse ForeignKey (`related_name`)
  - GenericRelation
- Il exécute plusieurs requêtes SQL puis associe les résultats en mémoire.
- Contrairement à `select_related()`, il **n'utilise pas de JOIN SQL**.
- Il est souvent combiné avec `select_related()` pour optimiser l'ensemble des relations.

## À retenir

| Relation | Méthode |
|----------|----------|
| ForeignKey | `select_related()` |
| OneToOneField | `select_related()` |
| ManyToMany | `prefetch_related()` |
| Reverse ForeignKey | `prefetch_related()` |

### Exemple complet

```python
books = (
    Book.objects
    .select_related("publisher")
    .prefetch_related(
        "authors",
        "reviews",
        "categories",
    )
)
```

Cette approche est considérée comme une **bonne pratique Django** pour développer des applications performantes et éviter les problèmes de requêtes N+1.
````
