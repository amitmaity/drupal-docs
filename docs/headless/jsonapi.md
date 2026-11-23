# JSON:API Deep Dive

**JSON:API** is Drupal's primary headless API. It ships in core, follows the [JSON:API specification](https://jsonapi.org/), and auto-exposes enabled entity types with filtering, sorting, pagination, sparse fieldsets, and relationship includes.

## Enable and discover

```bash
drush en jsonapi -y
curl -s https://my-site.ddev.site/jsonapi | jq '.links | keys'
```

The entry point at `/jsonapi` has an empty `data` array and lists every available resource type under `links`, keyed by resource type name. For authenticated requests it also adds `meta.links.me`, which points to the current user's resource:

```json
{
  "jsonapi": {
    "version": "1.1",
    "meta": {
      "links": {
        "self": { "href": "http://jsonapi.org/format/1.1/" }
      }
    }
  },
  "data": [],
  "meta": {
    "links": {
      "me": {
        "meta": { "id": "current-user-uuid" },
        "href": "https://my-site.ddev.site/jsonapi/user/user/current-user-uuid"
      }
    }
  },
  "links": {
    "node--article": { "href": "https://my-site.ddev.site/jsonapi/node/article" },
    "taxonomy_term--tags": { "href": "https://my-site.ddev.site/jsonapi/taxonomy_term/tags" },
    "self": { "href": "https://my-site.ddev.site/jsonapi" }
  }
}
```

## Endpoints

Resource URLs follow the pattern:

```
/jsonapi/{entity_type_id}/{bundle_id}
/jsonapi/{entity_type_id}/{bundle_id}/{uuid}
```

| Entity | Collection | Individual |
|--------|------------|------------|
| Article nodes | `/jsonapi/node/article` | `/jsonapi/node/article/{uuid}` |
| Users | `/jsonapi/user/user` | `/jsonapi/user/user/{uuid}` |
| Tags | `/jsonapi/taxonomy_term/tags` | `/jsonapi/taxonomy_term/tags/{uuid}` |
| Media (image) | `/jsonapi/media/image` | `/jsonapi/media/image/{uuid}` |
| Files | `/jsonapi/file/file` | `/jsonapi/file/file/{uuid}` |

Always send the correct `Accept` header:

```bash
curl -s \
  -H "Accept: application/vnd.api+json" \
  https://my-site.ddev.site/jsonapi/node/article
```

## Response structure

```json
{
  "jsonapi": { "version": "1.1" },
  "data": [
    {
      "type": "node--article",
      "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "attributes": {
        "drupal_internal__nid": 42,
        "title": "Hello JSON:API",
        "body": {
          "value": "<p>Content here</p>",
          "format": "basic_html",
          "processed": "<p>Content here</p>"
        },
        "created": "2026-01-15T10:30:00+00:00",
        "status": true
      },
      "relationships": {
        "field_author": {
          "data": { "type": "user--user", "id": "user-uuid-here" },
          "links": {
            "related": { "href": "..." },
            "self": { "href": "..." }
          }
        }
      },
      "links": {
        "self": { "href": "https://my-site.ddev.site/jsonapi/node/article/uuid" }
      }
    }
  ],
  "links": {
    "self": { "href": "..." },
    "next": { "href": "..." }
  }
}
```

## Filtering

Use the `filter` query parameter with condition groups.

### Simple equality

```bash
# Published articles only
curl -s -G \
  -H "Accept: application/vnd.api+json" \
  --data-urlencode 'filter[status]=1' \
  https://my-site.ddev.site/jsonapi/node/article
```

### Filter operators

```bash
# Title contains "Drupal"
curl -s -G \
  -H "Accept: application/vnd.api+json" \
  --data-urlencode 'filter[title-filter][condition][path]=title' \
  --data-urlencode 'filter[title-filter][condition][operator]=CONTAINS' \
  --data-urlencode 'filter[title-filter][condition][value]=Drupal' \
  https://my-site.ddev.site/jsonapi/node/article

# Created after a date (created/changed are timestamp fields: filter with a
# Unix timestamp — 1767225600 is 2026-01-01T00:00:00Z)
curl -s -G \
  -H "Accept: application/vnd.api+json" \
  --data-urlencode 'filter[recent][condition][path]=created' \
  --data-urlencode 'filter[recent][condition][operator]=>' \
  --data-urlencode 'filter[recent][condition][value]=1767225600' \
  https://my-site.ddev.site/jsonapi/node/article
```

!!! note "Timestamp vs date fields"
    Timestamp fields such as `created` and `changed` must be filtered with a Unix timestamp. ISO 8601 strings only work for Date (`datetime`) fields.

