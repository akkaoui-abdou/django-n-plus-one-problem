# 🚀 Django ORM - Guide complet de `select_related()` et `prefetch_related()`

## Sommaire

- [1. Introduction](#1-introduction)
- [2. Le problème des requêtes N+1](#2-le-problème-des-requêtes-n1)
- [3. `select_related()`](#3-select_related)
  - [3.1 Qu'est-ce que `select_related()` ?](#31-quest-ce-que-select_related)
  - [3.2 Exemple sans `select_related()`](#32-exemple-sans-select_related)
  - [3.3 Exemple avec `select_related()`](#33-exemple-avec-select_related)
  - [3.4 Fonctionnement interne](#34-fonctionnement-interne)
  - [3.5 Relations compatibles](#35-relations-compatibles)
  - [3.6 Exemple avec plusieurs ForeignKey](#36-exemple-avec-plusieurs-foreignkey)
  - [3.7 Exemple avec une relation OneToOne](#37-exemple-avec-une-relation-onetoone)
  - [3.8 Traverser plusieurs niveaux de relations](#38-traverser-plusieurs-niveaux-de-relations)
- [4. `prefetch_related()`](#4-prefetch_related)
  - [4.1 Qu'est-ce que `prefetch_related()` ?](#41-quest-ce-que-prefetch_related)
  - [4.2 Exemple sans `prefetch_related()`](#42-exemple-sans-prefetch_related)
  - [4.3 Exemple avec `prefetch_related()`](#43-exemple-avec-prefetch_related)
  - [4.4 Fonctionnement interne](#44-fonctionnement-interne)
  - [4.5 Relations compatibles](#45-relations-compatibles)
  - [4.6 Exemple avec une relation ManyToMany](#46-exemple-avec-une-relation-manytomany)
  - [4.7 Exemple avec une relation inverse (Reverse ForeignKey)](#47-exemple-avec-une-relation-inverse-reverse-foreignkey)
  - [4.8 Précharger plusieurs relations](#48-précharger-plusieurs-relations)
- [5. Combiner `select_related()` et `prefetch_related()`](#5-combiner-select_related-et-prefetch_related)
- [6. Différences entre les deux méthodes](#6-différences-entre-les-deux-méthodes)
- [7. Quand utiliser chaque méthode ?](#7-quand-utiliser-chaque-méthode)
- [8. Bonnes pratiques](#8-bonnes-pratiques)
- [9. Résumé pour un entretien](#9-résumé-pour-un-entretien)

---

# 1. Introduction

`select_related()` et `prefetch_related()` sont deux méthodes du **Django ORM** permettant d'optimiser les performances des requêtes lorsque l'on manipule des relations entre modèles.

Elles répondent toutes les deux au même problème — les **requêtes N+1** — mais avec des techniques différentes, adaptées à des types de relations différents :

| Méthode | Technique | Type de relation |
|---------|-----------|-------------------|
| `select_related()` | JOIN SQL | Relations à objet unique (ForeignKey, OneToOne) |
| `prefetch_related()` | Plusieurs requêtes SQL + assemblage Python | Relations à plusieurs objets (ManyToMany, Reverse FK) |

---

# 2. Le problème des requêtes N+1

Le problème des **requêtes N+1** survient lorsqu'une requête initiale (1) est suivie d'une requête supplémentaire pour **chaque** objet retourné (N).

Exemple avec 100 livres :

```
1 requête pour les livres

+100 requêtes pour la relation associée (éditeur, auteurs, etc.)

=101 requêtes SQL
```

C'est précisément ce que `select_related()` et `prefetch_related()` permettent d'éviter, chacune selon le type de relation concerné.

---

# 3. `select_related()`

## 3.1 Qu'est-ce que `select_related()` ?

`select_related()` est utilisé pour optimiser les relations où l'objet lié est **unique**.

Il :

- effectue un **JOIN SQL**,
- récupère l'objet principal et l'objet lié **en une seule requête**,
- fonctionne uniquement sur des relations à objet unique.

---

## 3.2 Exemple sans `select_related()`

## Modèles

```python
from django.db import models


class Publisher(models.Model):
    name = models.CharField(max_length=100)


class Book(models.Model):
    title = models.CharField(max_length=100)
    publisher = models.ForeignKey(
        Publisher,
        on_delete=models.CASCADE
    )
```

Requête

```python
books = Book.objects.all()

for book in books:
    print(book.title)
    print(book.publisher.name)
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
FROM publisher
WHERE id = ...;
```

Pour 100 livres :

```
1 + 100 = 101 requêtes SQL
```

---

## 3.3 Exemple avec `select_related()`

```python
books = Book.objects.select_related("publisher")

for book in books:
    print(book.title)
    print(book.publisher.name)
```

Cette fois Django effectue une seule requête, grâce à un `JOIN` :

```sql
SELECT book.*, publisher.*
FROM book
INNER JOIN publisher
ON book.publisher_id = publisher.id;
```

Nombre total :

```
1 requête SQL
```

---

## 3.4 Fonctionnement interne

```
Étape 1

JOIN SQL entre Book et Publisher

↓

Étape 2

Une seule ligne de résultat contient les deux objets

↓

Étape 3

Django construit les objets Python à partir de cette ligne unique
```

Un seul `JOIN` est utilisé, aucune requête supplémentaire n'est nécessaire.

---

## 3.5 Relations compatibles

`select_related()` fonctionne avec :

- ✅ ForeignKey
- ✅ OneToOneField

Il ne doit **pas** être utilisé pour :

- ManyToManyField
- Reverse ForeignKey
- GenericRelation

---

## 3.6 Exemple avec plusieurs ForeignKey

```python
class Publisher(models.Model):
    name = models.CharField(max_length=100)


class Editor(models.Model):
    name = models.CharField(max_length=100)


class Book(models.Model):
    title = models.CharField(max_length=100)

    publisher = models.ForeignKey(Publisher, on_delete=models.CASCADE)
    editor = models.ForeignKey(Editor, on_delete=models.CASCADE)
```

Optimisation :

```python
books = Book.objects.select_related(
    "publisher",
    "editor"
)
```

Une seule requête SQL sera exécutée, avec deux `JOIN` :

```sql
SELECT book.*, publisher.*, editor.*
FROM book
INNER JOIN publisher ON ...
INNER JOIN editor ON ...
```

---

## 3.7 Exemple avec une relation OneToOne

```python
class Profile(models.Model):
    bio = models.TextField()


class Author(models.Model):
    name = models.CharField(max_length=100)
    profile = models.OneToOneField(
        Profile,
        on_delete=models.CASCADE
    )
```

Optimisation :

```python
authors = Author.objects.select_related("profile")

for author in authors:
    print(author.name)
    print(author.profile.bio)
```

Django effectue uniquement une seule requête grâce au JOIN.

---

## 3.8 Traverser plusieurs niveaux de relations

Il est possible de traverser plusieurs niveaux de relations avec la notation `__`.

```python
class Country(models.Model):
    name = models.CharField(max_length=100)


class Publisher(models.Model):
    name = models.CharField(max_length=100)
    country = models.ForeignKey(Country, on_delete=models.CASCADE)


class Book(models.Model):
    title = models.CharField(max_length=100)
    publisher = models.ForeignKey(Publisher, on_delete=models.CASCADE)
```

Optimisation sur deux niveaux :

```python
books = Book.objects.select_related("publisher__country")

for book in books:
    print(book.publisher.country.name)
```

Une seule requête SQL est exécutée, avec deux `JOIN` en cascade.

---

# 4. `prefetch_related()`

## 4.1 Qu'est-ce que `prefetch_related()` ?

`prefetch_related()` est utilisé pour optimiser les relations où l'objet lié contient **plusieurs objets**.

Il :

- exécute plusieurs requêtes SQL,
- récupère les données associées,
- relie ensuite les objets directement en mémoire grâce à Python.

---

## 4.2 Exemple sans `prefetch_related()`

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

## 4.3 Exemple avec `prefetch_related()`

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

## 4.4 Fonctionnement interne

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

## 4.5 Relations compatibles

`prefetch_related()` fonctionne avec :

- ✅ ManyToMany
- ✅ Reverse ForeignKey (`related_name`)
- ✅ GenericRelation

Il ne doit **pas** être utilisé pour :

- ForeignKey
- OneToOneField

---

## 4.6 Exemple avec une relation ManyToMany

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

## 4.7 Exemple avec une relation inverse (Reverse ForeignKey)

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

Optimisation :

```python
authors = Author.objects.prefetch_related("books")

for author in authors:
    print(author.name)

    for book in author.books.all():
        print(book.title)
```

Django effectue uniquement deux requêtes.

---

## 4.8 Précharger plusieurs relations

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

# 5. Combiner `select_related()` et `prefetch_related()`

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

# 6. Différences entre les deux méthodes

| Critère | `select_related()` | `prefetch_related()` |
|----------|--------------------|----------------------|
| Type de relation | ForeignKey, OneToOne | ManyToMany, Reverse FK |
| Nombre de requêtes | 1 | 2 ou plus |
| Technique | JOIN SQL | Plusieurs requêtes SQL |
| Assemblage | SQL | Python |
| Performance | Excellente pour une relation simple | Excellente pour plusieurs relations |

---

# 7. Quand utiliser chaque méthode ?

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

# 8. Bonnes pratiques

✅ Utiliser `select_related()` pour toute relation **ForeignKey** ou **OneToOne** accédée en boucle.

```python
for book in books:
    print(book.publisher.name)
```

---

✅ Utiliser `prefetch_related()` pour toute relation **ManyToMany** ou **Reverse ForeignKey** accédée en boucle.

```python
for book in books:
    for author in book.authors.all():
        ...
```

---

✅ Regrouper plusieurs relations dans un seul appel.

```python
Book.objects.select_related(
    "publisher",
    "editor",
)

Book.objects.prefetch_related(
    "authors",
    "reviews",
    "categories",
)
```

---

✅ Utiliser la notation `__` pour traverser plusieurs niveaux avec `select_related()`.

```python
Book.objects.select_related("publisher__country")
```

---

✅ Combiner les deux méthodes lorsque le modèle contient les deux types de relations.

```python
Book.objects.select_related(
    "publisher"
).prefetch_related(
    "authors",
    "reviews",
)
```

---

❌ Éviter d'accéder à une relation en boucle sans optimisation :

```python
for book in books:
    print(book.publisher.name)          # sans select_related()

for author in authors:
    books = author.books.all()          # sans prefetch_related()
```

---

❌ Ne pas utiliser `select_related()` sur une relation **ManyToMany** ou **Reverse ForeignKey** : cela génère une erreur, car `select_related()` ne fonctionne que sur des relations à objet unique.

❌ Ne pas utiliser `prefetch_related()` là où `select_related()` suffit : cela génère des requêtes supplémentaires inutiles.

---

# 9. Résumé pour un entretien

- `select_related()` et `prefetch_related()` permettent tous deux d'optimiser les performances du Django ORM en évitant le problème des **requêtes N+1**.
- `select_related()` :
  - s'utilise avec **ForeignKey** et **OneToOneField**,
  - exécute une seule requête SQL grâce à un **JOIN**,
  - peut traverser plusieurs niveaux de relations avec la notation `__` (ex: `publisher__country`).
- `prefetch_related()` :
  - s'utilise avec **ManyToMany**, **Reverse ForeignKey** et **GenericRelation**,
  - exécute plusieurs requêtes SQL puis associe les résultats **en mémoire, via Python**,
  - ne réalise aucun `JOIN`.
- Les deux méthodes sont souvent **combinées** dans une même requête pour optimiser l'ensemble des relations d'un modèle.

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
    .select_related("publisher__country")
    .prefetch_related(
        "authors",
        "reviews",
        "categories",
    )
)
```

Cette approche est considérée comme une **bonne pratique Django** pour développer des applications performantes et éviter les problèmes de requêtes N+1.
