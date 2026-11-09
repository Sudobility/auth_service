# @sudobility/auth_service

Framework-agnostic Firebase authentication helpers for backend services. Provides token verification with caching, site admin checking, and user info retrieval.

## Installation

This package is published to **GitHub Packages** under the `@sudobility` scope.
Consumers need a `.npmrc` pointing the scope at the GitHub registry plus a token
with `read:packages`:

```
@sudobility:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GH_PACKAGES_READ_TOKEN}
```

```bash
bun add @sudobility/auth_service
```

### Peer Dependencies

```bash
bun add firebase-admin @sudobility/types
```

## Usage

### Initialization

```typescript
import { initializeAuth } from '@sudobility/auth_service';

initializeAuth({
  firebase: {
    projectId: process.env.FIREBASE_PROJECT_ID!,
    clientEmail: process.env.FIREBASE_CLIENT_EMAIL!,
    privateKey: process.env.FIREBASE_PRIVATE_KEY!,
  },
  siteAdminEmails: process.env.SITEADMIN_EMAILS,
});
```

### Token Verification (Hono example)

```typescript
import { createCachedVerifier, isSiteAdmin } from '@sudobility/auth_service';

const { verify: verifyToken } = createCachedVerifier(300000); // 5 min TTL

const decoded = await verifyToken(token);
const isAdmin = isSiteAdmin(decoded.email);
```

### User Info

```typescript
import { getUserInfo } from '@sudobility/auth_service';

const userInfo = await getUserInfo(userId); // null if the UID does not exist
```

### Account Deletion

Deletes a Firebase user and optionally revokes the Google / Apple OAuth grants
first. Google revocation needs the user's access token; Apple revocation needs a
fresh authorization code plus your Sign in with Apple key config.

```typescript
import { deleteUserAccount } from '@sudobility/auth_service';

const result = await deleteUserAccount(userId, {
  googleAccessToken: 'ya29...',
  appleAuthorizationCode: 'c1234...',
  appleConfig: {
    clientId: 'com.example.app',   // Services ID
    teamId: 'TEAM123',
    keyId: 'KEY123',
    privateKey: process.env.APPLE_PRIVATE_KEY!, // .p8 PEM contents
  },
});
// result: { userDeleted, googleTokenRevoked, appleTokenRevoked }
```

Firebase refresh tokens are always revoked and the user record deleted. Each
OAuth field is optional; a `*Revoked` flag is `null` when that revocation was
not attempted and `false` when it failed. Deletion still proceeds if a
revocation fails.

## API

| Function | Description |
|----------|-------------|
| `initializeAuth(config)` | Initialize Firebase Admin SDK and site admin list |
| `verifyIdToken(token)` | Verify Firebase ID token (no caching) |
| `createCachedVerifier(ttlMsOrOptions?)` | Create cached token verifier; returns `{ verify, stop, cache }` |
| `isSiteAdmin(email)` | Check if email is a site admin (case-insensitive) |
| `isAnonymousUser(decodedToken)` | Check if a decoded token is from an anonymous user |
| `getUserInfo(userId)` | Get user info from Firebase (`null` if not found) |
| `deleteUserAccount(userId, options?)` | Delete a user and revoke Google/Apple/Firebase tokens |
| `getFirebaseAuth()` | Get the Firebase Auth instance |

`createCachedVerifier` accepts either a numeric TTL in ms (backward compatible)
or an options object `{ ttlMs?, maxCacheSize? }`. Defaults: 5 min TTL, 1000
entries with LRU eviction.

## Environment Variables

| Variable | Description |
|----------|-------------|
| `FIREBASE_PROJECT_ID` | Firebase project ID |
| `FIREBASE_CLIENT_EMAIL` | Firebase service account email |
| `FIREBASE_PRIVATE_KEY` | Firebase service account private key |
| `SITEADMIN_EMAILS` | Comma-separated list of site admin emails |

## Development

```bash
bun run build        # Build to dist/ (ESM via tsc + fix-esm-imports)
bun run verify       # typecheck + lint + build (does NOT run tests)
bun run test:unit    # Run tests (vitest); use `bun run test:watch` to watch
bun run typecheck    # TypeScript check
bun run lint         # Run ESLint
```

> Use `bun run test:unit`, not `bun test`: the suite is written for Vitest
> (`vi.mock`, `vi.mocked`, ...), which Bun's built-in test runner does not
> implement.

## License

BUSL-1.1
