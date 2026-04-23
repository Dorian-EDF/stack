# Consignes IA — Équipe Python EDF

Tu aides une équipe de non-développeurs à créer des outils Python internes.
Priorité absolue : **code simple, lisible, maintenable**. Pas de sur-ingénierie.

---

## 1. Règles obligatoires (à appliquer systématiquement)

### Stack technique

Outils **toujours** utilisés dans chaque projet :

| Besoin | Outil | Interdit |
|---|---|---|
| Projet & dépendances | **uv** | Poetry, pip, venv, pyenv |
| Linter + Formatter | **Ruff** | Black, isort, flake8 |
| Tests | **pytest** | unittest |

Outils à utiliser **uniquement si le projet le nécessite** :

| Besoin | Outil | Interdit |
|---|---|---|
| HTTP | **httpx** | requests |
| Logs | **loguru** (`logger`) | `logging`, `print()` |
| Chemins fichiers | **pathlib.Path** | `os.path`, chemins en dur |
| Config/secrets | **python-dotenv** + variables d'env | Credentials en dur |
| Data (simple) | **pandas** | — |
| Data (performant) | **polars** | — |
| CLI avancée | **typer** | argparse |
| BDD | **sqlalchemy** | — |

Ne pas installer de dépendances inutiles : n'ajouter que ce qui est effectivement utilisé.

### Conventions de code

**Nommage :** `snake_case` (variables, fonctions, fichiers), `PascalCase` (classes), `MAJUSCULES` (constantes).

**Obligatoire :**
- Type hints sur toutes les fonctions : `def f(x: float) -> float:`
- Docstrings Google style pour les fonctions non triviales
- Gestion d'erreurs explicite avec messages clairs

**Recommandé (avec souplesse selon le contexte) :**
- Privilégier les fonctions courtes et à responsabilité unique, mais ne pas découper artificiellement si ça nuit à la lisibilité
- Privilégier un fichier par responsabilité, sans dogmatisme

**Interdit :**
- `print()` pour du logging → `loguru.logger`
- Chemins en dur → `Path`
- Credentials en dur → variables d'environnement + `.env`
- Code "clever" ou sur-ingénierie → privilégier la lisibilité

### Format des commits Git

```
feat: description courte      # nouvelle fonctionnalité
fix: description courte       # correction de bug
docs: description courte      # documentation
refactor: description courte  # nettoyage sans changement fonctionnel
chore: description courte     # maintenance (dépendances, CI...)
```

### Quand tu proposes du code

1. Explique brièvement ce que fait le code
2. Indique les dépendances à ajouter (`uv add ...`)
3. Propose un test pytest simple
4. Le code doit passer `ruff check` et `ruff format`

---

## 2. Structure projet (à utiliser pour tout nouveau projet ou restructuration)

Remplacer `mon-outil` / `mon_outil` par le vrai nom du projet (tirets pour le dossier racine, underscores pour le package Python).

```
mon-outil/
├── src/mon_outil/
│   ├── __init__.py
│   └── main.py
├── tests/
│   └── test_main.py
├── .github/workflows/ci.yml
├── pyproject.toml          ← généré par uv init --package
├── README.md
├── uv.lock                 ← généré par uv sync
├── .python-version         ← généré par uv init
├── .gitignore
└── consignes_ia.md
```

### Compléments à ajouter dans pyproject.toml

`uv init --package` génère le pyproject.toml avec `[project]`, `[project.scripts]` et `[build-system]` (utilise `uv_build`). Ne pas modifier le `[build-system]` généré.

Ajouter les dépendances de développement et la configuration Ruff :

```toml
[dependency-groups]
dev = [
    "pytest>=9.0",
    "ruff>=0.15",
]

[tool.ruff]
line-length = 100
lint.select = ["E", "F", "I", "UP"]

[tool.ruff.format]
quote-style = "double"
```

`uv init --package` génère automatiquement un `[project.scripts]`. Vérifier que le point d'entrée pointe vers la bonne fonction :
```toml
[project.scripts]
mon-outil = "mon_outil.main:main"
```

### .gitignore

```
__pycache__/
*.pyc
.venv/
.env
*.egg-info/
dist/
.ruff_cache/
```

### README.md

Chaque projet doit avoir un README.md qui suit ce format :

```markdown
# Nom de l'outil

## Objectifs de l'outil
À quoi sert l'outil.

## Installation
git clone <url>
uv sync

## Utilisation
uv run mon-outil [arguments]
Description des fonctionnalités.

## Configuration
Variables d'environnement nécessaires :
- `API_KEY` : clé pour le service X

## Contact
@prenom.nom du responsable de l'outil
```

---

## 3. Workflow Git (à appliquer si l'utilisateur travaille sur un repo)

- Ne jamais modifier `main` directement
- Branches : `feature/description` ou `fix/description`
- Avant chaque push :
```bash
uv run ruff check . --fix
uv run ruff format .
uv run pytest
```
- Pull Request → review par le responsable du projet → Squash and merge

---

## 4. Exemple de code conforme

```python
"""Module de calcul de performance."""

from loguru import logger


def calculer_performance(valeurs: list[float], periode: int = 12) -> float:
    """Calcule la performance annualisée.

    Args:
        valeurs: Liste des valeurs mensuelles.
        periode: Nombre de périodes par an.

    Returns:
        Performance annualisée en pourcentage.

    Raises:
        ValueError: Si la liste est vide ou contient des valeurs négatives.
    """
    if not valeurs:
        raise ValueError("La liste de valeurs ne peut pas être vide")
    if any(v < 0 for v in valeurs):
        raise ValueError("Les valeurs ne peuvent pas être négatives")

    rendement = (valeurs[-1] / valeurs[0]) - 1
    performance = ((1 + rendement) ** (periode / len(valeurs))) - 1

    logger.info(f"Performance calculée : {performance:.2%}")
    return performance
```
