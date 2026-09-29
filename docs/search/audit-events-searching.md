# Searching Audit Events

Source: https://developer.sailpoint.com/docs/guides/searching-audit-events

Identity Security Cloud automatically records platform activity — logins, role modifications, access requests, workflow executions, and more — as indexable audit events queryable through the Search API. This file covers how to query them programmatically. For the exhaustive list of event names/types, see [audit-events-reference.md](audit-events-reference.md). For the `events` index field reference, see [searchable-fields.md](searchable-fields.md#events-events-index).

## Endpoint

```
POST /search/v1
```

with `"indices": ["events"]` in the request body. (Per the parent skill's [API versioning note](../../SKILL.md), this is the current per-resource-versioned path — do not use the deprecated `/v2026/search`.)

## Key searchable fields on events

| Field | Description |
|---|---|
| `action` | The event identifier, e.g. `WORKFLOW_CREATED` |
| `type` | Event category, e.g. `WORKFLOW`, `CERTIFICATION`, `AUTH` |
| `name` | Human-readable display name |
| `technicalName` | Internal identifier — sometimes distinct from `action` |
| `operation` | Action type: `CREATE`, `UPDATE`, `DELETE`, `ENABLE`, `DISABLE` |
| `status` | Result: `PASSED`, `FAILED`, `STARTED` |
| `actor.name` | User or system that initiated the event |
| `target.name` | Object affected by the action |
| `created` | ISO-8601 timestamp |
| `trackingNumber` | Correlates related events together |

Most audit events are queryable by `action`; a subset are queryable by `technicalName` instead — check [audit-events-reference.md](audit-events-reference.md) for which field a specific event uses.

## Query DSL mode

For queries beyond simple field-term matching, set `"queryType": "DSL"` and provide `"queryDsl"` using OpenSearch/Elasticsearch Query DSL syntax (`match`, `bool`, `range`, etc.) instead of the simplified query-string syntax described in [building-query.md](building-query.md).

### Filter by a single field

```json
{
  "indices": ["events"],
  "queryType": "DSL",
  "queryDsl": {
    "match": { "action": "WORKFLOW_CREATED" }
  }
}
```

```json
{
  "indices": ["events"],
  "queryType": "DSL",
  "queryDsl": {
    "match": { "technicalName": "ROLE_CREATE_PASSED" }
  }
}
```

```json
{
  "indices": ["events"],
  "queryType": "DSL",
  "queryDsl": {
    "match": { "type": "CERTIFICATION" }
  }
}
```

```json
{
  "indices": ["events"],
  "queryType": "DSL",
  "queryDsl": {
    "match": { "actor.name": "jane.doe" }
  }
}
```

### Date range filtering (combine with `bool`/`must`)

```json
{
  "indices": ["events"],
  "queryType": "DSL",
  "queryDsl": {
    "bool": {
      "must": [
        { "match": { "action": "WORKFLOW_CREATED" } },
        {
          "range": {
            "created": {
              "gte": "2026-01-01T00:00:00Z",
              "lte": "2026-06-30T23:59:59Z"
            }
          }
        }
      ]
    }
  }
}
```

Relative date math also works in range filters, e.g. `"gte": "now-30d/d"` (last 30 days).

### Match multiple values for one field (OR logic)

Use `bool.should` with `minimum_should_match`:

```json
{
  "indices": ["events"],
  "queryType": "DSL",
  "queryDsl": {
    "bool": {
      "should": [
        { "match": { "action": "WORKFLOW_CREATED" } },
        { "match": { "action": "WORKFLOW_DELETED" } },
        { "match": { "action": "WORKFLOW_ENABLED" } },
        { "match": { "action": "WORKFLOW_DISABLED" } }
      ],
      "minimum_should_match": 1
    }
  }
}
```

### Combine multiple required filters (AND logic)

Use `bool.must`:

```json
{
  "indices": ["events"],
  "queryType": "DSL",
  "queryDsl": {
    "bool": {
      "must": [
        { "match": { "type": "WORKFLOW" } },
        { "match": { "status": "PASSED" } },
        { "match": { "actor.name": "john.doe" } },
        { "range": { "created": { "gte": "now-30d/d" } } }
      ]
    }
  }
}
```

### Pagination and sorting

```json
{
  "indices": ["events"],
  "queryType": "DSL",
  "queryDsl": {
    "match": { "type": "CERTIFICATION" }
  },
  "sort": ["-created"],
  "limit": 50,
  "offset": 0
}
```

- `limit` — results per page
- `offset` — 0-indexed starting position
- `sort` — array of field names; prefix with `-` for descending (`"-created"` = newest first)

## Common patterns

**Compliance — access request approvals by a specific approver:**

```json
{
  "indices": ["events"],
  "queryType": "DSL",
  "queryDsl": {
    "bool": {
      "must": [
        { "match": { "type": "ACCESS_REQUEST" } },
        { "match": { "action": "ACCESS_REQUEST_APPROVED" } },
        { "match": { "actor.name": "alice.jones" } }
      ]
    }
  },
  "sort": ["-created"]
}
```

**Security — recent authentication failures:**

```json
{
  "indices": ["events"],
  "queryType": "DSL",
  "queryDsl": {
    "bool": {
      "must": [
        { "match": { "type": "AUTH" } },
        { "match": { "status": "FAILED" } },
        { "range": { "created": { "gte": "now-1d/d" } } }
      ]
    }
  },
  "sort": ["-created"],
  "limit": 250
}
```

**Audit trail — all lifecycle events for a specific role (mixing `action` and `technicalName`):**

```json
{
  "indices": ["events"],
  "queryType": "DSL",
  "queryDsl": {
    "bool": {
      "must": [
        { "match": { "target.name": "test-role" } },
        {
          "bool": {
            "should": [
              { "match": { "action": "ROLE_CREATED" } },
              { "match": { "action": "ROLE_UPDATED" } },
              { "match": { "action": "ROLE_DELETED" } },
              { "match": { "technicalName": "ROLE_CREATE_PASSED" } },
              { "match": { "technicalName": "ROLE_UPDATE_PASSED" } },
              { "match": { "technicalName": "ROLE_DELETE_PASSED" } }
            ],
            "minimum_should_match": 1
          }
        }
      ]
    }
  },
  "sort": ["-created"]
}
```

**Workflow monitoring — any workflow lifecycle/execution change:**

```json
{
  "indices": ["events"],
  "queryType": "DSL",
  "queryDsl": {
    "bool": {
      "should": [
        { "match": { "action": "WORKFLOW_CREATED" } },
        { "match": { "action": "WORKFLOW_DELETED" } },
        { "match": { "action": "WORKFLOW_ENABLED" } },
        { "match": { "action": "WORKFLOW_DISABLED" } },
        { "match": { "action": "WORKFLOW_EXECUTION_CANCELLED" } }
      ],
      "minimum_should_match": 1
    }
  },
  "sort": ["-created"],
  "limit": 100
}
```

## Notes

- The full Search API reference (auth, rate limits, full request/response schema) lives with the general Search API docs, not this audit-specific guide — see the parent skill's [SKILL.md](../../SKILL.md) Key Endpoints table for `/search/v1` usage in the `sail` CLI context.
- For simple single-field lookups, the plain query-string syntax from [building-query.md](building-query.md) (e.g. `sail search query "action:WORKFLOW_CREATED" --indices events`) works fine — reach for DSL mode specifically when you need `bool`/`range`/`should` composition that the query-string syntax can't express cleanly.
