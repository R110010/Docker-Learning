# 🚀 Secure Private Docker Registry with NGINX Reverse Proxy (HTTPS)

This project documents my hands-on journey of building a **secure, production-style private Docker registry** using:

* 🐳 Docker Registry (`registry:2`)
* 🌐 NGINX (Reverse Proxy)
* 🔐 HTTPS with self-signed certificates
* 🧠 Debugging real-world proxy + protocol issues

---

# 🧠 Architecture

```
Client (Docker CLI)
        ↓ HTTPS
     NGINX (TLS Termination)
        ↓ HTTP
   Docker Registry (pi-registry)
```

### Key Idea:

* NGINX handles **HTTPS**
* Registry runs **HTTP internally**
* NGINX acts as a **reverse proxy**

---

# ⚙️ Setup Overview

## 1. Run Docker Registry

```bash
docker run -d \
  --name pi-registry \
  --network registry-net \
  -p 50000:5000 \
  -v pi-docker-local-registry-volume:/var/lib/registry \
  registry:2
```
```
-p 50000:5000 must be removes. it is simply used to push a test image(busybox).
you need to add insecure registry in your local system to push.
```
At this path /etc/docker/daemon.json add this entry 

```json
{
  "insecure-registries": ["192.168.1.200:50000"]
}
```
---

## 2. Create Docker Network

```bash
docker network create registry-net
docker network connect registry-net pi-registry
```

---

## 3. Generate SSL Certificate

```bash
openssl req -x509 -newkey rsa:4096 -sha256 -days 365 \
  -nodes \
  -keyout certs/domain.key \
  -out certs/domain.crt \
  -subj "/CN=192.168.1.200"
```

---

## 4. NGINX Configuration

```nginx
events {}

http {
    client_max_body_size 0;

    server {
        listen 443 ssl;
        server_name _;

        ssl_certificate /etc/nginx/certs/domain.crt;
        ssl_certificate_key /etc/nginx/certs/domain.key;

        add_header Docker-Distribution-Api-Version registry/2.0 always;

        location / {
            proxy_pass http://pi-registry:5000/;

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

            # 🔥 Critical fix for HTTPS awareness
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Ssl on;

            proxy_http_version 1.1;
            proxy_request_buffering off;
            proxy_buffering off;
            chunked_transfer_encoding on;
            proxy_set_header Connection "";

            proxy_read_timeout 900;
        }
    }
}
```

---

## 5. Run NGINX

```bash
docker run -d \
  --name nginx-registry \
  -p 443:443 \
  --network registry-net \
  -v $(pwd)/conf/nginx.conf:/etc/nginx/nginx.conf:ro \
  -v $(pwd)/certs:/etc/nginx/certs:ro \
  nginx
```

---

## 6. Trust Self-Signed Certificate (Client Side)

```bash
sudo mkdir -p /etc/docker/certs.d/192.168.1.200
sudo cp domain.crt /etc/docker/certs.d/192.168.1.200/ca.crt
sudo systemctl restart docker
```

---

# 🧪 Testing

### Pull Image

```bash
docker pull 192.168.1.200/busybox
```

### Push Image

```bash
docker tag hello-world 192.168.1.200/hello-world
docker push 192.168.1.200/hello-world
```

---

# 🐛 Major Debugging Challenge (Key Learning)

### ❌ Problem

Docker push was stuck in infinite retry:

```
POST /v2/... → 202
(repeating...)
```

---

### 🔍 Root Cause

Registry returned incorrect upload URL:

```
Location: http://192.168.1.200/...
This was confirmed with this command
curl -vk -X POST https://192.168.1.200/v2/hello-world/blobs/uploads/
```

But client was using:

```
https://192.168.1.200
```

👉 Docker refused to continue upload due to **protocol mismatch**

---

### ✅ Fix

```nginx
proxy_set_header X-Forwarded-Proto $scheme;
#This line was added to nginx.conf
```

---

### 🧠 Why This Matters

Without this header:

* Registry thinks request is HTTP
* Generates HTTP URLs
* Client rejects them
* Upload loop occurs

---

# 💡 Key Concepts Learned

### 1. Reverse Proxy Context Awareness

Backends rely on headers like:

* `X-Forwarded-Proto`
* `X-Forwarded-Host`

---

### 2. Docker Registry is Stateful

Push flow:

```
POST → PATCH → PUT
```

Even small inconsistencies break the flow.

---

### 3. NGINX Defaults Are Not Enough

Required tuning:

* `proxy_request_buffering off`
* `chunked_transfer_encoding on`
* `client_max_body_size 0`

---

### 4. TLS Termination Pitfalls

When terminating TLS at proxy:

* Backend must be informed of original protocol
* Otherwise generates incorrect URLs

---

# 🚀 Real-World Relevance

This exact issue occurs in:

* Kubernetes Ingress Controllers
* AWS ALB / API Gateway setups
* Production microservices behind proxies
* OAuth / redirect-based systems

---

# 🔥 Final Outcome

✅ Secure private Docker registry
✅ HTTPS with TLS termination
✅ Reverse proxy setup
✅ Correct Docker push/pull flow
✅ Debugged real-world production issue

---

# 📌 Future Improvements

* 🔐 Add authentication (Basic Auth / Token)
* ⚖️ Load balancing across multiple registry nodes
* 🌍 Use domain instead of IP
* ☸️ Integrate with Kubernetes (k3s)

---

# 🙌 Conclusion

This project demonstrates not just setup, but **deep debugging of distributed systems behavior**, especially:

* Reverse proxy interactions
* Protocol mismatches
* Stateful API flows

---

> ⚡ “It’s not just about making it work — it’s about understanding *why it breaks*.”

