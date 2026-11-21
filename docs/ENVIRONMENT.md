# Environment variables

All configuration comes from environment variables, parsed and validated with zod in [`src/config.ts`](../src/config.ts). An empty value is treated as unset, so the default applies. A bad value stops the server at startup with a message naming the variable.

Copy [`.env.example`](../.env.example) to `.env` for local development.

## Server

| Variable          | Default                 | Notes                                                                                                              |
| ----------------- | ----------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `PORT`            | `4000`                  | Port the API listens on                                                                                            |
| `HOST`            | `0.0.0.0`               | Interface to bind                                                                                                  |
| `PUBLIC_BASE_URL` | `http://localhost:4000` | Public URL of this service. Used to build the metadata and proof URIs stored on chain. A trailing slash is removed |
| `CORS_ORIGINS`    | `http://localhost:3000` | Comma separated list of allowed frontend origins                                                                   |
| `UPLOAD_DIR`      | `uploads`               | Directory for proof files                                                                                          |
| `METRICS_ENABLED` | `true`                  | Serve Prometheus metrics at `GET /metrics`. Restrict that path at your proxy in production                         |

## Database

| Variable       | Default                                             | Notes                      |
| -------------- | --------------------------------------------------- | -------------------------- |
| `DATABASE_URL` | `postgres://aidline:aidline@localhost:5433/aidline` | Postgres connection string |

For a hosted provider such as Neon, append `?sslmode=require` to the connection string so `pg` uses TLS, and point it at a development-only database. Never point development or tests at the production database. `npm test` uses its own temporary Postgres and ignores this variable.

## Stellar and the contract

| Variable              | Default                               | Notes                                                               |
| --------------------- | ------------------------------------- | ------------------------------------------------------------------- |
| `AIDLINE_NETWORK`     | `testnet`                             | `testnet`, `mainnet` or `futurenet`. Selects the network passphrase |
| `AIDLINE_RPC_URL`     | `https://soroban-testnet.stellar.org` | Soroban RPC endpoint                                                |
| `AIDLINE_CONTRACT_ID` | empty                                 | Contract to index. The indexer is off when this is empty            |
| `AIDLINE_TOKEN_ID`    | empty                                 | Token the contract raises in. Keep it in sync with the frontend     |

## Indexer

| Variable               | Default           | Notes                                                     |
| ---------------------- | ----------------- | --------------------------------------------------------- |
| `INDEXER_ENABLED`      | `true`            | Set to `false` to run the API alone, for example in tests |
| `INDEXER_START_LEDGER` | latest minus 1000 | Where to start on a fresh database                        |
| `INDEXER_POLL_MS`      | `5000`            | Poll interval in milliseconds. Minimum `1000`             |

## Proof storage

| Variable               | Default     | Notes                                                                               |
| ---------------------- | ----------- | ----------------------------------------------------------------------------------- |
| `STORAGE_DRIVER`       | `local`     | `local` writes under `UPLOAD_DIR`; `s3` uses any S3 compatible provider             |
| `S3_BUCKET`            | empty       | Required when the driver is `s3`                                                    |
| `S3_REGION`            | `us-east-1` | Signing region                                                                      |
| `S3_ENDPOINT`          | empty       | Custom endpoint URL, for example `http://minio:9000`. AWS is used when empty        |
| `S3_ACCESS_KEY_ID`     | empty       | Required when the driver is `s3`                                                    |
| `S3_SECRET_ACCESS_KEY` | empty       | Required when the driver is `s3`                                                    |
| `S3_SESSION_TOKEN`     | empty       | Optional temporary credentials                                                      |
| `S3_FORCE_PATH_STYLE`  | `true`      | Path-style URLs (`/bucket/key`), supported by MinIO and most S3 compatible services |

Either driver serves files from `${PUBLIC_BASE_URL}/uploads/<key>`, so proof URIs written on chain remain valid after a migration. No npm dependency is added: requests are signed with AWS Signature V4 using `node:crypto`.

## GDACS alerts

| Variable         | Default                             | Notes                                             |
| ---------------- | ----------------------------------- | ------------------------------------------------- |
| `GDACS_ENABLED`  | `true`                              | Set to `false` to skip the external feed entirely |
| `GDACS_FEED_URL` | `https://www.gdacs.org/xml/rss.xml` | RSS feed to poll                                  |
| `GDACS_POLL_MS`  | `300000`                            | Poll interval in milliseconds (minimum `60000`)   |

## Webhooks

| Variable                     | Default | Notes                                                             |
| ---------------------------- | ------- | ----------------------------------------------------------------- |
| `WEBHOOK_POLL_MS`            | `5000`  | How often due deliveries are sent                                 |
| `WEBHOOK_MAX_ATTEMPTS`       | `5`     | Attempts before a delivery is marked failed                       |
| `WEBHOOK_ALLOW_PRIVATE_URLS` | `false` | Allow local/private receiver URLs. Only for development and tests |

## Why the `AIDLINE_` prefix

The Stellar CLI reads `STELLAR_*` variables from `.env`. Sharing those names would make CLI commands run from this folder fail, so Aidline settings use their own prefix.

## Secrets

The backend holds no signing keys and never moves funds. `DATABASE_URL` is the only secret. Do not commit `.env`. The demo seeding script reads `SEED_ADMIN_SECRET` from the shell for a single run and never stores it.

### Indexer starting ledger and recovery

On a fresh database with no stored cursor, the indexer starts at
`INDEXER_START_LEDGER` if configured, otherwise 1,000 ledgers before the RPC
network tip. If that ledger is older than the RPC's retained history, the RPC
oldest available ledger is used instead. The indexer logs the selected ledger
and whether it was configured, defaulted, or clamped to the RPC oldest ledger.
Events before the chosen starting ledger are **not indexed**. Set
`INDEXER_START_LEDGER` explicitly when historic coverage matters and use an
RPC provider with sufficient history retention.

Indexer polling backs off exponentially after consecutive failures (up to two
minutes), returning to normal polling immediately after a successful sync.
The `/health` response includes `lastSyncedAt`, `secondsSinceSync`, and
`stale` (true after 60 seconds without a successful sync); Prometheus exposes
`aidline_indexer_seconds_since_sync` (-1 before the first successful sync).
