# 🚀 Django ORM - Guide complet de `select_related()`

## Sommaire

- [1. Qu'est-ce que `select_related()` ?](#1-quest-ce-que-select_related)
- [2. Pourquoi utiliser `select_related()` ?](#2-pourquoi-utiliser-select_related)
- [3. Exemple sans `select_related()`](#3-exemple-sans-select_related)
- [4. Exemple avec `select_related()`](#4-exemple-avec-select_related)
- [5. Fonctionnement interne](#5-fonctionnement-interne)
- [6. Relations compatibles](#6-relations-compatibles)
- [7. Exemple avec plusieurs ForeignKey](#7-exemple-avec-plusieurs-foreignkey)
- [8. Exemple avec une relation OneToOne](#8-exemple-avec-une-relation-onetoone)
- [9. Traverser plusieurs niveaux de relations](#9-traverser-plusieurs-niveaux-de-relations)
- [10. Combiner `select_related()` et `prefetch_related()`](#10-combiner-select_related-et-prefetch_related)
- [11. Différences entre `select_related()` et `prefetch_related()`](#11-différences-entre-select_related-et-prefetch_related)
- [12. Quand utiliser chaque méthode ?](#12-quand-utiliser-chaque-méthode)
- [13. Bonnes pratiques](#13-bonnes-pratiques)
- [14. Résumé pour un entretien](#14-résumé-pour-un-entretien)

---

# 1. Qu'est-ce que `select_related()` ?

`select_related()` est une méthode du **Django ORM** permettant d'optimiser les performances des requêtes lorsque l'on manipule des relations où l'objet lié est **unique**.

Contrairement à `prefetch_related()`, qui exécute plusieurs requêtes SQL puis assemble les résultats en Python, `select_related()` :

- effectue un **JOIN SQL**,
- récupère l'objet principal et l'objet lié **en une seule requête**,
- fonctionne uniquement sur des relations où un seul objet est associé.

Son objectif principal est, comme `prefetch_related()`, d'éviter le problème des **requêtes N+1**, mais avec une technique différente : le JOIN plutôt que des requêtes multiples.

---

# 2. Pourquoi utiliser `select_related()` ?

Prenons un exemple :

Un livre possède un seul éditeur (publisher).

```python
Book
    └── Publisher
```

Sans optimisation, Django effectue une requête supplémentaire pour récupérer l'éditeur de chaque livre.

Avec 100 livres :

```
1 requête pour les livres

+100 requêtes pour les éditeurs

=101 requêtes SQL
```

`select_related()` réduit cela à **1 seule requête SQL** grâce à un JOIN.

---

# 3. Exemple sans `select_related()`

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

# 4. Exemple avec `select_related()`

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

# 5. Fonctionnement interne

Django réalise les étapes suivantes :

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

# 6. Relations compatibles

`select_related()` fonctionne avec :

- ✅ ForeignKey
- ✅ OneToOneField

Il ne doit **pas** être utilisé pour :

- ManyToManyField
- Reverse ForeignKey
- GenericRelation

Dans ces cas, utilisez plutôt :

```python
prefetch_related()
```

---

# 7. Exemple avec plusieurs ForeignKey

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

# 8. Exemple avec une relation OneToOne

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

Sans optimisation

```python
authors = Author.objects.all()

for author in authors:
    print(author.name)
    print(author.profile.bio)
```

Chaque accès à

```python
author.profile
```

déclenche une requête SQL.

Optimisation :

```python
authors = Author.objects.select_related("profile")
```

Django effectue uniquement une seule requête grâce au JOIN.

---

# 9. Traverser plusieurs niveaux de relations

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

✅ Toujours utiliser `select_related()` lorsqu'une boucle accède à une relation **ForeignKey** ou **OneToOne**.

```python
for book in books:
    print(book.publisher.name)
```

---

✅ Regrouper plusieurs relations dans un seul appel.

```python
Book.objects.select_related(
    "publisher",
    "editor",
)
```

---

✅ Utiliser la notation `__` pour traverser plusieurs niveaux.

```python
Book.objects.select_related("publisher__country")
```

---

✅ Combiner avec `prefetch_related()`.

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
for book in books:
    print(book.publisher.name)
```

sans `select_related()`, car cela déclenche une requête SQL par livre.

---

❌ Ne pas utiliser `select_related()` sur une relation **ManyToMany** ou **Reverse ForeignKey** : cela génère une erreur, car `select_related()` ne fonctionne que sur des relations à objet unique.

---

# 14. Résumé pour un entretien

- `select_related()` permet d'optimiser les performances du Django ORM.
- Il évite le problème des **requêtes N+1**.
- Il est utilisé avec les relations :
  - ForeignKey
  - OneToOneField
- Il exécute une seule requête SQL grâce à un **JOIN**.
- Contrairement à `prefetch_related()`, il **n'assemble pas les objets en Python**, tout se fait au niveau SQL.
- Il peut traverser plusieurs niveaux de relations grâce à la notation `__` (ex: `publisher__country`).
- Il est souvent combiné avec `prefetch_related()` pour optimiser l'ensemble des relations d'un modèle.

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
