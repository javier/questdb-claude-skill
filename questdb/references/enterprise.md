# QuestDB Enterprise: Authentication & ACL

Skip this entirely for QuestDB Open Source (no auth needed).

Full ACL reference: fetch `query/sql/acl/create-user`, `query/sql/acl/create-service-account`, `query/sql/acl/alter-user`

## 1. Create User or Service Account

```sql
-- Interactive user (for humans / Grafana)
CREATE USER analyst;

-- Service account (for applications / pipelines)
CREATE SERVICE ACCOUNT ingestor;
```

Use a service account for automated pipelines. Use a user for interactive access.

## 2. Grant Permissions

```sql
-- Table & view creation (creator auto-gets all permissions on what they create)
GRANT CREATE TABLE TO analyst;
GRANT CREATE MATERIALIZED VIEW TO analyst;

-- Insert on pre-existing tables (not needed if the user created the table)
GRANT INSERT TO ingestor;

-- Protocol access — grant only what's needed
GRANT PGWIRE TO analyst;     -- PostgreSQL wire protocol (port 8812)
GRANT HTTP TO analyst;        -- REST API + HTTP ingestion (port 9000)
GRANT ILP TO ingestor;        -- ILP ingestion (port 9009)

-- Admin
GRANT LIST USERS TO admin_user;  -- list users and service accounts
```

## 3. Create Auth Tokens

**REST API token** (for HTTP ingestion and REST queries):
```sql
ALTER USER api_user CREATE TOKEN TYPE REST WITH TTL '3600d' REFRESH;
-- or for service accounts:
ALTER SERVICE ACCOUNT ingestor CREATE TOKEN TYPE REST WITH TTL '3600d' REFRESH;
```

Returns:
| name | token | expires_at | refresh |
|---|---|---|---|
| api_user | qt1Kd8Rqmy5jb_YdIJeJ... | 2035-12-20T09:44:30Z | true |

**JWK token** (for ILP/TCP ingestion):
```sql
ALTER USER sensor_writer CREATE TOKEN TYPE JWK;
-- or for service accounts:
ALTER SERVICE ACCOUNT ingestor CREATE TOKEN TYPE JWK;
```

Returns three values needed for ILP auth:
| name | public_key_x | public_key_y | private_key |
|---|---|---|---|
| sensor_writer | TQPRohJ4IKMUo... | D4JGDU5FT2jSo... | Herob8IUAJ6-d... |

## 4. Connection Strings with Auth

**HTTP ingestion (recommended — returns errors immediately):**
```python
# Open Source (no auth)
conf = f"http::addr={host}:9000;auto_flush_interval={auto_flush_interval};"

# Enterprise (REST token + TLS)
conf = f"https::addr={host}:9000;token={token};tls_verify=unsafe_off;auto_flush_interval={auto_flush_interval};"
```

**TCP/ILP ingestion (faster, but errors are silent — risk of data loss on disconnect):**
```python
# Open Source (no auth)
conf = f"tcp::addr={host}:9009;protocol_version=2;auto_flush_interval={auto_flush_interval};"

# Enterprise (JWK token + TLS)
conf = f"tcps::addr={host}:9009;username={ilp_user};token={private_key};token_x={public_key_x};token_y={public_key_y};tls_verify=unsafe_off;protocol_version=2;auto_flush_interval={auto_flush_interval};"
```

**Protocol choice:**
- **HTTP** (recommended): Returns errors immediately. Slightly slower. Use `http::` (open source) or `https::` (enterprise).
- **TCP/ILP**: Fastest throughput. Errors are NOT communicated back — data loss can happen silently on disconnect. Use `tcp::` (open source) or `tcps::` (enterprise).

**TLS / self-signed certificates:**
When the server uses `https` or `tcps`, the client verifies the TLS certificate by default. For self-signed certificates (common in dev/staging), add `tls_verify=unsafe_off;` to the connection string. **Always ask the user** whether the server uses a self-signed certificate before adding this flag — never add it silently in production configs.

## 5. PG Wire with Auth (Grafana, psycopg)

```python
import psycopg as pg

# Open Source
conn_str = f"user=admin password=quest host=localhost port=8812 dbname=qdb"

# Enterprise
conn_str = f"user={user} password={password} host={host} port=8812 dbname=qdb"

conn = pg.connect(conn_str)
```

No `sslmode` needed — QuestDB handles PG wire auth via its own user/password system.
