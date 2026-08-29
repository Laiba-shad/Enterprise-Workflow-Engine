# Encryption at Rest and TLS in Transit

This project stores application data in MongoDB and attachment files in MinIO, which is an S3-compatible object store.

## What the task means

- Encryption at rest: data is encrypted while it sits on disk, such as MongoDB volume files or MinIO object files.
- TLS in transit: traffic is encrypted while moving over the network, such as browser to Nginx, backend to MongoDB, or backend to S3/MinIO.

## What is implemented in this repo

- Browser-facing TLS is already handled by Nginx on `https://localhost`.
- The backend can now enforce HTTPS for S3/MinIO with `S3_REQUIRE_TLS=true`.
- The backend can now request S3 server-side encryption for every uploaded attachment.

S3 encryption options:

```env
S3_ENCRYPTION_ENABLED=true
S3_ENCRYPTION_MODE=S3
```

Use `S3` when the S3 provider has default server-side encryption configured. In AWS this maps to SSE-S3. In MinIO this requires MinIO/KMS support to be configured on the server.

```env
S3_ENCRYPTION_ENABLED=true
S3_ENCRYPTION_MODE=KMS
S3_ENCRYPTION_KMS_KEY_ID=alias/todo-attachments
```

Use `KMS` when AWS KMS or MinIO KES/KMS is configured.

```env
S3_ENCRYPTION_ENABLED=true
S3_ENCRYPTION_MODE=CUSTOMER
S3_ENCRYPTION_CUSTOMER_KEY_BASE64=<base64 encoded 32-byte AES key>
```

Use `CUSTOMER` only when you want the application to provide the AES-256 key for S3 SSE-C. Keep this key in a secrets manager, not in git. If this key is lost, existing encrypted objects cannot be read.

To generate a local test key in PowerShell:

```powershell
[Convert]::ToBase64String((1..32 | ForEach-Object { Get-Random -Maximum 256 }))
```

## MongoDB encryption at rest

MongoDB encryption at rest is not an application-code setting in this project. Choose one of these deployment options:

- MongoDB Atlas: enable encryption at rest in the cluster settings, optionally with customer-managed keys.
- MongoDB Enterprise: enable the encrypted storage engine with a KMS or local keyfile.
- Self-hosted local/dev MongoDB Community: use encrypted host disks or encrypted Docker volumes.

The current Docker Compose file uses the community `mongo:7.0` image, so disk encryption must come from the host or volume layer unless you change the database deployment.

## MongoDB TLS in transit

The backend uses `SPRING_DATA_MONGODB_URI`, so MongoDB TLS is enabled through the connection string once the MongoDB server is configured with certificates:

```env
SPRING_DATA_MONGODB_URI=mongodb://user:password@mongodb:27017/tododb?authSource=admin&tls=true
```

For production, also configure Java to trust the MongoDB certificate authority if it is private or self-signed.

## Recommended production checklist

- Replace local self-signed Nginx certificates with trusted CA certificates.
- Use an HTTPS S3 endpoint and set `S3_REQUIRE_TLS=true`.
- Enable `S3_ENCRYPTION_ENABLED=true` with `S3`, `KMS`, or `CUSTOMER`, depending on your object store.
- Enable MongoDB encryption at rest in Atlas, MongoDB Enterprise, or the encrypted storage layer.
- Enable MongoDB TLS and update `SPRING_DATA_MONGODB_URI` with `tls=true`.
- Store database passwords, S3 credentials, and encryption keys in a secrets manager.

## Local backups on Windows

The local Docker stack can be backed up from PowerShell:

```powershell
.\scripts\backup-local.ps1
```

The script writes timestamped backups under `infrastructure/backups/`:

- `mongodb/tododb.archive.gz`: a compressed `mongodump` archive.
- `buckets/todo-attachments/`: a filesystem sync of the MinIO bucket.

This backup folder is ignored by git. For production, copy these backups to a separate secure location, such as encrypted external storage or a private backup bucket.
