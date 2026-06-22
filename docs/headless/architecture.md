# Headless Architecture

Choosing the right decoupling strategy affects team structure, hosting, caching, and how much Drupal code you maintain. This page compares the three common patterns with diagrams and implementation notes.

## Three architecture patterns

### Traditional (coupled)

Drupal owns routing, business logic, and HTML rendering via Twig themes.

```mermaid
graph TB
    Browser[Browser] --> Apache[Web Server]
    Apache --> Drupal[Drupal Monolith]
    Drupal --> Twig[Twig Theme Engine]
    Drupal --> Entity[Entity API]
    Entity --> DB[(Database)]
    Admin[Content Editor] --> Drupal
```

**Characteristics:**

- Single deployment unit
- Full page cache and render cache work out of the box
- Contrib themes and Layout Builder available
- No CORS or API auth complexity

### Decoupled (progressive)

Drupal serves the admin UI and APIs; a JavaScript framework renders the public site.

```mermaid
graph TB
  subgraph public [Public Site]
    Browser[Browser] --> NextJS[Next.js App]
    NextJS --> JSONAPI[JSON:API / REST]
  end

  subgraph cms [CMS]
    Editor[Content Editor] --> Admin[Drupal Admin UI]
    Admin --> Drupal[Drupal Backend]
    JSONAPI --> Drupal
    Drupal --> DB[(Database)]
  end
```

**Characteristics:**

- Editors use familiar Drupal admin
- Public frontend is independently deployable
- Two codebases, coordinated releases
- Preview and draft workflows need extra wiring

### Fully headless

Drupal has no public theme. All consumers—web, mobile, IoT—use APIs exclusively.

```mermaid
graph TB
    Web[Next.js Website] --> API[API Gateway / Drupal]
    Mobile[React Native] --> API
    Signage[Digital Signage] --> API
    Voice[Voice Assistant] --> API

    API --> Drupal[Drupal API Backend]
    Drupal --> DB[(Database)]
    Drupal --> Search[Search API / Solr]
    Drupal --> Queue[Queue / Cron]

    Editor[Content Team] --> Admin[Admin UI only]
    Admin --> Drupal
```

**Characteristics:**

- Maximum flexibility for multi-channel delivery
- No Twig maintenance for public pages
- Requires API-first content modeling from day one
- Preview, SEO, and caching are frontend responsibilities

## Comparison matrix

| Factor | Traditional | Decoupled | Fully Headless |
|--------|-------------|-----------|----------------|
| Public HTML | Drupal Twig | Frontend framework | Frontend only |
| Admin UI | Drupal | Drupal | Drupal (or custom) |
| API exposure | Optional | Required for FE | Required for all |
| Caching | Core page cache | Split (Drupal + CDN) | Frontend + API cache |
| Team skills | PHP + Twig | PHP + JavaScript | PHP + JS + DevOps |
| Time to MVP | Fastest | Medium | Slowest |
| Multi-channel | Limited | Good | Excellent |

## Request flow: headless read path

```mermaid
sequenceDiagram
    participant Client as Frontend
    participant CDN as CDN / Edge
    participant Drupal as Drupal
    participant Cache as Render / Dynamic Page Cache
    participant DB as Database

    Client->>CDN: GET /jsonapi/node/article
    alt CDN hit (with stale-while-revalidate)
        CDN-->>Client: Cached JSON
    else CDN miss
        CDN->>Drupal: Forward request
        Drupal->>Cache: Check cache ID
        alt Cache hit
            Cache-->>Drupal: Cached response
        else Cache miss
            Drupal->>DB: Load entities
            DB-->>Drupal: Entity data
            Drupal->>Drupal: Normalize to JSON:API
        end
        Drupal-->>CDN: Response + Cache-Control
        CDN-->>Client: JSON + cache tags header
    end
```

## Request flow: headless write path

