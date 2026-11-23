# Walflow: self-hosted quick start

Walflow watches selected tables in your PostgreSQL database for committed `INSERT`, `UPDATE`, and `DELETE` changes. It turns those changes into JSON events, buffers them in Redis, and sends them to the webhook sinks that you configure in the web console.

Walflow does not host PostgreSQL and does not copy an initial snapshot of your data. You connect an existing PostgreSQL database and choose the tables to capture.

After startup, this same guide is available in the web console at [http://localhost:3000/docs](http://localhost:3000/docs), rendered with Fumadocs. Keep this file with the Docker Compose package so installation instructions remain available before the app is running.

## What this package starts

`docker compose up -d` starts two containers:

- `walflow-server`: the web console, API, CDC connector, and sequential webhook delivery worker.
- `redis`: the internal event buffer. It is not exposed to the host computer.

The web console is available at [http://localhost:3000](http://localhost:3000). Port `3000` is the supported default in this package.

Your saved configuration is stored in the Docker volume `walflow-data`. Redis events are stored in `walflow-redis-data`.

## Requirements

- Docker Desktop with Docker Compose on Windows or macOS, or Docker Engine plus the Docker Compose plugin on Linux.
- Network access from the Walflow container to your PostgreSQL database and webhook endpoint.
- A PostgreSQL role that can use logical replication for the source database.
- A Walflow container image reference supplied by the publisher.

You do not need to install Go, Node.js, Redis, or PostgreSQL locally.

## Download, extract, and configure

1. Obtain the Walflow Docker Compose ZIP and its SHA-256 checksum from the Walflow publisher. The ZIP does not include the server image; the publisher must provide an image reference separately.
2. Extract the ZIP using File Explorer's **Extract All** action.
3. Open PowerShell in the extracted folder. It contains `docker-compose.yml`, `.env.example`, and this README.
4. Create your private configuration file:

   ```powershell
   Copy-Item .env.example .env
   ```

5. Edit `.env` in a text editor. Set these four values:

   - `WALFLOW_IMAGE`: the exact published image reference supplied by the publisher.
   - `APP_ENCRYPTION_KEY`: a persistent Base64 key generated in the next step.
   - `ADMIN_EMAIL`: your administrator email address.
   - `ADMIN_PASSWORD`: a unique password with at least 12 characters.

Generate the encryption key once:

```powershell
$bytes = New-Object byte[] 32
[System.Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($bytes)
[Convert]::ToBase64String($bytes)
```

Copy the output into `APP_ENCRYPTION_KEY`. Keep it in a password manager or another secure backup location. Walflow uses it to encrypt saved database connection strings and webhook signing secrets. If you lose or change it, saved connections and secrets cannot be read after a restore.

Do not commit `.env`, send it in support messages, or share it with other users.

## Start and open Walflow

From the extracted folder:

```powershell
docker compose up -d
docker compose ps
```

Wait until `walflow-server` reports `healthy`, then open [http://localhost:3000](http://localhost:3000).

The first startup stores the administrator email and a password hash in `walflow-data`; it never logs the password. The current release creates this administrator record for the console's administrator model. It does **not yet enforce a login screen**, so run it only on a trusted local machine or protected private network.

After the initial startup succeeds, remove `ADMIN_EMAIL` and `ADMIN_PASSWORD` from `.env`. The stored administrator remains. To change the stored password later:

```powershell
$securePassword = Read-Host "New administrator password" -AsSecureString
$pointer = [Runtime.InteropServices.Marshal]::SecureStringToBSTR($securePassword)
try {
  [Runtime.InteropServices.Marshal]::PtrToStringBSTR($pointer) |
    docker compose run --rm -T walflow-server admin-password --stdin
} finally {
  [Runtime.InteropServices.Marshal]::ZeroFreeBSTR($pointer)
  Remove-Variable securePassword
}
```

## Add a PostgreSQL database

1. Open **Databases** in the console and choose **Add Database**.
2. Enter a display name and your PostgreSQL connection string.
3. Choose **Test Connection**. Walflow tests ordinary SQL connectivity and logical replication separately.
4. Save only after both checks pass.

For remote databases, use TLS in the connection string (`sslmode=require`, `sslmode=verify-ca`, or `sslmode=verify-full`). Logical replication needs a direct database endpoint. Do not use a transaction-pooling or `-pooler` endpoint for PostgreSQL logical replication.

### Logical replication prerequisites and provider caveats

- PostgreSQL must have `wal_level=logical`.
- The connecting role needs the PostgreSQL `REPLICATION` attribute and enough access to the selected tables.
- The provider must permit logical replication slots and publications. Some hosted PostgreSQL plans do not.
- Neon must have Logical Replication enabled in its project settings and normally requires a direct, unpooled connection endpoint.
- Supabase requires its direct connection string on port `5432` (`db.[ref].supabase.co`), not the connection pooler. Because Supabase direct endpoints resolve exclusively to IPv6 addresses, Docker Desktop must have IPv6 enabled in **Settings → Docker Engine** (`"ipv6": true, "fixed-cidr-v6": "2001:db8:1::/64", "ip6tables": true`); otherwise the container receives `Network unreachable`.
- Walflow does not fall back to polling. If a prerequisite fails, correct it at the source before creating a pipeline.

## Browse tables safely

Open a saved database and choose **Browse**. Walflow shows non-system schemas, tables, column information, primary keys, replica identity, and a small read-only row preview. The browser never accepts arbitrary SQL and does not modify source data.

Tables without a primary key or `REPLICA IDENTITY FULL` are flagged because complete `UPDATE` and `DELETE` capture may not be available.

## Create and run a pipeline

1. In the database browser, select one or more eligible tables.
2. Choose **Create Pipeline** and provide a name.
3. Review the selected tables, then create the pipeline.
4. Walflow validates the source again, creates its publication and logical replication slot, and starts capture.

The first release supports **one running pipeline** per Walflow installation. You may save additional pipelines, but stop the active one before starting another. Stopping a pipeline preserves its slot and publication. Delete is only available after stopping it, and removes its Walflow-managed slot and publication.

Important: Walflow publishes a committed transaction to Redis before it acknowledges the corresponding PostgreSQL LSN. This avoids acknowledging an event that was not buffered, but a restart or retry can still cause duplicate events. Webhook receivers must deduplicate using `X-CDC-Event-Id`.

## Create a Webhook sink

1. Open **Sinks** and choose **Create Sink**. This release opens the Webhook sink setup directly.
2. Choose the source pipeline and enter an HTTPS endpoint.
3. Enter a signing secret, or leave it blank for Walflow to generate one.
4. Review the configuration and create the sink.

Walflow displays a generated secret once only. Save it immediately; it is encrypted in `walflow-data` and cannot be displayed later.

For every request Walflow signs the exact JSON request body using HMAC-SHA256 over `timestamp + "." + body` and sends:

```text
X-CDC-Event-Id
X-CDC-Timestamp
X-CDC-Signature: v1=<hex HMAC-SHA256>
```

For safety, Walflow blocks webhook targets that resolve to loopback, private, link-local, or cloud-metadata addresses, and blocks redirects. Use a publicly reachable HTTPS endpoint.

## Test delivery

### Webhook.site

1. Open [Webhook.site](https://webhook.site/) and create a unique request URL.
2. Create an enabled sink using that HTTPS URL.
3. Choose **Send Test Event** on the sink card. The request appears in Webhook.site and the sink should show a successful attempt.
4. Start the linked pipeline and insert, update, or delete a row in one of its selected tables. The CDC event should appear in both Webhook.site and the sink delivery history.

### Included sample receiver

The included Go receiver listens only on `localhost`, which Walflow correctly blocks as a webhook destination. Run it and expose it through a public HTTPS tunnel, then use the tunnel URL for the sink:

```powershell
Set-Location sample\receiver
go run .
```

The receiver verifies the HMAC signature and prints the event ID and operation. Do not weaken the SSRF protection just to send to localhost.

## Sink health and delivery history

Each sink card shows its associated pipeline, enabled state, recent attempts, last success, last failure, and a redacted error summary. A saved sink is not automatically Healthy.

- **Healthy**: the newest observed delivery succeeded.
- **Degraded**: the newest delivery failed, but there are earlier successful deliveries.
- **Failing**: at least three recent deliveries all failed.
- **Paused**: the sink is disabled.
- **Unknown**: no delivery has been observed yet.

The console keeps the newest 100 delivery records per sink and displays the newest 20. Records contain event ID, timestamp, result, optional HTTP status, and a safe error summary. They never contain secrets or event payloads.

Delivery is sequential. On failure, the Redis event stays pending for a later retry; this release has no retry schedule, replay UI, or dead-letter queue.

## Logs and troubleshooting

Useful commands:

```powershell
docker compose ps
docker compose logs -f walflow-server
docker compose logs -f redis
```

| Problem | What to check |
| --- | --- |
| `walflow-server` does not start | Run `docker compose logs walflow-server`. Confirm `WALFLOW_IMAGE` points to an image you can pull and `APP_ENCRYPTION_KEY` is Base64 for exactly 32 random bytes. On first start both administrator values must be set. |
| Database test says SQL works but CDC is unavailable | Enable logical replication, use a direct endpoint, and ensure the database role has `REPLICATION`. |
| Connection fails | Confirm host, port, credentials, firewall rules, and TLS settings. The container—not just your browser—must reach the database. |
| Supabase connection fails ("Could not connect to PostgreSQL") | Supabase direct endpoints use IPv6. In Docker Desktop, go to **Settings → Docker Engine**, add `"ipv6": true, "fixed-cidr-v6": "2001:db8:1::/64", "ip6tables": true`, click **Apply & restart**, and restart containers (`docker compose restart`). |
| Pipeline does not capture a row | Confirm the pipeline is running and the changed table is selected by that pipeline. Only committed changes are captured. |
| Sink stays Unknown | Use **Send Test Event**. A saved endpoint alone is not delivery evidence. |
| Sink is Degraded or Failing | Inspect the redacted error and HTTP status, confirm the endpoint is public and returns a 2xx response, and verify the receiver accepts the HMAC headers. |
| Webhook is not called for a CDC event | Confirm the sink is enabled and linked to the same running pipeline that captures the changed table. Follow `docker compose logs -f walflow-server` and look for `webhook delivered` or `webhook delivery failed`. |

## Update Walflow

Edit `WALFLOW_IMAGE` in `.env` to the newer image reference supplied by the publisher, then run:

```powershell
docker compose pull
docker compose up -d
```

Do not add `-v` when updating. Docker keeps both persistent volumes.

## Back up and restore

Stop Walflow before backup:

```powershell
docker compose down
$backupPath = (Get-Location).Path
docker run --rm -v walflow-data:/data -v "${backupPath}:/backup" alpine tar czf /backup/walflow-data-backup.tar.gz -C /data .
docker run --rm -v walflow-redis-data:/data -v "${backupPath}:/backup" alpine tar czf /backup/walflow-redis-data-backup.tar.gz -C /data .
```

Keep the metadata backup and `APP_ENCRYPTION_KEY` together. To restore, use the same encryption key, stop the app, then run:

```powershell
docker compose down
$backupPath = (Get-Location).Path
docker run --rm -v walflow-data:/data -v "${backupPath}:/backup" alpine sh -c "rm -rf /data/* && tar xzf /backup/walflow-data-backup.tar.gz -C /data"
docker run --rm -v walflow-redis-data:/data -v "${backupPath}:/backup" alpine sh -c "rm -rf /data/* && tar xzf /backup/walflow-redis-data-backup.tar.gz -C /data"
docker compose up -d
```

## Stop or uninstall

Stop containers while keeping all customer data:

```powershell
docker compose down
```

Only when you intentionally want to permanently delete all saved databases, pipeline configuration, sinks, delivery history, and buffered events:

```powershell
docker compose down -v
```

This deletion cannot be undone unless you made a backup.

