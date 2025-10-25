# OpenCode - Web Console

> **Web-based interface and documentation site**

---

## Overview

OpenCode Web includes:
- **Documentation site** - Built with Astro + Starlight
- **Web console** - Browser-based interface (planned)
- **API documentation** - Interactive API explorer

**Tech Stack**:
- **Framework**: Astro
- **Docs**: Starlight
- **Database**: Drizzle ORM + PlanetScale
- **Auth**: OAuth providers
- **Deployment**: SST

**Package**: `packages/web`, `packages/console`

---

## Documentation Site

### Structure

```
packages/web/
├── src/
│   ├── content/
│   │   └── docs/           # Markdown docs
│   │       ├── index.md
│   │       ├── getting-started.md
│   │       ├── configuration.md
│   │       └── api/
│   │           └── *.md
│   │
│   ├── components/         # Custom components
│   └── pages/              # Astro pages
│
├── astro.config.mjs
└── package.json
```

### Development

```bash
cd packages/web
bun install
bun run dev
```

Opens at http://localhost:4321

### Building

```bash
bun run build
```

Static site in `dist/`

---

## Web Console (Future)

### Planned Features

**UI**:
- Browser-based chat interface
- Code editor integration
- File browser
- Session management

**Collaboration**:
- Shared sessions
- Team workspaces
- Real-time updates

**Cloud**:
- Hosted service
- API key management
- Usage tracking
- Billing

---

## Console Package

**Structure**:
```
packages/console/
├── app/                    # Frontend
├── core/                   # Shared logic
├── function/               # Serverless functions
├── mail/                   # Email templates
└── resource/               # SST resources
```

---

## Database

### Schema (Drizzle)

```typescript
// User
export const users = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  email: text("email").notNull().unique(),
  name: text("name"),
  createdAt: timestamp("created_at").defaultNow(),
})

// Session
export const sessions = pgTable("sessions", {
  id: uuid("id").primaryKey().defaultRandom(),
  userId: uuid("user_id").references(() => users.id),
  provider: text("provider").notNull(),
  model: text("model").notNull(),
  createdAt: timestamp("created_at").defaultNow(),
})

// Message
export const messages = pgTable("messages", {
  id: uuid("id").primaryKey().defaultRandom(),
  sessionId: uuid("session_id").references(() => sessions.id),
  role: text("role").notNull(),
  content: text("content").notNull(),
  tokens: integer("tokens"),
  cost: numeric("cost"),
  createdAt: timestamp("created_at").defaultNow(),
})
```

### Migrations

```bash
# Generate migration
bun run drizzle-kit generate

# Run migrations
bun run drizzle-kit migrate
```

---

## Authentication

### OAuth Providers

- GitHub
- Google
- Email (magic link)

### Implementation

```typescript
import { auth } from "@/auth"

export const GET = async (req: Request) => {
  const session = await auth.getSession(req)
  if (!session) {
    return Response.redirect("/login")
  }
  
  return Response.json({ user: session.user })
}
```

---

## Deployment

### SST Configuration

**sst.config.ts**:
```typescript
export default $config({
  app: "opencode",
  providers: {
    stripe: true,
    planetscale: true,
  }
})
```

### Infrastructure

**infra/console.ts**:
```typescript
const database = new sst.planetscale.Database("Database")

const web = new sst.aws.StaticSite("Web", {
  path: "packages/web",
  build: {
    command: "bun run build",
    output: "dist",
  }
})

const api = new sst.aws.Function("API", {
  handler: "packages/console/function/api.handler",
  link: [database],
})
```

### Deploy

```bash
# Deploy to dev
sst deploy

# Deploy to production
sst deploy --stage production
```

---

## Best Practices

**Documentation**:
- Keep docs in sync with features
- Include code examples
- Provide troubleshooting guides
- Version documentation

**Web Console**:
- Progressive enhancement
- Mobile-responsive
- Fast page loads
- Accessibility

**Security**:
- Secure authentication
- Rate limiting
- Input validation
- HTTPS only

---

For implementation, see `packages/web/` and `packages/console/`.

