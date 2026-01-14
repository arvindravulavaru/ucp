# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## About This Project

Universal Commerce Protocol (UCP) is an open standard for commerce interoperability. This repository contains:

- JSON Schema definitions for commerce primitives (checkout, payments, orders)
- OpenRPC specifications for the Embedded Commerce Protocol (ECP)
- Documentation site built with MkDocs
- Schema generation and validation tooling
- Interactive learning materials for UCP concepts

## Learning Materials

The `ucp-learning/` directory contains interactive tools for learning UCP:

- **ucp-learning-mindmap.html** - Standalone interactive accordion tree view covering:
  - 11 major sections of UCP (Foundations, Discovery, Shopping, Protocols, Security, etc.)
  - 53 topics with detailed explanations
  - Features: search, progress tracking, deep linking, localStorage persistence
- **ucp-learning-data.json** - Structured learning data (reusable for multiple view types)

To use: Open `ucp-learning/ucp-learning-mindmap.html` directly in a browser (no server required).

## Core Architecture

### Schema Source System (`source/` → `spec/`)

The codebase uses a **source-to-spec generation model**:

- **Source schemas** (`source/`) contain annotated JSON schemas with UCP-specific metadata
- **Generated specs** (`spec/`) are auto-generated per-operation schemas (create_req, update_req, response variants)
- **Annotations**: `ucp_request`, `ucp_response`, `ucp_shared_request` control how fields appear in different operations

**Key directories:**

- `source/schemas/shopping/types/` - Core type definitions (buyer, item, fulfillment, etc.)
- `source/services/shopping/` - Service definitions including embedded.json for ECP
- `spec/` - Generated output (committed to git, never edit manually)
- `generated/` - Generated TypeScript types

**Generation workflow:**

1. Edit schemas in `source/`
2. Run `python generate_schemas.py` to generate `spec/`
3. Verify generated outputs match expectations
4. If introducing new generation patterns, extend `generate_schemas.py`

### UCP Annotations

Source schemas use custom annotations to control per-operation variants:

- `"ucp_request": "omit|optional|required"` - Controls field behavior in create/update requests
- `"ucp_response": "omit"` - Omits field from responses
- `"ucp_shared_request": "optional|required"` - Applies to both create and update

Example: A field marked `"ucp_request": "omit"` won't appear in create_req.json or update_req.json but will appear in the response schema.

### Documentation System

MkDocs with custom macros (`main.py`) that generate API documentation from schemas:

- `schema_fields` - Generates tables from JSON schemas
- `method_fields` - Generates tables from OpenAPI/OpenRPC methods

The docs pull directly from `spec/` schemas, so schema changes automatically propagate to documentation.

## Common Commands

### Schema Development

```bash
# Generate spec/ from source/ schemas
python generate_schemas.py

# Validate all JSON/YAML in spec/
python validate_specs.py

# Generate TypeScript types (for SDK development)
npm install
# TypeScript generation commands in package.json
```

### Documentation

```bash
# Install dependencies
pip install -r requirements-docs.txt

# Serve docs locally with auto-reload
mkdocs serve --watch spec

# Build docs (strict mode - fails on warnings)
mkdocs build --strict
```

### Pre-commit Hooks

```bash
# Install hooks (runs spell check, trailing whitespace, yaml validation)
pip install pre-commit
pre-commit install

# Run manually
pre-commit run --all-files
```

### Python SDK Model Generation

```bash
# CI checks if models can be generated from schemas
bash scripts/ci_check_models.sh

# The script uses uv and datamodel-code-generator to create Pydantic models
```

## Code Quality

### Linting

- **Ruff** for Python (configured in `pyproject.toml`)
  - Line length: 80 chars
  - 2-space indent
  - Double quotes
  - Enforces: E, F, W, B, C4, SIM, N, UP, D, PTH, T20 rule sets

### Spell Checking

- **cspell** via pre-commit hooks
- Checks both file content and commit messages
- Dictionary: `.cspell/`

### Commit Messages

- Must follow **Conventional Commits** format: `type: description`
- Breaking changes: `type!: description`
- Common types: `feat`, `fix`, `docs`, `chore`, `refactor`, `test`, `perf`, `style`
- CI enforces this on PR titles

## Important Development Patterns

### Never Edit `spec/` Directly

The `spec/` directory is generated. Always edit `source/` schemas and regenerate.

### Schema References

- Internal references: `#/definitions/foo` (within same file)
- Cross-file references: `types/buyer.json#` or `../handlers/foo.json#`
- Generated schemas rewrite refs to point to appropriate variants (e.g., `buyer_resp.json#`)

### Adding New Schema Types

1. Create annotated schema in `source/schemas/shopping/types/`
2. Add UCP annotations as needed
3. Run `generate_schemas.py`
4. Update documentation if needed
5. Check that TypeScript types generate correctly
6. May need to extend generation logic in `generate_schemas.py` for new patterns

### Embedded Commerce Protocol (ECP)

- Defined in `source/services/shopping/embedded.json`
- Pass 3 of `generate_schemas.py` generates the final OpenRPC spec
- Aggregates methods and extension schemas
- Version is defined as `ECP_VERSION` in `generate_schemas.py`

## CI/CD

### GitHub Actions

- **Linter** (super-linter) - Validates code quality, configured in `.github/linters/`
- **Docs** - Builds docs with `mkdocs build --strict`
- **Conventional Commits** - Validates PR titles
- **Spellcheck** - Runs cspell on all files

### Pull Request Requirements

1. Pass all CI checks (linter, docs build, spell check, conventional commits)
2. CLA signed (Google CLA required)
3. PR title follows Conventional Commits format
4. If schemas changed, verify TypeScript types can be generated

## File Organization Reference

```
source/                    # Source schemas (edit these)
├── discovery/             # Profile and discovery schemas
├── schemas/shopping/      # Commerce domain schemas
│   └── types/            # Reusable type definitions
└── services/shopping/     # Service definitions (ECP)

spec/                      # Generated schemas (don't edit)
├── discovery/
├── schemas/shopping/
└── services/shopping/

docs/                      # MkDocs documentation
├── documentation/         # Guides and concepts
├── specification/         # Auto-generated API docs
└── index.md

ucp-learning/              # Interactive learning materials
├── ucp-learning-mindmap.html   # Standalone accordion tree view
└── ucp-learning-data.json      # Structured learning data

generated/                 # Generated SDKs (TypeScript, etc.)
```

## Testing Philosophy

When making changes:

1. Run `python validate_specs.py` after schema generation
2. Run `mkdocs build --strict` to catch broken doc links
3. Check `scripts/ci_check_models.sh` passes for schema changes
4. Use pre-commit hooks to catch formatting issues early

## Python Environment

Recommended: Use virtual environment to avoid polluting global Python:

```bash
python3 -m venv .ucp
source .ucp/bin/activate
pip install -r requirements-docs.txt
```

Prefix venv with `.` so git ignores it (already in .gitignore patterns).
