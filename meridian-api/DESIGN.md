# meridian-api: Amount Precision, Auth Flows & Pagination Design

**Issues:** #1642 (Amount Precision), #1646 (Pagination & Cursor Semantics)
**Date:** 2026-08-28

---

## Overview

This module implements production-grade amount precision, overflow-safe authentication/account-recovery flows, and deterministic cursor-based pagination for the Meridian API layer.

---

## Architecture

```
meridian-api/src/
├── lib.rs                  # Crate root — re-exports amount, auth, pagination
├── amount.rs               # Type-safe Stellar amounts with checked arithmetic
├── pagination.rs           # Opaque cursors, page limits, scope safety
└── auth/
    ├── mod.rs              # Auth module root — re-exports public types
    ├── errors.rs           # AuthError enum with structured context
    ├── tokens.rs           # Token lifecycle (issue, verify, refresh, revoke)
    ├── flows.rs            # Sign-in, logout, verification + paginated queries
    ├── recovery.rs         # Account recovery + paginated queries
    └── validation.rs       # Auth-context validation helpers
```

---

## Design Invariants

### Amount Precision (`amount.rs`)

1. **Strictly positive**: An `Amount` is always `> 0`. Zero and negative values are rejected at construction time via `Amount::new()`.
2. **Stellar range**: The raw stroop value is `<= i64::MAX`. Stored as `u64` to prevent accidental negative arithmetic.
3. **Exact decimal conversion**: `from_xlm_decimal()` rejects strings with more than 7 fractional digits.
4. **Checked arithmetic**: All operations use checked primitives. Overflow/underflow returns `AmountError` *before* any state change.
5. **128-bit intermediates**: Fee calculations use `u128` intermediates to prevent intermediate overflow.
6. **Balance accumulation**: `accumulate_balance()` and `deduct_balance()` are the canonical ways to update balances.

### Auth Flows (`auth/`)

7. **Stale token rejection**: Expired, revoked, or unknown tokens are rejected without state changes.
8. **Token rotation**: Refresh tokens are rotated on every use. Replay of a rotated token revokes the entire session.
9. **Multi-device safety**: Concurrent logins create independent sessions. Logout revokes all tokens.
10. **Recovery idempotency**: Repeated recovery requests produce the same outcome.
11. **Amount validation before state change**: Balance snapshots are validated through `Amount` before tokens are issued.
12. **No partial state on failure**: Any failed operation leaves zero side-effects.

### Pagination & Cursor Semantics (`pagination.rs`)

13. **Deterministic ordering**: All paginated queries sort by `(created_at DESC, id DESC)` — the same cursor always resolves to the same position regardless of concurrent inserts.
14. **Opaque cursors**: Cursors are encoded byte sequences with a Fletcher-16 integrity checksum. They cannot be fabricated or tampered with without detection.
15. **Scope safety**: Each cursor is bound to a specific subject (user id). A cursor for user A cannot be used to paginate user B's data.
16. **Page limits**: Every paginated query enforces `MIN_PAGE_SIZE` (1) and `MAX_PAGE_SIZE` (100). Callers cannot request unlimited pages.
17. **End-of-stream**: When no more results exist, `next_cursor` is `None` and `has_more` is `false`.
18. **Empty results**: An empty page returns `items: []`, `next_cursor: None`, `has_more: false`, `total_count: 0`.
19. **Invalid cursors**: Malformed, corrupted, or scope-mismatched cursors return `PaginationError`, never partial data.
20. **Idempotent re-queries**: Re-fetching the same page (same cursor) returns the same results.

---

## Cursor Encoding

Cursors encode: `[subject_len:u16][subject_bytes][timestamp:u64 BE][id:u64 BE][checksum:u16]`

- **Fletcher-16 checksum** detects accidental corruption and casual tampering.
- **Base-64 encoding** makes cursors safe for HTTP headers, query parameters, and storage.
- A production implementation should replace Fletcher-16 with HMAC-SHA256 for cryptographic integrity.

---

## Failure Behavior

