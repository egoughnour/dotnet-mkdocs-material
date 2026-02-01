# Contributing

Thank you for your interest in contributing!

## Development

This is a composite GitHub Action — it's just YAML and Bash, no build step required.

### Testing locally

1. Create a test .NET project with `MkDocsSharp.MDGen` configured
2. Reference the local action in a workflow:
   ```yaml
   - uses: ./path/to/dotnet-mkdocs-material
   ```
3. Run with [act](https://github.com/nektos/act) or push to a test branch

### Pull Request Process

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test the action
5. Submit a pull request

## Code Style

- Bash scripts use `set -euo pipefail`
- YAML uses 2-space indentation
- Use `::notice::` and `::error::` for GitHub Actions annotations
