# Security

Auth is **SCRAM-SHA-256 (P15)** when the user has an `s1:` verifier
(CREATE/ALTER USER writes that), **cleartext** for leftover `h2:`
rows, or a **shared token**. PG wire is TLS 1.3 when `BOILA_TLS_CERT`
and `BOILA_TLS_KEY` (PEM) are set (P22-3). HTTP stays plaintext — put
it behind a terminator if it leaves localhost.

## Modes

| Catalog users | `BOILA_TOKEN` | Behaviour |
|---------------|---------------|-----------|
| none | unset | **Trust** — no password, no HTTP header |
| none | set | Token is the password / Bearer / `X-Boila-Token` → superuser |
| some (`s1:`) | unset | PG SASL SCRAM-SHA-256; HTTP Basic still sends the password |
| some (`h2:`) | unset | PG cleartext; HTTP Basic |
| some | set | User+password **or** token (token → superuser) |

An empty catalog is “open”. The first `CREATE USER` turns the server
into a password server.

## Users and roles

```sql
CREATE USER [IF NOT EXISTS] ana PASSWORD 'secret';
CREATE USER ana IDENTIFIED BY 'secret' SUPERUSER;
CREATE ROLE analyst PASSWORD '…';          -- alias of CREATE USER

ALTER USER ana PASSWORD 'new';
DROP USER [IF EXISTS] ana;

SET ROLE ana PASSWORD 'secret';
SET ROLE NONE;                             -- RESET ROLE / SET SESSION AUTHORIZATION
SHOW USERS;   -- also SHOW ROLES
SHOW GRANTS [FOR ana];
```

`SET ROLE` without a password is allowed only for a superuser
switching to another role.

Passwords in the meta store are `h2:` + HMAC-SHA256
(`hmac_sha256_hex("boila$pw$v2", pw)`). Older `h1:` djb2 rows still
verify. Compare is constant-time.

## Privileges

```sql
GRANT SELECT, INSERT ON users TO ana;
GRANT ALL ON * TO ana;                     -- all tables in the current db
GRANT CONNECT ON DATABASE analytics TO ana;
REVOKE UPDATE, DELETE ON users FROM ana;
```

| Privilege | Bit | Covers |
|-----------|-----|--------|
| `CONNECT` | 1 | Open / `USE` a database |
| `SELECT` | 2 | Reads |
| `INSERT` | 4 | Inserts |
| `UPDATE` | 8 | Updates |
| `DELETE` | 16 | Deletes |
| `CREATE` | 32 | CREATE TABLE / INDEX / … |
| `DROP` | 64 | DROP |
| `ALTER` | 128 | ALTER |
| `ALL` | 255 | All of the above |

Targets: `ON [TABLE] name` (current database — the lexer has no `.`
for `db.table`), `ON *`, `ON DATABASE name`.

Meta keys (registry store): `u|<user>`, `a|<user>|<db>|*`,
`a|<user>|<db>|<table>`.

## HTTP

```
Authorization: Bearer <token>
X-Boila-Token: <token>
Authorization: Basic base64(user:password)
```

`/health` and `/ready` are unauthenticated. `/sql` and `/metrics`
require auth once a token or user exists.

Database pinning (`?db=`, `X-Boila-Db`, `boila_db` cookie) is
HMAC-signed when `BOILA_TOKEN` is set. `BOILA_CSRF=1` requires
cookie and header to match. Details: [http.md](http.md).

## PostgreSQL wire

`BOILA_AUTH=scram|cleartext|trust` (default **auto**: SASL if the
user’s hash is `s1:`, else cleartext). SCRAM is AuthenticationSASL
`SCRAM-SHA-256`. A cleartext `p` when SASL is required → `28000`.
Failure: `28P01`. `BOILA_TOKEN` still works as cleartext or as an
ephemeral SCRAM password when no matching user exists.

TLS 1.3 is opt-in: set `BOILA_TLS_CERT` / `BOILA_TLS_KEY` (PEM paths)
and an SSLRequest is answered `'S'`, upgrading the connection via the
`std/net` TLS 1.3 server (`tls_accept`: x25519, AES-128/256-GCM, cert
key RSA-PSS or ECDSA-P256). Unset → answered `'N'` (plaintext), which
is also what real-Postgres-facing clients get unless they set
`PGSSLMODE=require|prefer`. Auth still runs inside the encrypted
channel when TLS is on.

## What this is not

- No cert chain validation on the server (single leaf), no client-cert
  auth, no SCRAM channel binding (`-PLUS`).
- No row-level security, no column grants.
- No audit log.
- No WebSocket (removed from the surface on purpose).
- Without `BOILA_TLS_CERT`/`BOILA_TLS_KEY`, passwords cross the wire
  in the clear.

Treat `BOILA_TOKEN` like a root password. Rotate by restarting with a
new value and `ALTER USER … PASSWORD`.
