# Code Review Skills — se-demo-backend

This is the Go REST API backend for the Qodo demo suite. It serves a React frontend and persists data in Firestore under Firebase Auth user IDs.

## Primary Skill: Firestore Tenant Isolation

Every Firestore query that reads or writes user data must include a `Where("userId", "==", userID)` clause. A query without this clause leaks data across tenants — the bug is invisible until a user sees another user's records.

The correct pattern builds the query incrementally:
```go
q := s.col().Where("userId", "==", userID)
if f.Completed != nil {
    q = q.Where("completed", "==", *f.Completed)
}
```

Flag any handler that calls `.Documents(ctx)` without a userId filter on a shared collection.

## Skill: Ownership Check Before Mutation

Before any PUT, PATCH, or DELETE that touches a specific document, the handler must verify `doc.Data()["userId"] == userID`. Missing this check lets any authenticated user modify or delete another user's data.

Pattern to look for: `ref.Get(ctx)` followed by a mutation without checking the userId field.

## Skill: JSON Tag Stability

Go struct `json:"..."` tags are the source of truth for the API contract. Any rename to a json tag in `api/services/` is a **breaking change** — the frontend types in `se-demo-frontend/src/types/` and `se-demo-risk/frontend/src/types/risk.ts` mirror these exactly.

Flag any PR that renames a json tag without a confirmed matching frontend change.

## Skill: Boolean Query Param Parsing

The server parses boolean filter params as `b := v == "true"`. A client sending `1` or `yes` silently resolves to `false`. The canonical wire values are `"true"` and `"false"` only.

Handler pattern to enforce: `strconv.ParseBool` or explicit `v == "true"` check — not `v != ""` or `v != "0"`.

## Skill: Server-Side Input Validation

Request bodies must be validated before hitting Firestore. Required fields (e.g., `title`) must be non-empty. Enum fields (e.g., `priority`) must be one of the allowed values: `low`, `medium`, `high`. Unvalidated enum values are stored and later served to the frontend, which has no case for them.

## Skill: No Blocking Calls in Request Handlers

`time.Sleep` must never appear in a request-handling goroutine — it holds the connection and blocks the server under load. Delays belong in background workers (Pub/Sub push handlers or goroutines).

## Skill: Batch Commit Error Handling

`batch.Commit(ctx)` returns an error. Ignoring it means a write failure is silently swallowed and the client receives a 200 with stale data. Always check and propagate the error.

## Cross-Repo Awareness

This repo is the contract source for `se-demo-frontend` and `se-demo-risk`. Changes to json tags, endpoint paths, or enum values here must be reflected in the corresponding frontend type files. Flag PRs where a struct field or route is changed without a matching frontend update.
