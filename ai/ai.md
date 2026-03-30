Global config path
```
cd ~
cd .config/opencode
vim opencode.json
```

Copy the content at `opencode.json` (**Change URL path if necessary**):
```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "ollama": {
      "models": {
        "qwen3-coder-next:q8_0": {
          "_launch": true,
          "name": "qwen3-coder-next:q8_0"
        }
      }
    },
    "name": "Ollama (local)",
    "npm": "@ai-sdk/openai-compatible",
    "options": {
      "baseURL": "http://127.0.0.1:11434/v1"
    }
  }
}
```
