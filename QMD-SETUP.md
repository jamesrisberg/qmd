# QMD Setup (James's fork with AST symbol extraction)

## 1. Install

You need [Bun](https://bun.sh) first. If you don't have it:

```sh
curl -fsSL https://bun.sh/install | bash
```

Then install QMD:

```sh
git clone https://github.com/jamesrisberg/qmd.git
cd qmd
git checkout dev
bun install
bun link
```

Verify: `qmd status` should run without errors.

## 2. Index your project

```sh
# Add your repo as a collection (run from your project dir)
qmd collection add . --name my-project

# Add context so search understands what's in here
qmd context add qmd://my-project/ "Brief description of your project"

# Generate embeddings with AST-aware chunking for code files
qmd embed --chunk-strategy auto
```

The `--chunk-strategy auto` flag is what enables AST-aware chunking — it uses tree-sitter to split code files at function/class boundaries instead of arbitrary line counts. Without it you get basic regex chunking.

## 3. Connect to Claude Code

Add to your project's `.mcp.json` (or `~/.claude/settings.json` for global):

```json
{
  "mcpServers": {
    "qmd": {
      "command": "qmd",
      "args": ["mcp"]
    }
  }
}
```

Restart Claude Code. It will now have `query`, `get`, `multi_get`, and `status` tools.

## 4. Re-index after changes

```sh
qmd update                        # re-index files
qmd embed --chunk-strategy auto   # re-embed with AST chunking
```

---

## How agents should use QMD

### Search (`query` tool)

The `query` tool accepts typed sub-queries. Combine types for best results:

| Type | When to use | Input style |
|------|------------|-------------|
| `lex` | Know exact terms, names, identifiers | Keywords: `handleAuth session token` |
| `vec` | Conceptual/natural language question | Question: `how does authentication work` |
| `hyde` | Complex topic, want best recall | Write 50-100 words of what the answer looks like |

Example — searching for auth logic:
```json
{
  "searches": [
    { "type": "lex", "query": "authenticate session token" },
    { "type": "vec", "query": "how does user authentication work" }
  ],
  "limit": 10
}
```

First query gets 2x weight in fusion — put your best guess first.

Results include:
- File path and docid (`#abc123`)
- Score and best matching snippet
- **Symbols** — function/class/method signatures found in the matching chunk (e.g. `[function] authenticate(user: User): Promise<boolean>`)

### Retrieve (`get` / `multi_get`)

```json
// Single file by path or docid
{ "path": "src/auth.ts" }
{ "path": "#abc123" }

// Multiple files by glob
{ "pattern": "src/auth/**/*.ts" }
```

### Tips
- Use `lex` for exact names and code identifiers
- Use `vec` when you're not sure of the vocabulary
- Add `intent` field to disambiguate vague queries
- Use `get` with docids from search results to pull full files
- Check `status` to see what collections are indexed
