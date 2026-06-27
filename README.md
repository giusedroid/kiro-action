# kiro-action

Run Kiro CLI headlessly in GitHub Actions.

`kiro-action` is a small composite action for sending a prompt to Kiro from a workflow. It installs the Kiro CLI, runs `kiro-cli chat` in non-interactive mode, passes `KIRO_API_KEY` through the environment, and exposes Kiro stdout as the `response` output.

```yaml
- uses: giusedroid/kiro-action@v1
  with:
    kiro-api-key: ${{ secrets.KIRO_API_KEY }}
    prompt: |
      Review this repository and update the README.
```

## Requirements

- A GitHub Actions runner with `bash` and `curl`.
- A valid Kiro API key stored in GitHub Secrets.
- Repository checkout before running the action when Kiro needs access to repository files.

## Usage

```yaml
name: Kiro

on:
  workflow_dispatch:

jobs:
  run-kiro:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - id: kiro
        uses: giusedroid/kiro-action@v1
        with:
          kiro-api-key: ${{ secrets.KIRO_API_KEY }}
          prompt: |
            Review this repository and update the README.

      - name: Print Kiro response
        run: |
          printf '%s\n' "${KIRO_RESPONSE}"
        env:
          KIRO_RESPONSE: ${{ steps.kiro.outputs.response }}
```

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `prompt` | Yes | | Prompt to send to Kiro. Multiline prompts are supported. |
| `kiro-api-key` | Yes | | Kiro API key. Use GitHub Secrets. |
| `working-directory` | No | `.` | Directory where Kiro should run. |
| `trust-tools` | No | `read,grep` | Comma-separated list of tools Kiro is allowed to use. Ignored when `trust-all-tools` is `true`. |
| `trust-all-tools` | No | `false` | Set to `true` to allow Kiro to use all available tools. |
| `require-mcp-startup` | No | `false` | Set to `true` to fail the run if configured MCP servers do not start successfully. |

## Outputs

| Output | Description |
| --- | --- |
| `response` | Full stdout emitted by Kiro. |

## Examples

### Simple repository review

```yaml
name: Kiro review

on:
  workflow_dispatch:

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: giusedroid/kiro-action@v1
        with:
          kiro-api-key: ${{ secrets.KIRO_API_KEY }}
          trust-tools: read,grep
          prompt: |
            Review this repository for clarity, correctness, and missing documentation.
            Suggest improvements without modifying files.
```

### Content generation

```yaml
name: Generate content

on:
  workflow_dispatch:

jobs:
  content:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - id: kiro
        uses: giusedroid/kiro-action@v1
        with:
          kiro-api-key: ${{ secrets.KIRO_API_KEY }}
          trust-tools: read,grep
          prompt: |
            Draft release notes from the repository history and current README.
            Return Markdown only.

      - name: Save generated content
        run: |
          printf '%s\n' "${KIRO_RESPONSE}" > release-notes-draft.md
        env:
          KIRO_RESPONSE: ${{ steps.kiro.outputs.response }}
```

### Using write permissions

This action does not commit changes automatically in v1. If you ask Kiro to edit files, review those changes before publishing or merging them. For PR-based workflows, pair this action with a purpose-built action such as `peter-evans/create-pull-request`.

```yaml
name: Kiro write task

on:
  workflow_dispatch:

permissions:
  contents: write

jobs:
  update-docs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: giusedroid/kiro-action@v1
        with:
          kiro-api-key: ${{ secrets.KIRO_API_KEY }}
          trust-tools: read,grep,write
          prompt: |
            Update README.md to include a concise setup section.
            Do not commit the change.

      - name: Show generated diff
        run: git diff -- README.md

      # Optional PR workflow:
      # - uses: peter-evans/create-pull-request@v6
      #   with:
      #     title: Update README with Kiro
      #     commit-message: Update README with Kiro
```

### MCP startup validation

```yaml
name: Kiro with MCP validation

on:
  workflow_dispatch:

jobs:
  mcp:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: giusedroid/kiro-action@v1
        with:
          kiro-api-key: ${{ secrets.KIRO_API_KEY }}
          require-mcp-startup: "true"
          trust-tools: read,grep
          prompt: |
            Use the configured MCP context to summarize the repository structure.
```

## Security Notes

- Prefer least-privilege `trust-tools`. The default is `read,grep`.
- Avoid `trust-all-tools` unless the workflow truly requires broad tool access.
- Store `KIRO_API_KEY` in GitHub Secrets and pass it with `${{ secrets.KIRO_API_KEY }}`.
- Review generated changes before publishing or merging.
- Keep workflow `permissions` as narrow as possible. Only add `contents: write` when a workflow needs to create commits or pull requests.
- Do not run this action on untrusted pull request input with write permissions or broad tool trust.

## Behavior

- Installs Kiro CLI using `curl -fsSL https://cli.kiro.dev/install | bash`.
- Adds `$HOME/.kiro/bin` and `$HOME/.local/bin` to `PATH`.
- Runs `kiro-cli chat --no-interactive`.
- Passes the API key as `KIRO_API_KEY`.
- Captures stdout and exposes it as `steps.<id>.outputs.response`.
- Fails fast when required inputs are empty, boolean inputs are invalid, installation fails, or Kiro exits with a non-zero status.

## Roadmap

- Auto-commit mode.
- PR creation mode.
- Artifact upload.
- Kirolet support.
- MCP presets.
- Content and research workflows.

## License

MIT
