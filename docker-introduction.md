
# Docker-Tutorials

# What is Docker?
Docker is a platform for building, running, and shipping applications in lightweight, portable containers.
Think of it like a way to package your app with everything it needs—code, libraries, dependencies, configuration—into a single unit (a container) that can run anywhere without worrying about “it works on my machine” problems.


## The Difference Between Containers and Virtual Machines
Here’s a clear breakdown of **containers vs. virtual machines (VMs)** — both let you run isolated environments, but they do it in different ways.

---

## **1️⃣ Virtual Machines**

* **What they are:**
  Emulated computers that run on top of a *hypervisor* (like VMware, VirtualBox, Hyper-V).

* **How they work:**
  Each VM has:

  * Its own full **guest operating system**
  * Virtual CPU, memory, storage, and network
  * Apps and dependencies
* **Isolation:**
  Strong (each VM is fully independent with its own OS)
* **Overhead:**
  Heavy — each VM runs an entire OS, so it consumes more resources.

**Example:**

* You run Ubuntu VM and Windows VM on the same laptop — each has its own OS kernel.

---

## **2️⃣ Containers**

* **What they are:**
  Lightweight, isolated environments that **share the host OS kernel**.
* **How they work:**

  * Each container has **its own filesystem, libraries, and dependencies**
  * All containers share the **same operating system kernel**
* **Isolation:**
  Process-level isolation (less heavy than VMs)
* **Overhead:**
  Very light — no full OS inside each container.

**Example:**

* You run a Python app in one container and a PostgreSQL database in another — both share the host’s Linux kernel but are isolated from each other.

---

## 📊 **Quick Comparison Table**

| Feature         | Containers 🐳                                   | Virtual Machines 💻            |
| --------------- | ----------------------------------------------- | ------------------------------ |
| OS per instance | No (share host kernel)                          | Yes (full OS per VM)           |
| Boot time       | Seconds                                         | Minutes                        |
| Resource usage  | Low                                             | High                           |
| Isolation       | Process-level                                   | Hardware-level                 |
| Portability     | High (same container runs anywhere with Docker) | Medium (depends on hypervisor) |
| Storage size    | MBs                                             | GBs                            |
| Speed           | Faster                                          | Slower                         |

---

![Vms Vs Containers](./images/containers-vs-virtual-machines2.jpg)

## 📌 **Analogy**

* **VM:** Renting a whole house (own kitchen, bathroom, electricity) — more privacy but more expensive.
* **Container:** Renting an apartment in a building — you share infrastructure but still have your own space.

---


## Importance of Docker
- Makes it easy for developers to set up a new projects, especially when collaborating with others
- Makes it trivial especially  if the setup is the same across different projects & devs
- Especially if the devs have a similar project setup across different OS systems
- Ensures that the application runs the same way in development, testing, and production environments.
- Simplifies deployment and scaling of applications.
- Helps in isolating applications and their dependencies, reducing conflicts.


# Docker Ecosystem
Consists of **Docker client**, **Docker host**, and **Docker registry**.
These are three different parts of the Docker ecosystem, and understanding how they fit together makes Docker much easier to use.

---

## **1️⃣ Docker Client**

* **What it is:** The command-line tool (`docker`) or Docker Desktop GUI that you use to talk to the Docker daemon.
* **Role:** Sends commands like `docker run`, `docker build`, `docker pull` to the **Docker host**.
* **Example:**
  When you run:

  ```bash
  docker run nginx
  ```

  you are using the **Docker client** to tell the Docker host to run a container.

---

## **2️⃣ Docker Host**

* **What it is:** The machine (physical or virtual) where **Docker daemon** (`dockerd`) runs.
* **Role:** Builds, runs, and manages your Docker containers and images.
* **Contains:**

  * **Docker daemon** – background service handling requests from the client
  * **Images** – blueprints for containers
  * **Containers** – running instances of images
* **Example:**

  * On your laptop, your host might be `localhost`.
  * On a cloud server, your host might be an EC2 instance or a VM.

---

## **3️⃣ Docker Registry**

* **What it is:** A remote storage for Docker images.
* **Role:** Stores and distributes Docker images between different hosts.
* **Examples:**

  * **Public:** Docker Hub (`hub.docker.com`), GitHub Container Registry, Google Artifact Registry
  * **Private:** Your own hosted registry
* **Flow Example:**

  * `docker pull nginx` → Client asks host to pull the `nginx` image from Docker Hub (registry)
  * `docker push myapp` → Client tells host to upload an image to a registry

---

## 🔄 How They Work Together

Imagine the flow when you run:

```bash
docker run hello-world
```

1. **Client** (`docker` CLI) sends request → “Run hello-world”
2. **Host** (Docker daemon) checks if it has the image locally
3. If not, **Host** pulls it from a **Registry** (like Docker Hub)
4. **Host** starts the container and returns output to the **Client**

---

### 📌 Quick Analogy

* **Client** = Waiter (takes your order)
* **Host** = Kitchen (cooks the food — builds/starts containers)
* **Registry** = Pantry (stores ingredients — images)

---

![Docker Architecture](./images/docker-architecture.svg)

# Main Docker Components
1. Docker Container - where a specific component is located
2. Docker Image - Contains all the info needed to create a new container, its the container blueprint
3. Volume - is the storage facility for the containers.
4. Networking - Enables separate containers to talk to each other.


* COMPOSE
 > Is a tool that enables you manage multiple containers and set your entire backend with a single file.

- A simple way to learn docker usage with a simple full-stack app.


# REFERENCE LINKS:
 > Introduction to using Docker
 - https://www.pluralsight.com/guides/create-docker-images-docker-hub
 
 > Build Your Docker Images Automatically When You Push on GitHub
 - https://medium.com/better-programming/build-your-docker-images-automatically-when-you-push-on-github-18e80ece76af
