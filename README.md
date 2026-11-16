# Techniques de test 2025/2026 : Triangulator

Micro-service de triangulation d'ensembles de points 2D, développé et testé dans le cadre du cours de Techniques de test.
Le service récupère un ensemble de points auprès du `PointSetManager`, calcule sa triangulation avec l'algorithme de Bowyer-Watson (implémenté sans bibliothèque externe) et renvoie le résultat au format binaire.

L'accent du projet est mis sur la démarche de test : plan de tests, tests unitaires, tests d'API, tests d'intégration, tests de performance, couverture et qualité de code.

## Étudiant

- Nom : Chaieb
- Prénom : Yassine
- Groupe : M1 ILSEN classique

## Table des matières

- [Architecture](#architecture)
- [API](#api)
- [Formats binaires](#formats-binaires)
- [Structure du projet](#structure-du-projet)
- [Installation](#installation)
- [Lancement du service](#lancement-du-service)
- [Commandes disponibles](#commandes-disponibles)
- [Stratégie de test](#stratégie-de-test)
- [Algorithme](#algorithme)
- [Documents du projet](#documents-du-projet)

## Architecture

Le `Triangulator` est le composant réalisé dans ce projet. Il ne stocke aucune donnée : il interroge le `PointSetManager` à chaque requête.

```
Client  --->  Triangulator  --->  PointSetManager  --->  Base de données
                (ce projet)
```

Déroulement d'une requête :

1. Le client envoie un `PointSetID` au `Triangulator`.
2. Le `Triangulator` récupère le `PointSet` binaire auprès du `PointSetManager`.
3. Il décode les points et calcule la triangulation.
4. Il renvoie les `Triangles` au client, au format binaire.

## API

Spécification complète : [triangulator.yml](TP/triangulator.yml).

### `GET /triangulation/{pointSetId}`

| Code | Signification |
|---|---|
| 200 | Triangulation réussie, réponse binaire (`application/octet-stream`) |
| 400 | `pointSetId` qui n'est pas un UUID valide |
| 404 | `PointSetID` inconnu du `PointSetManager` |
| 500 | Données de points invalides ou échec de la triangulation |
| 503 | `PointSetManager` indisponible, trop lent ou en erreur |

Les erreurs sont renvoyées en JSON, avec les champs `code` et `message` définis dans la spécification.

Exemple :

```bash
curl http://localhost:5000/triangulation/123e4567-e89b-12d3-a456-426614174000 --output triangles.bin
```

Le service interroge le `PointSetManager` à l'adresse `http://localhost:5001` (constante `POINTSET_MANAGER_URL` dans `src/app.py`).

## Formats binaires

Tous les entiers et flottants sont codés sur 4 octets, en little-endian.

**PointSet**

| Contenu | Taille |
|---|---|
| Nombre de points N (entier non signé) | 4 octets |
| N points : X (float), Y (float) | 8 octets par point |

**Triangles**

| Contenu | Taille |
|---|---|
| Sommets, identiques à un PointSet | 4 + 8 x N octets |
| Nombre de triangles T (entier non signé) | 4 octets |
| T triangles : 3 indices de sommets | 12 octets par triangle |

L'encodage et le décodage sont dans `src/binary_utils.py`.

## Structure du projet

```
.
├── src/
│   ├── app.py                      # API Flask (route /triangulation/<id>)
│   ├── binary_utils.py             # Encodage et décodage binaire
│   └── triangulator/
│       └── triangulation.py        # Algorithme de triangulation
├── TP/
│   ├── SUJET.md                    # Énoncé
│   ├── PLAN.md                     # Plan de tests
│   ├── RETEX.md                    # Retour d'expérience
│   ├── triangulator.yml            # Spécification OpenAPI du Triangulator
│   ├── point_set_manager.yml       # Spécification OpenAPI du PointSetManager
│   └── tests/
│       ├── conftest.py             # Configuration pytest et marqueur perf
│       ├── unit/                   # Triangulation et binaire
│       ├── api/                    # Contrat de l'API
│       ├── integration/            # Flux complet avec PointSetManager simulé
│       └── perf/                   # Tests de performance
├── makefile
├── pyproject.toml                  # Règles ruff
├── requirements.txt
└── dev_requirements.txt
```

## Installation

Prérequis : Python 3.10 ou supérieur.

```bash
git clone https://github.com/chaiebyassine/techniques_de_test_2025_2026.git
cd techniques_de_test_2025_2026

python -m venv .venv
source .venv/bin/activate          # Windows : .venv\Scripts\activate

pip install -r requirements.txt
pip install -r dev_requirements.txt
```

Le service n'a besoin que de Flask : l'appel au `PointSetManager` utilise la bibliothèque standard de Python.

## Lancement du service

Depuis la racine du projet :

```bash
python -m src.app
```

Le service écoute sur le port 5000.

## Commandes disponibles

| Commande | Rôle |
|---|---|
| `make test` | Lance tous les tests (les tests de performance sont ignorés par défaut) |
| `make unit_test` | Lance tous les tests sauf les tests de performance |
| `make perf_test` | Lance uniquement les tests de performance |
| `make coverage` | Génère le rapport de couverture (terminal et HTML) |
| `make lint` | Vérifie la qualité du code avec `ruff check` |
| `make doc` | Génère la documentation HTML avec `pdoc3` dans `docs/` |

## Stratégie de test

Le plan de tests initial est dans [PLAN.md](TP/PLAN.md). Les tests ont été écrits avant l'implémentation (approche test first), puis ajustés une fois le code en place.

| Catégorie | Fichier | Tests | Ce qui est vérifié |
|---|---|---|---|
| Unitaires, triangulation | `TP/tests/unit/test_triangulation.py` | 9 | Triangle minimal, carré, points colinéaires, 0, 1 et 2 points, indices valides, aucun triangle dégénéré, arêtes qui ne se croisent pas |
| Unitaires, binaire | `TP/tests/unit/test_binary_utils.py` | 9 | Tailles attendues, aller-retour encodage/décodage, buffers trop courts ou incomplets, endianness |
| API | `TP/tests/api/test_api.py` | 8 | Réponse 200 et binaire valide, UUID invalide (400), PointSetID inconnu (404), PointSet corrompu et échec de triangulation (500), mauvais en-tête `Accept`, format JSON des erreurs |
| Intégration | `TP/tests/integration/test_integration.py` | 9 | Flux complet avec `PointSetManager` simulé, erreurs 404 et 503, et appels HTTP réels vers un faux `PointSetManager` local (réponses 200, 404, 500 et service injoignable) |
| Performance | `TP/tests/perf/test_performance.py` | 5 | Triangulation de 1 000, 5 000 et 10 000 points, encodage/décodage de 1 000 et 10 000 points |

Total : 40 tests. Les 35 tests hors performance passent, la couverture du code de `src/` est de 98 % pour `app.py` et pour la triangulation, et de 93 % pour le module binaire. `make lint` ne signale aucune erreur.

Les tests de performance sont marqués `@pytest.mark.perf`. Ils sont exclus des exécutions normales et se lancent à part avec `make perf_test` (environ 50 secondes).

## Algorithme

La triangulation utilise l'algorithme de **Bowyer-Watson** (triangulation de Delaunay) :

1. Création d'un super-triangle qui englobe tous les points.
2. Insertion des points un par un : suppression des triangles dont le cercle circonscrit contient le nouveau point, puis reconstruction du polygone creux.
3. Suppression des triangles qui touchent les sommets du super-triangle.

Cas particuliers : moins de 3 points ou points tous colinéaires donnent une liste de triangles vide.

La complexité est en O(n²). Pour de grands ensembles de points, une structure spatiale ou un algorithme diviser pour régner en O(n log n) serait plus adapté (voir le retour d'expérience).

## Documents du projet

- [Énoncé](TP/SUJET.md)
- [Plan de tests](TP/PLAN.md)
- [Retour d'expérience](TP/RETEX.md)
