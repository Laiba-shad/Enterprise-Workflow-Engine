# Nginx Before and After Implementation Guide

## Purpose of This Document

This document explains what changed after adding Nginx to the Enterprise Todo Application, why Nginx was added, and how it changes the way users and developers access the application.

It focuses on the architectural difference between the application before Nginx and the application after Nginx SSL termination was implemented.

## What Is Nginx?

Nginx is a web server and reverse proxy. In this project, it is used as the single public entry point for the Docker-based application.

Instead of users opening many different container ports directly, users open one HTTPS application URL. Nginx receives the request and forwards it to the correct internal service.

Nginx is responsible for:

1. Accepting browser traffic on ports `80` and `443`.
2. Redirecting HTTP traffic to HTTPS.
3. Handling SSL/TLS certificates.
4. Forwarding frontend requests to Angular.
5. Forwarding API requests to Spring Boot.
6. Forwarding Keycloak authentication routes.
7. Forwarding MinIO console traffic.
8. Keeping internal service ports hidden from normal browser access.

## What SSL Termination Means

SSL termination means HTTPS is handled at Nginx.

The browser connects securely to Nginx:

```text
Browser -> HTTPS -> Nginx
```

After Nginx receives the secure request, it forwards traffic to internal Docker services over the private Docker network:

```text
Nginx -> HTTP -> frontend/backend/keycloak/minio
```

This is normal for reverse-proxy architecture. The public connection is encrypted with HTTPS, while internal service-to-service traffic stays inside the Docker network.

## Before Nginx

Before Nginx, each service was accessed directly through its own port.

Example access pattern:

| Service | Direct URL Before Nginx |
| --- | --- |
| Angular frontend | `http://localhost:4200` |
| Spring Boot backend | `http://localhost:8081` |
| Keycloak | `http://localhost:8080` |
| MinIO console | `http://localhost:9001` |
| Elasticsearch | `http://localhost:9200` |
| MongoDB | `localhost:27017` |
| Redis | `localhost:6379` |

This worked for development, but it had several limitations:

1. Users had to know many different ports.
2. The frontend and authentication flow had to deal with multiple origins.
3. HTTPS was not centralized.
4. The setup was less similar to a production deployment.
5. Internal infrastructure services were more exposed than necessary.
6. Keycloak redirect URLs and frontend API URLs were harder to keep consistent.

## After Nginx

After Nginx, users and developers access the application through HTTPS URLs handled by Nginx.

Public development URLs:

| Purpose | URL |
| --- | --- |
| Angular frontend | `https://localhost/` |
| Spring Boot API | `https://localhost/api/v1/...` |
| Keycloak realm endpoints | `https://localhost/realms/todo/...` |
| Keycloak admin console | `https://localhost/admin/` |
| Keycloak static resources | `https://localhost/resources/...` |
| MinIO console | `https://minio.localhost/` |

Internal services still run on their own ports inside Docker, but users do not normally open those ports directly.

Example internal routing:

| Public Request | Nginx Routes To |
| --- | --- |
| `https://localhost/` | `http://frontend:4200` |
| `https://localhost/api/` | `http://backend:8081` |
| `https://localhost/realms/` | `http://keycloak:8080` |
| `https://localhost/resources/` | `http://keycloak:8080` |
| `https://localhost/admin/` | `http://keycloak:8080` |
| `https://minio.localhost/` | `http://minio:9001` |

## What Changed in the Application

The biggest change is that the application now behaves like one HTTPS application instead of a group of separate local services.

Before:

```text
Browser -> frontend port
Browser -> backend port
Browser -> keycloak port
Browser -> minio port
```

After:

```text
Browser -> HTTPS Nginx -> correct internal service
```

This improves the architecture because:

1. The browser only needs stable HTTPS URLs.
2. Authentication redirects use `https://localhost`.
3. The frontend can call backend APIs through `/api/...`.
4. Keycloak is accessed through the same public host.
5. MinIO is accessed through `https://minio.localhost`.
6. MongoDB and Redis are no longer normal browser-facing services.

## What Users Should Open Now

Use these URLs:

```text
https://localhost/
https://localhost/admin/
https://localhost/realms/todo/.well-known/openid-configuration
https://minio.localhost/
```

Do not expect these route prefixes to work as normal pages:

```text
https://localhost/api/
https://localhost/realms/
https://localhost/resources/
```

They are routing prefixes. For example, `/api/` is protected by backend security and may return `401 Unauthorized` when opened directly without a token. That means the backend route is protected, not necessarily broken.

## How to Verify Nginx Is Working

Validate Nginx configuration:

```powershell
docker compose exec nginx nginx -t
```

Expected result:

```text
syntax is ok
test is successful
```

Check HTTP to HTTPS redirect:

```powershell
curl.exe -I http://localhost
```

Expected result:

```text
HTTP/1.1 301 Moved Permanently
Location: https://localhost/
```

Check frontend through HTTPS:

```powershell
curl.exe -k -I https://localhost
```

Expected result:

```text
HTTP/1.1 200 OK
```

Check Keycloak through Nginx:

```powershell
curl.exe -k https://localhost/realms/todo/.well-known/openid-configuration
```

Expected result: JSON containing the Keycloak issuer:

```text
"issuer":"https://localhost/realms/todo"
```

Check backend route through Nginx:

```powershell
curl.exe -k -I https://localhost/api/v1/todos
```

Expected result may be:

```text
HTTP/1.1 401 Unauthorized
```

That is acceptable when security is enabled because the endpoint requires a valid Keycloak token.

## What Did Not Change

Nginx did not replace the backend, frontend, database, cache, search engine, authentication server, or file storage service.

Those services still exist:

| Service | Role |
| --- | --- |
| Angular | User interface |
| Spring Boot | API and business logic |
| MongoDB | Main todo database |
| Redis | Cache |
| Elasticsearch | Search index |
| Keycloak | Authentication |
| MinIO | File storage |

Nginx only controls how public traffic enters the system and where that traffic is routed.

## Final Result

After adding Nginx, the application has a cleaner and more production-like access pattern:

```text
User -> HTTPS -> Nginx -> Internal Docker Services
```

This provides a single secure entry point, simpler URLs, centralized SSL termination, and better separation between public application routes and internal infrastructure services.
