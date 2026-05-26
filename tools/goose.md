# Goose AI Agent Knowledge

## Config Locations
- **Windows:** `%APPDATA%\Block\goose\config\config.yaml`
- **macOS/Linux:** `~/.config/goose/config.yaml`

## Config Format
Top-level YAML keys (same names as env vars):

```yaml
GOOSE_PROVIDER: ollama
GOOSE_MODEL: qwen2.5-coder:7b
OLLAMA_HOST: http://localhost:11434
OLLAMA_TIMEOUT: '600'

# Optional: dual-model setup
GOOSE_PLANNER_PROVIDER: ollama
GOOSE_PLANNER_MODEL: qwen2.5-coder:32b
```

## Key Settings
| Setting | Purpose | Default |
|---------|---------|---------|
| `GOOSE_PROVIDER` | LLM provider | (must set) |
| `GOOSE_MODEL` | Model name | (must set) |
| `OLLAMA_HOST` | Ollama server URL | `http://localhost:11434` |
| `OLLAMA_TIMEOUT` | Request timeout (seconds) | `600` |
| `GOOSE_MODE` | Tool execution mode | `auto` |
| `GOOSE_TOOLSHIM` | Enable tool-call shim | `false` |
| `GOOSE_TEMPERATURE` | Sampling temperature | `0.7` |

## Precedence
Environment variables > config.yaml > defaults

## Extensions
Extensions are configured in config.yaml under the `extensions:` key.
Key builtins: `developer`, `memory`, `todo`, `skills`, `apps`

## Remote Ollama
Point `OLLAMA_HOST` to a remote machine:
```yaml
OLLAMA_HOST: http://192.168.1.10:11434
```
The remote machine must have Ollama running on `0.0.0.0:11434` with firewall open.

## Tool Shim
For models with weak native tool-calling support, enable:
```yaml
GOOSE_TOOLSHIM: true
GOOSE_TOOLSHIM_OLLAMA_MODEL: llama3.2
```

## Project-Level Config
- `.goose.yaml` in project root for project-specific settings
- `.goosehints` in project root for behavior hints