### Filter groups (AND / OR)

```bash
# (status = 1) AND (title contains "API")
curl -s -G \
  -H "Accept: application/vnd.api+json" \
  --data-urlencode 'filter[published][condition][path]=status' \
  --data-urlencode 'filter[published][condition][value]=1' \
  --data-urlencode 'filter[title][condition][path]=title' \
  --data-urlencode 'filter[title][condition][operator]=CONTAINS' \
  --data-urlencode 'filter[title][condition][value]=API' \
  --data-urlencode 'filter[title][condition][memberOf]=published-group' \
  --data-urlencode 'filter[published][condition][memberOf]=published-group' \
  --data-urlencode 'filter[published-group][group][conjunction]=AND' \
  https://my-site.ddev.site/jsonapi/node/article
```

### Filter by relationship

```bash
# Articles by author user ID 5
curl -s -G \
  -H "Accept: application/vnd.api+json" \
  --data-urlencode 'filter[author][condition][path]=field_author.meta.drupal_internal__target_id' \
  --data-urlencode 'filter[author][condition][value]=5' \
  https://my-site.ddev.site/jsonapi/node/article
```

## Sorting

```bash
# Newest first
curl -s -G \
  -H "Accept: application/vnd.api+json" \
  --data-urlencode 'sort=-created' \
  https://my-site.ddev.site/jsonapi/node/article

# Title ascending, then created descending
curl -s -G \
  -H "Accept: application/vnd.api+json" \
  --data-urlencode 'sort=title,-created' \
  https://my-site.ddev.site/jsonapi/node/article
```

## Includes (relationships)

Load related entities in a single request:

```bash
curl -s -G \
  -H "Accept: application/vnd.api+json" \
  --data-urlencode 'include=field_author,field_tags,field_hero_media.field_media_image' \
  https://my-site.ddev.site/jsonapi/node/article/a1b2c3d4-uuid
```

Response adds an `included` array:

```json
{
  "data": { "...": "primary resource" },
  "included": [
    {
      "type": "user--user",
      "id": "author-uuid",
      "attributes": { "display_name": "Jane Editor" }
    },
    {
      "type": "media--image",
      "id": "media-uuid",
      "relationships": {
        "field_media_image": {
          "data": { "type": "file--file", "id": "file-uuid" }
        }
      }
    }
  ]
}
```

!!! tip "Deep includes"
    Chain relationship paths with dots: `field_hero_media.field_media_image` pulls the file entity behind a media reference.

## Sparse fieldsets

Reduce payload size by requesting only specific fields:

```bash
# Only title and created on articles
curl -s -G \
  -H "Accept: application/vnd.api+json" \
  --data-urlencode 'fields[node--article]=title,created,path' \
  https://my-site.ddev.site/jsonapi/node/article

# Sparse fields on included resources too
curl -s -G \
  -H "Accept: application/vnd.api+json" \
  --data-urlencode 'include=field_author' \
  --data-urlencode 'fields[node--article]=title,field_author' \
  --data-urlencode 'fields[user--user]=display_name,mail' \
  https://my-site.ddev.site/jsonapi/node/article
```

## Pagination

Drupal JSON:API uses offset/limit pagination:

```bash
# First page (50 items — default page size)
curl -s -G \
  -H "Accept: application/vnd.api+json" \
  --data-urlencode 'page[offset]=0' \
  --data-urlencode 'page[limit]=10' \
  https://my-site.ddev.site/jsonapi/node/article

# Second page
curl -s -G \
  -H "Accept: application/vnd.api+json" \
  --data-urlencode 'page[offset]=10' \
  --data-urlencode 'page[limit]=10' \
  https://my-site.ddev.site/jsonapi/node/article
```

Follow `links.next` in the response for cursor-style iteration:

```typescript
async function fetchAllArticles(baseUrl: string): Promise<unknown[]> {
  const items: unknown[] = [];
  let url: string | null = `${baseUrl}/jsonapi/node/article?page[limit]=50`;

  while (url) {
    const response = await fetch(url, {
      headers: { Accept: 'application/vnd.api+json' },
    });
    const json = await response.json();
    items.push(...json.data);
    url = json.links?.next?.href ?? null;
  }

  return items;
}
```

## POST — create resources

