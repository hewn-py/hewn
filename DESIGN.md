# hewn design document

## What it is

`hewn` is an opinionated Python project scaffolding tool. You run it in an empty directory and it produces a fully configured, ready-to-work project. It makes decisions for you by default, and gets out of the way.

It follows the principles at [clig.dev](https://clig.dev/).

---

## Project types

```
hewn library    publishable package, hatch build backend
hewn project    general-purpose application
```

`hewn library --entrypoint` additionally generates a `__main__.py`, wires `[project.scripts]` in `pyproject.toml`, and applies clig.dev conventions.

---

## Configuration

### Layers (lowest to highest precedence)

1. **Hardcoded defaults**: shipped with hewn, updated with releases
2. **`~/.config/hewn/config.toml`**: user preferences, persisted across projects
3. **CLI flags**: per-invocation overrides, always win

### Modes

- **Default mode** (no flags): never prompts. All values resolved from layers 1–3.
- **Interactive mode** (`--no-defaults`): prompts for each configurable value, showing the resolved default as the suggestion.

The global config can set `no_defaults = true` for a specific project type, making interactive mode the default when running that type without flags.

### Config structure

```toml
# Global defaults, apply to all project types unless overridden below.
[defaults]
license = "MIT"
python_version = "3.12"
platform = "github"          # controls CI, PR/issue templates, and Pages setup
author_name = "Jane Smith"   # overrides git config user.name if set
author_email = "jane@example.com"

# Per-type overrides, merged on top of [defaults] for that type.
[library]
no_defaults = true
type_checker = "mypy"

```

All keys available under `[defaults]` are also available under `[library]` and `[project]`. Per-type values take precedence over `[defaults]`.

---

## Auto-detection

| Thing        | How                     |
| ------------ | ----------------------- |
| Project name | Current directory name  |
| Author name  | `git config user.name`  |
| Author email | `git config user.email` |

### Git handling

- If `.git` exists: reuse it
- If `.git` does not exist: `git init` with `master` as the default branch

### Non-empty directory

hewn refuses to run if the directory contains anything other than `.git`.

---

## Defaults

| Option                       | Default                                            |
| ---------------------------- | -------------------------------------------------- |
| Package manager              | pdm                                                |
| Build backend (library only) | hatch                                              |
| Type checker                 | basedpyright                                       |
| Linter                       | ruff                                               |
| Formatter                    | ruff                                               |
| Test framework               | pytest                                             |
| License                      | ISC                                                |
| `requires-python`            | hardcoded current stable - 1 (e.g. `>=3.14;<3.15`) |
| Git default branch           | master                                             |
| Platform                     | github                                             |
| Docs (`library`)             | zensical (on)                                      |
| Docs (`project`)             | off                                                |

---

## What gets generated

### All types

- `src/<name>/` with `__init__.py`
- `pyproject.toml`
- `README.md`
- `LICENSE`
- `.gitignore`
- `tests/`
- `CONTRIBUTING.md`
- `SECURITY.md`
- CI quality workflow: typecheck, lint, test run in matrix on push and pull request
- `pdm run` scripts for each tool

### GitHub platform (default)

- `.github/PULL_REQUEST_TEMPLATE.md`
- `.github/ISSUE_TEMPLATE/bug_report.md`
- `.github/ISSUE_TEMPLATE/feature_request.md`
- `renovate.json`: Renovate configuration, extending the `config:recommended` preset and with various additional settings enabled, such as "pinDigests".

### `library` additionally

- Hatch build backend in `pyproject.toml`
- GitHub Actions release workflow: triggered on `v*.*.*` tags, builds and publishes to PyPI via trusted publishing
- README setup section covering:
  1. Configuring the PyPI trusted publisher (OIDC, manual step)
  2. Enabling GitHub Pages from Actions in repo settings
  3. Updating the `github-pages` environment in repo settings to restrict deployments to `v*.*.*` tag pushes exclusively

### `library --entrypoint` additionally

- `src/<name>/__main__.py`
- `[project.scripts]` entry in `pyproject.toml`
- `pdm run app` script

---

## Architecture

### Scaffold state

Everything is built in memory before any file is written. The state is a plain dataclass:

```python
@dataclass
class ScaffoldState:
    pyproject: dict            # serialized to pyproject.toml (TOML)
    gitignore: list[str]       # serialized line by line, order preserved
    readme: list[tuple[str, str]]  # (section title, content), rendered in order
    files: dict[str, str]      # path → content, single owner per path
```

All parts contribute to this shared state. At the end, `state.write()` serializes everything to disk atomically. If `pdm install` fails after that, it is reported as a warning, the scaffold itself is not rolled back.

### CI quality workflow

The quality workflow (`quality.yml`) runs all checks in a matrix so they execute in parallel, each with its own log and independent pass/fail status:

```yaml
jobs:
  quality:
    strategy:
      matrix:
        check: [typecheck, lint, test] # populated from active parts
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.x" }
      - uses: pdm-project/setup-pdm@v4
      - run: pdm install
      - run: pdm run ${{ matrix.check }}
```

Each part contributes its check name (matching its `pdm run` script) to the matrix. The setup steps are fixed because pdm is the only supported package manager. Parts do not own the workflow file, they only declare a check name. The CI assembler collects all check names and builds the workflow.

### Parts

A **part** is a self-contained unit of scaffolding logic. Each part:

- Belongs to a **category** (e.g. `TypeChecker`, `Linter`, `TestFramework`)
- Receives a **context** at instantiation (project type, name, resolved config, other active parts)
- Holds a **config** instance typed to its category's config model
- Contributes to `ScaffoldState` by mutating shared structures or writing owned files

```python
class Part(ABC, Generic[T]):
    def __init__(self, config: T, context: ScaffoldContext) -> None:
        self.config = config
        self.context = context

    @abstractmethod
    def pyproject(self) -> dict: ...           # deep-merged into state.pyproject

    @abstractmethod
    def pdm_scripts(self) -> dict[str, str]:   # merged into [tool.pdm.scripts]
        ...

    def quality_check(self) -> str | None:     # script name contributed to CI matrix
        return None                            # None means no CI step

    def files(self) -> dict[str, str]:         # owned files, optional
        return {}
```

### Categories and registration

Each category defines a **category-level config model** (`msgspec.Struct`). Concrete implementations subclass the category and register themselves automatically:

```python
class TypeCheckerConfig(msgspec.Struct):
    strict: bool = True

class TypeChecker(Part[TypeCheckerConfig]):
    _registry: ClassVar[dict[str, type[TypeChecker]]] = {}

    def __init_subclass__(cls, **kwargs) -> None:
        TypeChecker._registry[cls.slug] = cls

class BasedPyright(TypeChecker):
    slug = "basedpyright"
```

`TypeChecker._registry["basedpyright"]` resolves the implementation. Config is shared at the category level, there is no per-implementation config subclass. Adding a new type checker means adding a new subclass; nothing else changes.

### Config resolution

For each part category, the resolved config is built as:

1. `msgspec.Struct` field defaults (hardcoded)
2. Merged with matching section from `~/.config/hewn/config.toml`
3. Merged with any CLI flags targeting that category

The resulting struct instance is passed to the part on construction. `msgspec.convert` handles decoding and type coercion from the raw config dict.

### Context

`ScaffoldContext` is passed to every part and contains:

- Resolved project metadata (name, type, python version, author, remote URL)
- All active part instances (so parts can introspect siblings, e.g. the CI assembler can enumerate active type checkers and test frameworks)

---

## Docs

Documentation generation is a first-class optional feature.

| Type      | Default       | Options          |
| --------- | ------------- | ---------------- |
| `library` | zensical (on) | sphinx, off      |
| `project` | off           | zensical, sphinx |

When enabled, hewn generates:

- `docs/` with an `index.md`
- Docs tool added as a dev dependency in `pyproject.toml`
- `pdm run docs` script to build/serve locally
- A `docs` check added to the CI quality matrix
- A GitHub Actions docs deployment workflow, with trigger depending on project type:
  - `library`: deploys on `v*.*.*` tag push, using the `github-pages` environment
  - `project`: deploys on push to `master`

**Zensical**: configured via a minimal `zensical.yml` at the project root.

**Sphinx**: generates `docs/conf.py` with autodoc, Napoleon, and the furo theme.

---

## Open questions

- GitLab CI support alongside GitHub Actions?
- Additional project types: what comes after `library` and `project`? Candidates include web APIs, workers/consumers, bots, data pipelines. When should a new type be added vs handled by flags on an existing type?