| Condition | Amount Module | Auth Module | Pagination Module |
|---|---|---|---|
| Zero amount | `Err(AmountError::Zero)` | `Err(AuthError::AmountValidationFailed)` | — |
| Overflow | `Err(AmountError::Overflow)` | `Err(AuthError::AmountValidationFailed)` | — |
| Token expired | — | `Err(AuthError::TokenExpired)` | — |
| Token revoked | — | `Err(AuthError::TokenRevoked)` | — |
| Token replay | — | `Err(AuthError::TokenAlreadyUsed)` | — |
| Account locked | — | `Err(AuthError::AccountLocked { .. })` | — |
| Invalid cursor | — | — | `Err(PaginationError::CursorInvalid)` |
| Scope mismatch | — | — | `Err(PaginationError::CursorScopeMismatch)` |
| Invalid page size | — | — | Clamped to `[1, 100]` |

---

## Paginated Collections

Three collections support cursor-based pagination:

| Collection | Query Method | Sort Key | Subject Filter |
|---|---|---|---|
| Sessions | `AuthService::list_sessions()` | `(created_at, id)` | `session.subject == subject` |
| Tokens | `AuthService::list_tokens()` | `(access.issued_at, access.id)` | Active, non-superseded sessions |
| Recovery Requests | `RecoveryService::list_requests()` | `(created_at, id)` | `request.subject == subject` |

Each also supports filtered variants (e.g., `list_requests_by_status()`).

---

## Compatibility Impact

- **No breaking changes.** All modules are additive.
- The `Amount` type uses Stellar's native stroop representation.
- Pagination is opt-in — existing code continues to work without changes.
- `RecoveryStatus` is now publicly exported from the auth module.

---

## Migration / Rollout

- Adopt `Amount` incrementally to replace raw `u64`/`i64` amounts.
- Replace ad-hoc token handling with `TokenStore`.
- Use `list_sessions()`, `list_tokens()`, `list_requests()` instead of direct collection access for UI pagination.
- Cursor format is self-describing — no migration needed when upgrading.

---

## Security Assumptions

1. **System clock is monotonic**: Token expiry and cursor timestamps rely on `SystemTime`.
2. **Token store integrity**: The in-memory `TokenStore` is a reference implementation.
3. **No cryptographic verification**: Token signatures are out of scope.
4. **Subject uniqueness**: Token subjects are assumed unique and externally validated.
5. **Cursor integrity**: Fletcher-16 is sufficient for reference implementations. Production should use HMAC-SHA256.
6. **Pagination does not leak cross-subject data**: Scope validation enforces subject binding on every cursor use.

---

## Test Coverage

**163 tests** covering:

- **Amount module (50 tests):** Construction, decimal parsing, i64 conversion, display, arithmetic, fees, balance operations, oracle tests.
- **Auth tokens (10 tests):** Issue, verify, expiry, revocation, refresh, replay, monotonic IDs.
- **Auth flows (25 tests):** Login, balance validation, lockout, logout, verification, multi-device, refresh, **plus 10 new pagination tests** for session/token listing.
- **Auth recovery (20 tests):** Request, idempotency, completion, replay, cancellation, **plus 9 new pagination tests** for recovery request listing.
- **Auth validation (29 tests):** Amount, balance, fee, session, subject, password, recovery validators.
- **Pagination module (29 tests):** Cursor encode/decode, base64, Fletcher-16, page size normalization, empty pages, `compute_page` (empty, single-page, multi-page, exact boundary, scope isolation, stale cursor, large results, deterministic ordering, cursor between items, page-size-one).

---

## Validation Commands

```bash
cargo fmt -- --check        # formatting
cargo clippy -- -D warnings # linting
cargo test                  # full test suite (163 tests)
```

All three pass cleanly as of 2026-08-28.

---

## Files Changed

| File | Description |
|---|---|
| `meridian-api/Cargo.toml` | Crate configuration |
| `meridian-api/src/lib.rs` | Crate root (exports pagination) |
| `meridian-api/src/amount.rs` | Amount type, arithmetic, 50 tests |
| `meridian-api/src/pagination.rs` | Cursor encoding, page types, **new** |
| `meridian-api/src/auth/mod.rs` | Auth module root |
| `meridian-api/src/auth/errors.rs` | AuthError enum |
| `meridian-api/src/auth/tokens.rs` | Token lifecycle, 10 tests |
| `meridian-api/src/auth/flows.rs` | Auth flows + paginated queries, 25 tests |
| `meridian-api/src/auth/recovery.rs` | Recovery flow + paginated queries, 20 tests |
| `meridian-api/src/auth/validation.rs` | Validation helpers, 29 tests |
| `meridian-api/DESIGN.md` | This file |

**No unrelated refactors, no disabled CI, no generated noise, no secrets.**
