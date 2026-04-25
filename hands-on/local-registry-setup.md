# 🐳 Setting Up a Local Docker Registry on Raspberry Pi

> **Learning Journey**: Running a private Docker registry on Raspberry Pi for local network image distribution

## 📋 Table of Contents
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Architecture Overview](#architecture-overview)
- [Step-by-Step Setup](#step-by-step-setup)
- [Key Concepts & Mental Map](#key-concepts--mental-map)
- [Troubleshooting Journey](#troubleshooting-journey)
- [Important Learnings](#important-learnings)
- [References](#references)

---

## 🎯 Overview

This documentation covers setting up a **private Docker registry** on a Raspberry Pi that can be accessed by other devices on the same local network. This is useful for:
- Learning Docker registry internals
- Sharing custom images across local devices
- Avoiding Docker Hub rate limits
- Faster image pulls on local network

**Network Setup**: All devices connected to the same router (local network)

---

## ✅ Prerequisites

- Raspberry Pi with Docker installed
- Another machine (laptop) with Docker installed
- Both devices on the same network/router
- Basic understanding of Docker commands
- Raspberry Pi IP address (in this case: `192.168.1.200`)

---

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                  Local Network                       │
│                 (Same Router)                        │
│                                                      │
│  ┌──────────────────┐      ┌──────────────────┐    │
│  │  Raspberry Pi    │      │     Laptop       │    │
│  │  192.168.1.200   │◄────►│   (Client)       │    │
│  │                  │      │                  │    │
│  │  ┌────────────┐  │      │  ┌────────────┐  │    │
│  │  │  Registry  │  │      │  │   Docker   │  │    │
│  │  │  Container │  │      │  │   Daemon   │  │    │
│  │  │            │  │      │  └────────────┘  │    │
│  │  │ Port 50000 │  │      │                  │    │
│  │  └────────────┘  │      │  Push/Pull       │    │
│  │        ▲         │      │  Images          │    │
│  │        │         │      │                  │    │
│  │  Volume Storage  │      └──────────────────┘    │
│  │  (Persistent)    │                              │
│  └──────────────────┘                              │
└─────────────────────────────────────────────────────┘
```

---

## 🚀 Step-by-Step Setup

### **Step 1: Run Registry Container on Raspberry Pi**

```bash
docker run -d \
  -p 50000:5000 \
  --restart=always \
  --name pi-registry \
  -v pi-docker-local-registry-volume:/var/lib/registry \
  registry:2
```

#### 🔍 **Command Breakdown**:

| Flag | Purpose |
|------|---------|
| `-d` | Run container in **detached mode** (background) |
| `-p 50000:5000` | **Port mapping**: Host port 50000 → Container port 5000 |
| `--restart=always` | Auto-restart container on Pi reboot |
| `--name pi-registry` | Assign a friendly name to the container |
| `-v pi-docker-local-registry-volume:/var/lib/registry` | Create **persistent volume** for image storage |
| `registry:2` | Official Docker registry image (version 2) |

#### 📝 **Key Note**: 
The volume ensures your pushed images **persist** even if the container is deleted or recreated.

---

### **Step 2: Verify Registry is Running**

```bash
# Check container status
docker ps
```

**Output**:
```
CONTAINER ID   IMAGE        COMMAND                  CREATED          STATUS          PORTS                                         NAMES
0aba706ec315   registry:2   "/entrypoint.sh /etc…"   28 seconds ago   Up 26 seconds   0.0.0.0:50000->5000/tcp, :::50000->5000/tcp   pi-registry
```

#### ✅ **What to Look For**:
- STATUS: `Up` (container is running)
- PORTS: `0.0.0.0:50000->5000/tcp` (mapping is correct)

---

### **Step 3: Test Registry API Locally (on Pi)**

```bash
# ❌ This will FAIL (wrong port)
curl http://localhost:5000/v2/

# ✅ This will SUCCEED (correct port)
curl http://localhost:50000/v2/
```

**Successful Output**: `{}`

#### 💡 **Why This Works**:
- The registry container listens on **internal port 5000**
- We mapped it to **host port 50000**
- From the Pi's perspective, we access via `localhost:50000`

---

### **Step 4: Prepare Image on Laptop**

```bash
# Pull a lightweight image for testing
docker pull busybox:latest

# Tag it with your registry address
docker tag busybox:latest 192.168.1.200:50000/busybox:latest
```

#### 🧠 **Mental Map - Image Tagging**:

```
Original Image:              Tagged Image:
busybox:latest       →       192.168.1.200:50000/busybox:latest
                              │         │      │       │
                              │         │      │       └── Tag
                              │         │      └────────── Image name
                              │         └───────────────── Port
                              └─────────────────────────── Registry IP
```

**Why Tag?** Docker needs to know WHERE to push the image. The tag format tells Docker:
- **Host**: 192.168.1.200
- **Port**: 50000
- **Image**: busybox:latest

---

### **Step 5: First Push Attempt (Will Fail)**

```bash
docker push 192.168.1.200:50000/busybox:latest
```

**Error**:
```
Get "https://192.168.1.200:50000/v2/": http: server gave HTTP response to HTTPS client
```

#### ⚠️ **Why This Failed**:

Docker daemon **defaults to HTTPS** for security, but our registry is running on **HTTP** (insecure).

---

### **Step 6: Configure Docker to Allow Insecure Registry**

Create/edit `/etc/docker/daemon.json` on your **laptop**:

```bash
sudo nano /etc/docker/daemon.json
```

**Add this configuration**:
```json
{
  "insecure-registries": ["192.168.1.200:50000"]
}
```

#### 🔒 **Security Note**:
This tells Docker: "Trust this registry even though it's HTTP". 
⚠️ **Only use for local/learning environments!**

---

### **Step 7: Restart Docker Daemon**

```bash
sudo systemctl restart docker
```

#### 📌 **Critical Step**: 
The daemon.json changes **only take effect after restart**.

---

### **Step 8: Push Image (Success!)**

```bash
docker push 192.168.1.200:50000/busybox:latest
```

**Output**:
```
The push refers to repository [192.168.1.200:50000/busybox]
7f74ca728556: Pushed 
latest: digest: sha256:11f85134f388cff5f4c66f9bb4c5942249c1f6f7eb8b3889948d953487b5f7a8 size: 527
```

✅ **Success Indicators**:
- `Pushed` status
- Digest hash generated
- Size information shown

---

### **Step 9: Verify on Raspberry Pi**

```bash
# List all images in registry
curl http://localhost:50000/v2/_catalog
```

**Output**:
```json
{"repositories":["busybox"]}
```

🎉 **Your image is now stored in your private registry!**

---

## 🧠 Key Concepts & Mental Map

### **Port Mapping: 50000:5000 vs 5000:5000**

#### ❓ **Why use 50000:5000 instead of 5000:5000?**

```
Scenario 1: Using 5000:5000
┌─────────────────────────────┐
│     Raspberry Pi Host        │
│                              │
│  Port 5000 (OCCUPIED)        │
│      ▼                       │
│  ┌──────────────┐            │
│  │  Container   │            │
│  │  Port 5000   │            │
│  └──────────────┘            │
└─────────────────────────────┘
Problem: If another service uses port 5000,
Docker will fail to start the container!
```

```
Scenario 2: Using 50000:5000 ✅
┌─────────────────────────────┐
│     Raspberry Pi Host        │
│                              │
│  Port 50000 (AVAILABLE)      │
│      ▼                       │
│  ┌──────────────┐            │
│  │  Container   │            │
│  │  Port 5000   │            │
│  └──────────────┘            │
└─────────────────────────────┘
Benefit: Avoids port conflicts!
```

#### 🎯 **Best Practices**:

| Port Choice | Pros | Cons |
|-------------|------|------|
| **5000:5000** | Standard, memorable | Port conflicts common |
| **50000:5000** | Avoids conflicts, clear separation | Non-standard port |
| **Custom (e.g., 8080:5000)** | Flexible | May conflict with other services |

**Recommendation**: Use a **high port number** (like 50000) to avoid conflicts with common services.

---

### **HTTP vs HTTPS in Docker Registry**

```
┌──────────────────────────────────────────────────────┐
│                HTTPS Registry (Secure)                │
│                                                       │
│  ✅ Encrypted communication                           │
│  ✅ Authentication support                            │
│  ✅ Production-ready                                  │
│  ❌ Requires SSL certificates                         │
│  ❌ More complex setup                                │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│                HTTP Registry (Insecure)               │
│                                                       │
│  ✅ Simple setup (learning)                           │
│  ✅ No certificates needed                            │
│  ⚠️  Unencrypted (visible to network)                 │
│  ⚠️  Requires "insecure-registries" config            │
│  ❌ NOT for production                                │
└──────────────────────────────────────────────────────┘
```

---

### **Docker Registry Workflow**

```
┌─────────────────────────────────────────────────────┐
│             Image Push/Pull Workflow                 │
└─────────────────────────────────────────────────────┘

Push Workflow:
1. docker tag busybox:latest 192.168.1.200:50000/busybox:latest
   └─► Adds registry address to image

2. docker push 192.168.1.200:50000/busybox:latest
   ├─► Docker checks daemon.json for insecure-registries
   ├─► Connects to 192.168.1.200:50000
   ├─► Uploads layers to registry
   └─► Registry stores in /var/lib/registry (volume)

Pull Workflow (from any device):
1. Configure insecure-registries on target device
2. docker pull 192.168.1.200:50000/busybox:latest
   └─► Downloads image from local registry (faster!)
```

---

## 🔧 Troubleshooting Journey

### **Problem 1: Cannot Connect to Registry**

```bash
curl http://localhost:5000/v2/
# Error: Failed to connect to localhost port 5000
```

**Root Cause**: Using wrong port (container internal vs host mapped)

**Solution**: Use the **host-mapped port** (50000)
```bash
curl http://localhost:50000/v2/
```

---

### **Problem 2: HTTP Response to HTTPS Client**

```
Get "https://192.168.1.200:50000/v2/": http: server gave HTTP response to HTTPS client
```

**Root Cause**: Docker daemon requires HTTPS by default

**Solution**: Add registry to insecure-registries list
```json
{
  "insecure-registries": ["192.168.1.200:50000"]
}
```

**Then restart**: `sudo systemctl restart docker`

---

### **Problem 3: Changes Not Taking Effect**

**Symptom**: Modified daemon.json but still getting errors

**Root Cause**: Docker daemon not restarted

**Solution**: Always restart after changing daemon.json
```bash
sudo systemctl restart docker
```

⚠️ **Warning**: This stops all running containers temporarily!

---

## 💡 Important Learnings

### **🔑 Key Takeaways**

1. **Port Mapping Format**: `HOST_PORT:CONTAINER_PORT`
   - External access uses HOST_PORT
   - Internal container uses CONTAINER_PORT

2. **Persistent Storage**: Always use volumes (`-v`) for registry data
   - Without volume: Data lost on container restart
   - With volume: Data persists across restarts

3. **Insecure Registries**: Required for HTTP registries
   - Must be configured on **every client** machine
   - Requires daemon restart to take effect

4. **Testing Strategy**: 
   - Test locally first (`curl localhost:50000/v2/`)
   - Then test from remote machine
   - Verify with catalog endpoint (`/v2/_catalog`)

5. **IP Address**: Use static IP or hostname for registry
   - DHCP may change Pi's IP
   - Consider setting static IP in router

---

### **📚 Mental Models**

#### **Docker Registry as a Library**

```
Registry = Library Building
├── Images = Books
├── Layers = Book Chapters
├── Tags = Book Editions
└── Catalog = Library Index

Push = Donate a book to library
Pull = Borrow a book from library
Volume = Library's storage room (persists even if building renovated)
```

#### **Network Communication**

```
Client Machine                     Raspberry Pi
     │                                  │
     │  1. docker push                  │
     │────────────────────────────────► │
     │     192.168.1.200:50000          │
     │                                  │
     │                            2. Registry
     │                               receives
     │                               layers
     │                                  │
     │  3. Success confirmation         │
     │ ◄────────────────────────────────│
     │                                  │
     │                            4. Stores in
     │                               volume
     │                                  ▼
     │                           /var/lib/registry
```

---

## 🎓 Next Steps for Learning

1. **Add Authentication**: Secure your registry with basic auth
2. **Enable HTTPS**: Set up TLS certificates (self-signed or Let's Encrypt)
3. **Web UI**: Install registry web UI for visual management
4. **Cleanup**: Learn registry garbage collection
5. **Mirror Setup**: Use registry as Docker Hub cache/mirror

---

## 📖 References

- [Docker Registry Documentation](https://docs.docker.com/registry/)
- [Registry Configuration Reference](https://docs.docker.com/registry/configuration/)
- [Deploy a Registry Server](https://docs.docker.com/registry/deploying/)
- [Insecure Registries](https://docs.docker.com/registry/insecure/)

---

## 🏷️ Tags
`#docker` `#raspberry-pi` `#docker-registry` `#container` `#learning` `#devops` `#local-network`

---

**Created**: 2024 | **Purpose**: Learning & Documentation  
**Environment**: Local Network, Non-Production

---

## 📝 Quick Reference Commands

```bash
# Start registry
docker run -d -p 50000:5000 --restart=always --name pi-registry \
  -v pi-docker-local-registry-volume:/var/lib/registry registry:2

# Check registry status
docker ps
curl http://localhost:50000/v2/

# Tag and push image
docker tag IMAGE_NAME 192.168.1.200:50000/IMAGE_NAME:TAG
docker push 192.168.1.200:50000/IMAGE_NAME:TAG

# Pull from registry
docker pull 192.168.1.200:50000/IMAGE_NAME:TAG

# View catalog
curl http://192.168.1.200:50000/v2/_catalog

# View specific image tags
curl http://192.168.1.200:50000/v2/IMAGE_NAME/tags/list

# Stop registry
docker stop pi-registry

# Remove registry (keeps volume/data)
docker rm pi-registry

# Remove volume (deletes all images)
docker volume rm pi-docker-local-registry-volume
```

---

*Happy Learning! 🚀*