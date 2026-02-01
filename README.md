# dotnet-mkdocs-material

Build a .NET project (triggering [MkDocsSharp.MDGen](https://github.com/egoughnour/mkdocs-sharp) markdown generation from XML doc comments), then build a [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) site.

## How it works

1. **`dotnet build`** — Compiles your project, which triggers `MkDocsSharp.MDGen` to generate Markdown files in `docs/`
2. **Generate `mkdocs.yml`** — If missing, creates a minimal config with sensible defaults
3. **`pip install`** — Installs MkDocs from `docs/requirements.txt` (or falls back to `mkdocs-material`)
4. **`mkdocs build`** — Produces a static site in `site/`

## Prerequisites

Your .NET project needs:

```xml
<PropertyGroup>
  <GenerateDocumentationFile>true</GenerateDocumentationFile>
</PropertyGroup>

<ItemGroup>
  <PackageReference Include="MkDocsSharp.MDGen" Version="1.6.0" />
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
| `site_name` | `API Documentation` | Site name (for auto-generated mkdocs.yml) |
| `strict` | `false` | Fail on MkDocs warnings |
| `dotnet_build_args` | | Extra args for `dotnet build` |
| `mkdocs_build_args` | | Extra args for `mkdocs build` |

## Outputs

| Output | Description |
|--------|-------------|
| `site_dir` | Path to built site (`site`) |

## Examples

### Minimal

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

### With custom requirements

Create `docs/requirements.txt` to pin specific versions:

```txt
mkdocs-material==9.5.18
mkdocs-awesome-pages-plugin==2.9.2
```

Then the action will use your pinned versions instead of defaults.

### With strict mode

```yaml
- uses: egoughnour/dotnet-mkdocs-material@v1
  with:
    strict: "true"
```

## Customizing mkdocs.yml

The action generates a minimal `mkdocs.yml` if one doesn't exist. For more control, create your own:

```yaml
site_name: My Project
docs_dir: docs
site_dir: site

theme:
  name: material
  palette:
    primary: indigo

nav:
  - Home: index.md
  - API Reference: api/

plugins:
  - search

markdown_extensions:
  - admonition
  - pymdownx.highlight
  - pymdownx.superfences
```

## License

MIT
