# personal

Personal Claude Code plugin for dodi workflows - frontend development, Supabase, and GitHub automation.

## Installation

### From GitHub (Recommended)

First, add the marketplace:

```bash
/plugin marketplace add azlekov/my-claude-code
```

Then install the plugin:

```bash
/plugin install personal@azlekov/my-claude-code
```

Or use the interactive plugin manager:

```bash
/plugin
```

Navigate to the **Discover** tab to browse and install plugins.

### Update Plugin

Plugins auto-update by default. To manually refresh:

```bash
/plugin marketplace update azlekov/my-claude-code
```

### Local Development

For local development, load the plugin directly:

```bash
claude --plugin-dir /path/to/claude-code-plugin
```

## Components

### Skills

| Skill | Description |
|-------|-------------|
| `react` | Modern React development with latest hooks and patterns |
| `nextjs` | Next.js App Router, Server Components, and caching |
| `tailwindcss` | Tailwind CSS v4 with @theme directive and OKLCH colors |
| `shadcn` | shadcn/ui components, forms, and theming |
| `supabase-expert` | Supabase Auth SSR, RLS policies, and Edge Functions |
| `postgres-nanoid` | Contextual nanoid identifiers with prefixes for PostgreSQL |

### Agents

| Agent | Description |
|-------|-------------|
| `frontend-developer` | Senior frontend developer specializing in Next.js, React, Tailwind, and shadcn/ui |
| `fullstack-developer` | Full-stack development with frontend skills + Supabase + nanoid identifiers |

### Commands

| Command | Description |
|---------|-------------|
| `/personal:work-on-github-issue <issue>` | Analyze and work on a GitHub issue end-to-end |
| `/personal:add-supabase <framework>` | Full guided Supabase setup (nextjs, kotlin, ios, flutter) |

### MCP Servers

| Server | Description |
|--------|-------------|
| `supabase` | Supabase MCP for database introspection and management |
| `trigger` | Trigger.dev MCP for background jobs and task management |
| `context7` | Context7 MCP for up-to-date library documentation |

## Configuration

### Environment Variables

Set the following environment variables for MCP servers:

```bash
# Supabase (required for supabase MCP)
export SUPABASE_ACCESS_TOKEN="your-supabase-token"

# Context7 (optional, for higher rate limits)
export CONTEXT7_API_KEY="your-context7-key"

# Trigger.dev authenticates via CLI when needed
```

### Getting Tokens

- **Supabase**: Get from [Supabase Dashboard](https://supabase.com/dashboard/account/tokens)
- **Context7**: Get from [Context7 Dashboard](https://context7.com/dashboard)
- **Trigger.dev**: Authenticates interactively via CLI

## Nanoid Prefix Conventions

The `postgres-nanoid` skill uses contextual prefixes for public identifiers:

| Entity | Prefix | Example |
|--------|--------|---------|
| User | `usr_` | `usr_V1StGXR8_Z5jdHi` |
| Organization | `org_` | `org_kJ7mNpQ2xWzL9aB` |
| Order | `ord_` | `ord_xYz7aBcDeF2gH5i` |
| Product | `prd_` | `prd_mN3kL9pQwE7rT4u` |
| Invoice | `inv_` | `inv_9sK3pLmNqR5tU8v` |

**Rule**: Use nanoid for public IDs (APIs, URLs), UUID for auth.users references.

## License

MIT
