# Todo Project Security Hardening Summary

This document explains, in simple language, what was done to make the Todo project safer. It is written for a non-technical reader.

## Why This Work Was Needed

The Todo project stores two important kinds of information:

- Todo data, such as task names, due dates, and user-related records.
- Uploaded files, such as PDF attachments.

If this information is not protected, someone could potentially read it from the network while it is moving between systems, or read it from storage if a disk, database, or file store is exposed.

The goal of this work was to protect data in two situations:

- When data is stored.
- When data is moving between systems.

## Key Terms

### Encryption at Rest

Encryption at rest means data is protected while it is saved somewhere, such as a database, disk, or file storage bucket.

Simple example:

If someone got access to the raw storage files, the data should not be readable as normal text or normal files without the right keys.

### TLS in Transit

TLS is the security technology behind HTTPS.

TLS in transit means data is protected while it travels over a network.

Simple example:

When a browser talks to the Todo app, or when the backend talks to the database, the connection should be encrypted so other people cannot read the traffic.

### Backups

Backups are copies of data saved separately so the project can be recovered if data is lost, damaged, or accidentally deleted.

## What We Changed

## 1. MongoDB Database Security

The project was updated so the backend can use a MongoDB Atlas connection string.

MongoDB Atlas is a managed MongoDB service. In this setup:

- The database is hosted outside the local Docker stack.
- Atlas provides encryption at rest for stored database data.
- Atlas connections use TLS, so data moving between the backend and MongoDB is encrypted.

This means the Todo data is now protected both while stored in Atlas and while traveling between the backend and Atlas.

## 2. File Storage Encryption

The Todo project uses MinIO for uploaded attachments, such as PDFs.

We added support for server-side encryption for uploaded files. This means uploaded files can be encrypted by the storage system when they are saved.

For the local Docker setup, MinIO was configured with a server-side encryption key. The backend now requests encryption when it uploads attachments.

The final working local mode is:

- MinIO manages the storage encryption key.
- The backend asks MinIO to encrypt uploaded files.
- New uploaded PDFs show `Encryption: SSE-S3` when checked with the MinIO client.

Important note:

Files uploaded before encryption was enabled are not automatically changed. Old files must be re-uploaded or copied through the encrypted path if they also need encryption.

## 3. HTTPS for the Web App

The project already uses Nginx as the public entry point.

Nginx is configured to serve the app through HTTPS locally:

- `https://localhost` for the Todo app.
- `https://minio.localhost` for the MinIO console.

This protects traffic between the browser and Nginx.

In local development, the certificate is self-signed. Because of that, the browser may not show a clean padlock icon. This does not mean HTTPS is absent; it means the certificate is not trusted like a production certificate.

For production, the self-signed certificate should be replaced with a trusted certificate, such as one from Let's Encrypt.

## 4. Backup Service

We added a Docker-based MongoDB Atlas backup service.

Previously, backups were done from the local machine using a script. Now the backup can run inside a Docker container.

The backup service:

- Uses a MongoDB Docker image that includes `mongodump`.
- Reads the Atlas connection string from the environment.
- Creates a date-named backup folder.
- Saves the database dump into the local `infrastructure/backups` folder.

Example backup folder:

```text
infrastructure/backups/2026-05-19/dump/
```

This makes backups more repeatable because the backup tool runs inside Docker instead of depending on what is installed directly on the developer's machine.

## What Is Protected Now

### Database Data

Status: protected when using MongoDB Atlas.

- Stored database data is encrypted by Atlas.
- Data moving between the backend and Atlas uses TLS.

### Uploaded Files

Status: protected for new uploads after encryption was enabled.

- New uploaded files are encrypted in MinIO using server-side encryption.
- Backend-to-MinIO traffic now uses HTTPS inside Docker.
- Existing older files may still be unencrypted unless they are re-uploaded or migrated.

### Browser Traffic

Status: protected locally with HTTPS, but using a local self-signed certificate.

- Browser traffic uses HTTPS through Nginx.
- A trusted production certificate is still needed for a real production deployment.

### Backups

Status: Docker-based backup service added.

- Backups are saved under `infrastructure/backups`.
- The backup folder is ignored by git so backups are not accidentally committed to source control.

## What Still Matters for Production

Before using this in a real production environment, these items should be handled:

- Use a trusted HTTPS certificate, such as Let's Encrypt.
- Store passwords and encryption keys in a secret manager, not plain files.
- Restrict MongoDB Atlas network access to trusted IP addresses only.
- Use strong MinIO credentials instead of default local credentials.
- Move backup files to a secure external location, not only the local machine.
- Decide whether internal service-to-service traffic also needs TLS in the local Docker network.

## How To Verify

### Verify MongoDB Atlas TLS

The backend logs should show that MongoDB is connecting to Atlas and that SSL is enabled.

Look for:

```text
sslSettings=SslSettings{enabled=true
```

This means the backend-to-database connection is encrypted.

### Verify File Encryption

After uploading a new PDF, list the MinIO files:

```powershell
docker run --rm --network infrastructure_todo-network -e MC_HOST_todo=https://minioadmin:minioadmin@minio:9000 minio/mc:latest --insecure ls --recursive todo/todo-attachments
```

Then check the new file:

```powershell
docker run --rm --network infrastructure_todo-network -e MC_HOST_todo=https://minioadmin:minioadmin@minio:9000 minio/mc:latest --insecure stat todo/todo-attachments/YOUR_FILE_PATH
```

Copy these commands as one line in PowerShell. If the command is split in the middle, PowerShell can pass the line break into the container and the MinIO Client command may fail.

The result should include:

```text
Encryption: SSE-S3
```

That confirms the uploaded file is encrypted at rest.

### Run a Database Backup

From the `infrastructure` folder:

```powershell
docker compose --profile backup run --rm backup
```

The backup should appear under:

```text
infrastructure/backups/YYYY-MM-DD/dump/
```

## Short Summary

We improved the Todo project by protecting database data, uploaded files, web traffic, and backups.

MongoDB Atlas protects the database with encryption and TLS. MinIO now encrypts new uploaded files. Nginx provides HTTPS for browser traffic. A Docker-based backup service now creates repeatable database backups.

The project is much safer than before, but production still needs proper certificate management, secret management, restricted network access, and secure off-machine backup storage.
