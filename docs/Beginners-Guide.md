# Beginner to Advanced Project Architecture Guide

This document explains the core technologies used in the **Todo-New** project. It moves from "What is this?" to "How does it work in the configuration?"

---

## 1. Docker & Images: The "Shipping Container" of Code

### Beginner: What is it?
Imagine you are moving to a new house. Instead of carrying every plate and chair individually, you put them in a **Shipping Container**.
- **Image**: This is the "Blueprint" or the "Packed Box." It contains the code, the operating system (like Linux), and all the libraries needed to run the app. It doesn't "run"; it just sits there.
- **Container**: This is the "Running Instance." When you tell Docker to "run" an image, it creates a container. It's like taking the box out of the shipping container and actually using the items.
- **Docker Compose**: This is the "Foreman." Instead of starting 7 different containers manually, you give the foreman a list (`docker-compose.yaml`), and he starts them all in the right order.

### Advanced: How it works in your project
In your `docker-compose.yaml`:
- **`build: context: ../todo-frontend`**: Tells Docker to look in the frontend folder, find the `Dockerfile`, and build a custom **Image** for your app.
- **`networks: todo-network`**: Docker creates a private virtual network. Inside this network, services can talk to each other using their names (e.g., the backend can reach Redis by just calling `http://redis:6379`).
- **`volumes`**: Containers are "amnesiac"—if they stop, they forget everything. Volumes map a folder on your computer to a folder in the container so data (like your database or Keycloak settings) stays safe even if the container restarts.

---

## 2. Nginx: The "Traffic Police" & SSL Termination

### Beginner: What is it?
Nginx is a **Reverse Proxy**. 
- Without Nginx: You would have to remember `localhost:4200` for frontend, `localhost:8081` for backend, and `localhost:8080` for Keycloak.
- With Nginx: You only go to `https://localhost`. Nginx looks at the URL and decides where to send you. 
  - `/api/*` -> Goes to Backend.
  - `/realms/*` -> Goes to Keycloak.
  - `/` -> Goes to Frontend.

### Intermediate: SSL and SSL Termination
- **SSL (Secure Sockets Layer)**: Now officially called **TLS (Transport Layer Security)**. It's the "S" in `HTTPS`. It encrypts the data between your browser and the server so hackers can't see your password.
- **SSL Termination**: Your Nginx is configured for "Termination." This means the encrypted "Secret" connection ends at Nginx. 
  1. Your Browser <--- **Encrypted (HTTPS)** ---> Nginx
  2. Nginx <--- **Unencrypted (HTTP)** ---> Backend/Keycloak
  - This is faster because the Backend doesn't have to waste power "decoding" the encryption; Nginx handles it all at the door.

### Advanced: Configuration Breakdown (`nginx.conf`)
- **`listen 443 ssl;`**: Tells Nginx to listen on port 443 (the standard for HTTPS).
- **`proxy_set_header X-Forwarded-Proto https;`**: Since Nginx "terminates" the SSL, the Backend thinks the request is just regular HTTP. This header tells the Backend: "Hey, the user actually used HTTPS, don't worry."
- **`resolver 127.0.0.11;`**: This is the internal Docker DNS. It allows Nginx to find the IP addresses of `backend` or `frontend` containers dynamically.

---

## 3. Redis: The "Short-Term Memory"

### Beginner: What is it?
Redis is an **In-Memory Data Store**.
- **Database (MongoDB)**: Like a filing cabinet. It's huge, safe, but slow to open and search.
- **Redis**: Like a sticky note on your desk. It's tiny and very fast.

### Intermediate: Fallback Strategy
When the Backend needs data:
1. It checks Redis first (**Cache Hit**). If it's there, it returns it instantly.
2. If it's NOT in Redis (**Cache Miss**), it goes to MongoDB (the "Fallback").
3. Once it gets the data from MongoDB, it saves a copy in Redis for next time.

### Advanced: Configuration (`docker-compose.yaml`)
- **`command: redis-server --requirepass ${REDIS_PASSWORD}`**: This ensures Redis isn't open to everyone. Even inside the Docker network, the backend must "log in" to Redis.

---

## 4. Elasticsearch: The "Super-Fast Librarian"

### Beginner: What is it?
Elasticsearch is a search engine. While MongoDB is good at storing "Folders" (User A, User B), Elasticsearch is good at "Searching inside every word of every document" across millions of entries instantly.

### Advanced: Why use it?
If you search for "Buy milk" in a regular database, it might look through every row one by one. Elasticsearch creates an "Inverted Index" (like the index at the back of a book) so it knows exactly which documents contain the word "milk" without looking at everything else.

---

## 5. Keycloak: The "Bouncer"

### Beginner: What is it?
Keycloak handles your Login. Instead of your app storing passwords (which is dangerous), your app asks Keycloak: "Is this person allowed in?"

### Intermediate: The "Random Redirect" Problem
You mentioned sometimes you are redirected to sign-in even without entering a password. This usually happens because of **Tokens**:
1. **Access Token**: A "VIP Pass" that lasts a short time (e.g., 5 minutes).
2. **Refresh Token**: A "Long-term Pass" (e.g., 30 minutes).
- If your Access Token expires, the Frontend uses the Refresh Token to get a new one silently (you don't see anything).
- If your **Refresh Token** expires, or if your **Session** on the Keycloak server is killed, you are "Redirected to the sign-in page." 
- **Why no password?** If you still have a valid "Session Cookie" in your browser, Keycloak recognizes you and logs you back in immediately without asking for a password, but it still has to "redirect" you to its own page to issue a new token.

### Advanced: Configuration (`todo-realm.json`)
This file contains the "Rules" for your project:
- Who can register?
- How long do tokens last?
- What are the "Roles" (Admin, User)?
When Docker starts, it **imports** this file so you don't have to set up Keycloak manually every time.

---

## 6. Port Exposure: What is "Expose" vs "Ports"?

In your `docker-compose.yaml`:
- **`ports: - "80:80"` (Nginx)**: This "Exposes" the port to your REAL computer. You can type `localhost:80` in Chrome and it works.
- **`expose: - "8081"` (Backend)**: This makes the port visible ONLY to other containers in the Docker network. Your browser **cannot** reach `localhost:8081` directly. This is safer because it hides the backend behind the Nginx "Traffic Police."

---

## Summary of Configuration Purpose
1. **`.env`**: Stores your secrets (passwords). This keeps them out of the main code.
2. **`docker-compose.yaml`**: The master plan for starting all services.
3. **`nginx.conf`**: The rules for directing traffic and handling SSL.
4. **`todo-realm.json`**: The predefined security settings for Keycloak.
