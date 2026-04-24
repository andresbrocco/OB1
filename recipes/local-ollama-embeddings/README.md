# Local Ollama Embeddings

> Generate embeddings locally via Ollama and insert thoughts into Supabase — no cloud API key required for the embedding step.

## Quick Reference

### Environment Variables

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `SUPABASE_URL` | Supabase project URL | — | Yes (unless `--dry-run`) |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase service role key | — | Yes (unless `--dry-run`) |
| `OLLAMA_BASE_URL` | Base URL of the local Ollama instance | `http://localhost:11434` | No |

### Commands

```bash
# Install dependencies
pip install -r requirements.txt

# Set up environment
cp .env.example .env
# Edit .env, then:
export $(cat .env | xargs)

# Embed a single thought (inline)
python embed-local.py "My important thought"

# Embed from stdin
echo "My important thought" | python embed-local.py

# Embed all lines from a plain-text file
python embed-local.py --file notes.txt

# Embed from a JSONL file
python embed-local.py --file thoughts.jsonl

# Dry run — generate embeddings without writing to Supabase
python embed-local.py --file notes.txt --dry-run --verbose

# Use a specific model and source label
python embed-local.py --file notes.txt --model mxbai-embed-large --source journal
```

### CLI Options

| Flag | Description | Default |
|------|-------------|---------|
| `--file FILE` | Read thoughts from `.txt` or `.jsonl` | — |
| `--model NAME` | Ollama embedding model | `nomic-embed-text` |
| `--ollama-url URL` | Ollama base URL (overrides `OLLAMA_BASE_URL`) | `http://localhost:11434` |
| `--source LABEL` | Source label stored in thought metadata | `ollama-local` |
| `--dry-run` | Embed without inserting into Supabase | off |
| `--verbose` | Print embedding dimension for each thought | off |
| `--batch-size N` | Thoughts per Ollama request | `1` |

### Supported Embedding Models

| Model | Dimensions | Notes |
|-------|-----------|-------|
| `nomic-embed-text` | 768 | Default; fast, small |
| `mxbai-embed-large` | 1024 | Higher quality |
| `rjmalagon/gte-qwen2-1.5b-instruct-embed-f16` | 1536 | Matches default Open Brain schema |

### Input Formats

**Plain text (`.txt`)** — one thought per line, blank lines skipped.

**JSONL (`.jsonl`)** — one JSON object per line:

```jsonl
{"content": "My thought", "source": "journal", "metadata": {"tag": "personal"}}
{"content": "Another thought"}
```

Required key: `content`. Optional keys: `source`, `metadata`.

### Prerequisites

- Python 3.10+
- [Ollama](https://ollama.ai) installed and running (`ollama serve`)
- Target embedding model pulled locally (e.g., `ollama pull nomic-embed-text`)
- Open Brain Supabase project with the `thoughts` table

## Common Tasks

### Start Ollama and Pull a Model

```bash
# Start the Ollama server (runs in background)
ollama serve

# Pull the default model
ollama pull nomic-embed-text

# Or pull a 1536-dim model that matches the default Open Brain schema
ollama pull rjmalagon/gte-qwen2-1.5b-instruct-embed-f16
```

### Test Connectivity Before a Full Run

```bash
python embed-local.py "test thought" --dry-run --verbose
```

### Embed a Large File in One Command

```bash
cat my-notes.txt | python embed-local.py --source my-notes --verbose
```

### Use a Non-Default Embedding Dimension

The default Open Brain schema stores `vector(1536)`. If you use `nomic-embed-text` (768-dim) or `mxbai-embed-large` (1024-dim), run this SQL against your Supabase project first:

```sql
-- Example: switch to 768-dim for nomic-embed-text
ALTER TABLE thoughts ALTER COLUMN embedding TYPE vector(768);
```

The script will print a reminder when a mismatch is detected.

## Troubleshooting

| Symptom | Cause | Solution |
|---------|-------|----------|
| `Cannot reach Ollama at http://localhost:11434` | Ollama server is not running | Run `ollama serve` in a separate terminal |
| `Ollama embedding failed (404)` | Model not pulled locally | Run `ollama pull <model-name>` |
| `SUPABASE_URL environment variable required` | `.env` not loaded | Run `export $(cat .env | xargs)` |
| Supabase insert returns HTTP 400 | Embedding dimension mismatch | Alter the `embedding` column type to match the model's dimensions (see above) |
| Empty output / no thoughts processed | Input file has no non-blank lines | Check file encoding and line endings |

## Related

- [CONTEXT.md](CONTEXT.md) — Architecture context
- [../../primitives/README.md](../../primitives/README.md) — Reusable primitives