```mermaid
sequenceDiagram
    participant Client as Frontend
    participant Drupal as Drupal
    participant Auth as Auth (OAuth/JWT)
    participant Entity as Entity API
    participant Queue as Queue

    Client->>Drupal: PATCH /jsonapi/node/article/uuid
    Drupal->>Auth: Validate token
    Auth-->>Drupal: User context
    Drupal->>Entity: Validate + save
    Entity->>Entity: Invalidate cache tags
    Entity->>Queue: Enqueue webhook job
    Queue-->>Client: 200 OK (async webhook)
```

## Layered architecture

```mermaid
graph TB
    subgraph presentation [Presentation Layer]
        Next[Next.js SSG/ISR]
        Mobile[Mobile App]
    end

    subgraph api [API Layer]
        JSONAPI[JSON:API]
        REST[REST Resources]
        GraphQL[GraphQL - contrib]
        Custom[Custom Controllers]
    end

    subgraph drupal [Drupal Application Layer]
        Controllers[Controllers / Resource Plugins]
        Normalizers[Serializer Normalizers]
        Access[Access Checkers]
        Entity[Entity API]
    end

    subgraph data [Data Layer]
        DB[(MariaDB / PostgreSQL)]
        Files[File System / S3]
        Redis[Redis Cache]
    end

    Next --> JSONAPI
    Mobile --> REST
    Next --> Custom
    JSONAPI --> Controllers
    REST --> Controllers
    GraphQL --> Controllers
    Custom --> Controllers
    Controllers --> Normalizers
    Controllers --> Access
    Access --> Entity
    Entity --> DB
    Entity --> Files
    Controllers --> Redis
```

## Hosting topology

A typical production headless stack:

```
┌─────────────────────────────────────────────────────────┐
│  CDN (Cloudflare / Fastly)                              │
│  ├── Static assets (Next.js on Vercel)                  │
│  └── API cache (optional, tag-aware)                    │
└─────────────────────────────────────────────────────────┘
         │                              │
         ▼                              ▼
┌─────────────────┐          ┌─────────────────────────┐
│  Vercel         │          │  Platform.sh / Acquia   │
│  Next.js 15     │  HTTPS   │  Drupal 11              │
│  ISR + webhooks │ ◄──────► │  JSON:API + OAuth       │
└─────────────────┘          └─────────────────────────┘
```

## Drupal modules by pattern

| Pattern | Core modules | Common contrib |
|---------|--------------|----------------|
| All headless | `jsonapi`, `serialization` | `jsonapi_extras`, `subrequests` |
| REST endpoints | `rest` | — |
| Auth | `basic_auth` (dev only) | `simple_oauth`, custom JWT |
| CORS | — (services.yml) | `cors` (if not using services.yml) |
| Preview | — | `next`, `decoupled_router` |
| GraphQL | — | `graphql`, `graphql_compose` |

## Configuration export for headless sites

Export API-related configuration with the rest of your site:

```bash
# Export all config
drush config:export -y

# Key config files for headless setups
ls config/sync/ | grep -E 'jsonapi|rest|cors|oauth'
```

Typical headless config entities:

- `jsonapi.settings.yml` — read-only mode, base path
- `rest.resource.*.yml` — REST resource configuration
- `user.role.api_consumer.yml` — API consumer role
- `simple_oauth.settings.yml` — OAuth settings

## Decision guide

```mermaid
flowchart TD
    Start[New project?] --> Multi{Multiple frontends<br/>or mobile app?}
    Multi -->|Yes| Headless[Fully headless]
    Multi -->|No| Interactive{Highly interactive<br/>public UI?}
    Interactive -->|Yes| Decoupled[Decoupled]
    Interactive -->|No| Brochure{Simple brochure<br/>site?}
    Brochure -->|Yes| Traditional[Traditional Drupal]
    Brochure -->|No| Decoupled
```

## Related pages

- [JSON:API Deep Dive](jsonapi.md) — primary API layer
- [Authentication](auth/index.md) — securing API access
- [Cache Invalidation](cache-invalidation.md) — keeping frontends fresh
- [Next.js integration](nextjs.md) — reference frontend architecture
