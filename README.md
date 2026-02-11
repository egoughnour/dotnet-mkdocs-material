# dotnet-mkdocs-material

Build a .NET project (triggering [MkDocsSharp.MDGen](https://github.com/egoughnour/mkdocs-sharp) markdown generation from XML doc comments), then build a [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) site.

Supports **multi-project targeting** — each project generates API docs into its own subdirectory under `docs/api/`, producing a unified site with shared search and navigation.

## How it works

1. **`dotnet build`** — Compiles your project(s), triggering `MkDocsSharp.MDGen` to generate Markdown into `docs/`
2. **Generate `mkdocs.yml`** — If missing, creates a config with nav entries for each project
3. **`pip install`** — Installs MkDocs from `requirements.txt` (or falls back to `mkdocs-material`)
4. **`mkdocs build`** — Produces a static site in `site/` (can be skipped for RTD workflows)

## Prerequisites

Your .NET project needs:

```xml
<PropertyGroup>
  <GenerateDocumentationFile>true</GenerateDocumentationFile>
</PropertyGroup>

<ItemGroup>
  <PackageReference Include="MkDocsSharp.MDGen" Version="2.1.0" />
</ItemGroup>
```

That's it. The action handles everything else.

## Quick start

```yaml
- uses: actions/checkout@v4
- uses: egoughnour/dotnet-mkdocs-material@v1
```

## Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `dotnet_version` | `8.0.x` | .NET SDK version |
| `python_version` | `3.12` | Python version |
| `configuration` | `Release` | Build configuration |
| `projects` | *(empty)* | Newline-separated project paths for multi-project builds |
| `site_name` | `API Documentation` | Site name (for auto-generated mkdocs.yml) |
| `strict` | `false` | Fail on MkDocs warnings |
| `build_site` | `true` | Set to `false` to only generate markdown (skip `mkdocs build`) |
| `dotnet_build_args` | | Extra args for `dotnet build` |
| `mkdocs_build_args` | | Extra args for `mkdocs build` |

## Outputs

| Output | Description |
|--------|-------------|
| `site_dir` | Path to built site (`site`), empty when `build_site` is false |
| `docs_dir` | Path to docs directory containing generated markdown |

## Examples

### Single project (original behavior)

```yaml
name: docs
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: egoughnour/dotnet-mkdocs-material@v1
```

### Multi-project targeting

Each project gets its own subdirectory under `docs/api/`. The project name is derived from the `.csproj` filename (dots become dashes, lowercased).

```yaml
- uses: egoughnour/dotnet-mkdocs-material@v1
  with:
    site_name: "Complexity Analysis"
    projects: |
      src/ComplexityAnalysis.Core/ComplexityAnalysis.Core.csproj
      src/ComplexityAnalysis.Roslyn/ComplexityAnalysis.Roslyn.csproj
      src/ComplexityAnalysis.Solver/ComplexityAnalysis.Solver.csproj
      src/ComplexityAnalysis.Calibration/ComplexityAnalysis.Calibration.csproj
      src/ComplexityAnalysis.Engine/ComplexityAnalysis.Engine.csproj
```

This generates:
```
docs/
  api/
    complexityanalysis-core/index.md
    complexityanalysis-roslyn/index.md
    complexityanalysis-solver/index.md
    complexityanalysis-calibration/index.md
    complexityanalysis-engine/index.md
```

### Generate markdown only (for Read the Docs)

When RTD handles the `mkdocs build`, set `build_site: false` so the action only generates markdown:

```yaml
- uses: egoughnour/dotnet-mkdocs-material@v1
  with:
    build_site: "false"
    projects: |
      src/MyLib.Core/MyLib.Core.csproj
      src/MyLib.Api/MyLib.Api.csproj
```

Then push the generated `docs/` to a `docs` branch that RTD watches.

### Deploy to GitHub Pages

```yaml
name: docs

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: egoughnour/dotnet-mkdocs-material@v1
        with:
          site_name: "MyProject API"

      - uses: actions/upload-pages-artifact@v3
        with:
          path: site

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/deploy-pages@v4
        id: deployment
```

### Push docs branch for RTD

Full workflow that generates multi-project markdown and pushes to a `docs` branch:

```yaml
name: docs

on:
  push:
    branches: [main]
    paths:
      - 'src/**'
      - 'docs/**'
      - 'mkdocs.yml'

permissions:
  contents: write

jobs:
  generate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: egoughnour/dotnet-mkdocs-material@v1
        with:
          build_site: "false"
          projects: |
            src/MyLib.Core/MyLib.Core.csproj
            src/MyLib.Api/MyLib.Api.csproj

      - name: Push to docs branch
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

          # Collect docs artifacts
          mkdir -p /tmp/docs-stage
          cp mkdocs.yml /tmp/docs-stage/
          cp -r docs/ /tmp/docs-stage/docs/
          [ -f requirements.txt ] && cp requirements.txt /tmp/docs-stage/
          [ -f .readthedocs.yaml ] && cp .readthedocs.yaml /tmp/docs-stage/

          # Switch to docs branch
          git checkout --orphan docs-tmp
          git rm -rf .
          cp -r /tmp/docs-stage/* .
          git add -A
          git commit -m "docs: update generated API documentation"
          git branch -D docs 2>/dev/null || true
          git branch -m docs
          git push origin docs --force
```

### With custom requirements

Create `docs/requirements.txt` to pin specific versions:

```txt
mkdocs-material==9.5.18
mkdocs-awesome-pages-plugin==2.9.2
```

## Customizing mkdocs.yml

The action generates a minimal `mkdocs.yml` if one doesn't exist. For full control, create your own:

```yaml
site_name: My Project
docs_dir: docs
site_dir: site

theme:
  name: material
  palette:
    - scheme: default
      primary: indigo
    - scheme: slate
      primary: indigo

nav:
  - Home: index.md
  - Guides:
    - Getting Started: guides/quickstart.md
  - API Reference:
    - Core: api/mylib-core/index.md
    - Api: api/mylib-api/index.md

plugins:
  - search
  - awesome-pages

markdown_extensions:
  - admonition
  - pymdownx.highlight
  - pymdownx.superfences
```

## License

MIT
