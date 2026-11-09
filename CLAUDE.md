# Auth Service

> **Git policy — never auto-commit or auto-push.** Leave your work in the working tree.
> Run `git commit`, `git push`, `gh pr create`, or `scripts/push_all.sh` **only when the user
> explicitly asks in that turn**. Approval for an earlier change does not carry forward, and
> finishing a task is not permission to commit it. Do not use feature branches; stay on the
> current branch.

`@sudobility/auth_service` — framework-agnostic Firebase authentication helpers for backend
services. Published to **GitHub Packages** (public, `@sudobility` scope). This is a pure
helper library; it ships no middleware — consuming apps build their own from these functions.

## Tech Stack

- **Language**: TypeScript (strict, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes: false`)
- **Runtime / package manager**: Bun (do NOT use npm/yarn/pnpm to install)
- **Build**: `tsc` → ESM only, then `scripts/fix-esm-imports.mjs` rewrites relative specifiers
  to add `.js` extensions (TS leaves extensionless imports Node ESM rejects).
- **Test**: Vitest
- **Auth**: Firebase Admin SDK (`firebase-admin`, peer dependency)

## Commands (verified)

```bash
bun run build        # Build to dist/ (tsc -p tsconfig.esm.json + fix-esm-imports)
bun run verify       # typecheck -> lint -> build; does NOT run tests
bun run test:unit    # Run tests (vitest run); bun run test:watch to watch
bun run typecheck    # tsc --noEmit
bun run lint         # eslint src/
bun run clean        # rm -rf dist
```

- **Use `bun run test:unit`, never `bun test`.** The suite relies on Vitest APIs
  (`vi.mock`, `vi.mocked`, `vi.resetModules`) that Bun's built-in test runner lacks —
  `bun test` reports failures that are not real.
- `tsconfig.json` sets `noEmit: true`; the build uses `tsconfig.esm.json` which flips it off.

## Project Structure

```
src/
├── index.ts                    # Public barrel export
├── init.ts                     # initializeAuth / isAuthInitialized / resetAuthInitialization
├── types/index.ts              # Type defs; re-exports UserInfoResponse from @sudobility/types
└── helpers/
    ├── index.ts                # Helper barrel
    ├── FirebaseHelper.ts       # Admin SDK init, getFirebaseAuth, verifyIdToken, isAnonymousUser
    ├── TokenCache.ts           # createCachedVerifier (TTL + LRU in-memory cache)
    ├── AdminHelper.ts          # Site admin whitelist (createAdminChecker from @sudobility/types)
    ├── UserInfoHelper.ts       # getUserInfo (Firebase UserRecord -> UserInfoResponse)
    └── AccountDeletionHelper.ts# deleteUserAccount + Google/Apple OAuth revocation
tests/                          # Vitest unit tests (Firebase mocked, never hits backend)
```

## Exported Functions

| Function | Notes |
|----------|-------|
| `initializeAuth(config)` | Init Firebase Admin + site admin list. Call once at startup. |
| `verifyIdToken(token)` | Uncached verify; round-trips to Firebase every call. |
| `createCachedVerifier(ttlMsOrOptions?)` | Cached verifier → `{ verify, stop, cache }`. |
| `isSiteAdmin(email)` | Case-insensitive; `false` if uninitialized or email nullish. |
| `isAnonymousUser(decodedToken)` | Takes a decoded token, not a raw string. |
| `getUserInfo(userId)` | Returns `null` on `auth/user-not-found`; rethrows other errors. |
| `deleteUserAccount(userId, options?)` | Revokes Google/Apple/Firebase tokens, deletes user. |
| `getFirebaseAuth()` | Raw Auth instance; throws if not initialized. |

`createCachedVerifier` accepts a numeric TTL (ms, legacy) or `{ ttlMs?, maxCacheSize? }`.
Defaults: 5 min TTL, 1000 entries, LRU eviction. `deleteUserAccount` always revokes Firebase
refresh tokens and deletes the record; each OAuth revocation is optional and its result flag is
`null` when not attempted, `false` on failure — deletion proceeds regardless.

## Environment Variables (read by consumers, passed into `initializeAuth`)

| Variable | Description |
|----------|-------------|
| `FIREBASE_PROJECT_ID` | Firebase project ID |
| `FIREBASE_CLIENT_EMAIL` | Service account email |
| `FIREBASE_PRIVATE_KEY` | Service account private key (literal `\n` is normalized to newlines) |
| `SITEADMIN_EMAILS` | Comma-separated admin email whitelist |

## Gotchas

- **`initializeAuth()` must run before any other function** — otherwise `getFirebaseAuth()`
  throws "Firebase Admin not initialized".
- **`firebase-admin` and `@sudobility/types` are peer dependencies** — do not add them to
  `dependencies`.
- **ESM-only output** — consumers must use `"type": "module"` / ESM imports.
- **Token cache starts a `setInterval`** — call the returned `stop()` in tests to avoid
  dangling timers.
- **`initializeFirebaseAdmin` reuses an existing app** — if other Firebase Admin apps already
  exist in the process it adopts the first one instead of creating its own.

## Workspace & Downstream

Part of the **ShapeShyft** multi-project workspace (parent dir; see `../CLAUDE.md` for the full
dependency graph and build order). Direct consumer: `shapeshyft_api` (auth for protected
routes). Other `*_api` backends consume it too.

After changing this package:
1. `bun run verify` (and `bun run test:unit`) here.
2. Publish (CI publishes to GitHub Packages on push to `main`/`develop` via
   `johnqh/workflows` unified-cicd; `publish-target: github`).
3. In a consumer: `bun update @sudobility/auth_service`, then rebuild.

### Local dev (link instead of publishing)

```bash
# here:            bun link
# in consumer:     bun link @sudobility/auth_service
# rebuild here:    bun run build
# when done:       bun unlink @sudobility/auth_service && bun install  (in consumer)
```