!!! warning "JSON:API is read-only by default"
    Core ships `jsonapi.settings:read_only: true`, so every POST, PATCH and DELETE below returns `405 Method Not Allowed` until you allow writes (see [Read-only mode](#read-only-mode)):

    ```bash
    drush config:set jsonapi.settings read_only false -y
    ```

    The `-u admin:admin` examples also need the core `basic_auth` module enabled (local development only).

```bash
curl -s -X POST \
  -u admin:admin \
  -H "Accept: application/vnd.api+json" \
  -H "Content-Type: application/vnd.api+json" \
  https://my-site.ddev.site/jsonapi/node/article \
  -d '{
    "data": {
      "type": "node--article",
      "attributes": {
        "title": "New Article via JSON:API",
        "body": {
          "value": "<p>Created with POST</p>",
          "format": "basic_html"
        }
      },
      "relationships": {
        "field_tags": {
          "data": [
            { "type": "taxonomy_term--tags", "id": "tag-uuid-1" }
          ]
        }
      }
    }
  }'
```

Required permissions:

- JSON:API has no permissions of its own (the `restful *` permissions belong to the REST module and are not used here)
- The authenticated user needs the normal entity permission, e.g. `create article content`

## PATCH — update resources

```bash
curl -s -X PATCH \
  -u admin:admin \
  -H "Accept: application/vnd.api+json" \
  -H "Content-Type: application/vnd.api+json" \
  https://my-site.ddev.site/jsonapi/node/article/a1b2c3d4-uuid \
  -d '{
    "data": {
      "type": "node--article",
      "id": "a1b2c3d4-uuid",
      "attributes": {
        "title": "Updated Title"
      }
    }
  }'
```

!!! note "PATCH requires id"
    The `id` in the payload must match the UUID in the URL.

## DELETE

```bash
curl -s -X DELETE \
  -u admin:admin \
  -H "Accept: application/vnd.api+json" \
  https://my-site.ddev.site/jsonapi/node/article/a1b2c3d4-uuid
```

Returns `204 No Content` on success.

## Read-only mode

Read-only mode is the **default** in core: only GET requests are accepted. Keep it on for read-only frontends, and turn it off only on sites that need mutations through JSON:API:

```yaml
# config/sync/jsonapi.settings.yml
read_only: true   # default; set to false to accept POST, PATCH and DELETE
```

Or via Drush:

```bash
# Allow writes
drush config:set jsonapi.settings read_only false -y

# Lock down again
drush config:set jsonapi.settings read_only true -y
```

With writes enabled, entity access still applies: each request needs a user with the matching create/edit/delete permission.

## JSON:API Extras (contrib)

Customize resource names, disable fields, and override normalizers:

```bash
composer require drupal/jsonapi_extras
drush en jsonapi_extras -y
```

Example: rename `node--article` to `articles`:

```yaml
# jsonapi_extras.jsonapi_resource_config.node--article.yml
id: node--article
disabled: false
path: articles
resourceType: articles
resourceFields:
  title:
    disabled: false
    fieldName: title
    publicName: title
```

Now available at `/jsonapi/articles`.

## TypeScript client example

```typescript
import type { JsonApiDocument, JsonApiResource } from './types';

const DRUPAL_BASE = process.env.NEXT_PUBLIC_DRUPAL_BASE_URL!;

export async function getArticles(): Promise<JsonApiResource[]> {
  const params = new URLSearchParams({
    'filter[status]': '1',
    'sort': '-created',
    'page[limit]': '20',
    'fields[node--article]': 'title,created,path,field_author',
    'include': 'field_author',
  });

  const response = await fetch(
    `${DRUPAL_BASE}/jsonapi/node/article?${params}`,
    {
      headers: { Accept: 'application/vnd.api+json' },
      next: { tags: ['node_list:article'] },
    },
  );

  if (!response.ok) {
    throw new Error(`JSON:API error: ${response.status}`);
  }

  const doc: JsonApiDocument = await response.json();
  return doc.data as JsonApiResource[];
}
```

## Permissions checklist

JSON:API defines no permissions; it applies normal entity access to every request. For article nodes:

| Action | Permission |
|--------|------------|
| Read collection | Entity view access (`access content` for published nodes); items the user cannot view are omitted |
| Read individual | Entity view access (`access content`; `view own unpublished content` etc. for drafts) |
| Create | `create article content` (and `read_only: false`) |
| Update | `edit own article content` or `edit any article content` (and `read_only: false`) |
| Delete | `delete own article content` or `delete any article content` (and `read_only: false`) |

## Related pages

- [REST API](rest-api.md) — custom REST resource plugins
- [Custom JSON APIs](custom-json-apis.md) — controllers and normalizers
- [Authentication](auth/index.md) — OAuth, JWT, cookies
- [Cache Invalidation](cache-invalidation.md) — tag propagation
