<img src="./architecture.excalidraw.png" alt="Voting App Architecture" width="600" />

---

# DevOps Voting Application

End-to-end, containerized voting application running on **AWS EC2**, showcasing a microservices architecture, two-tier Docker networking, and a complete CI/CD-ready setup.

> Note: The application is deployed and hosted on an **Amazon EC2 instance** which acts as the main server for all Dockerized services.

---

## 1. Overview

This project is a distributed voting application where users can vote between two options (e.g., cats vs. dogs) and see **real-time results**. It is designed as a practical DevOps lab to demonstrate:

- Multi-language microservices (Python, Node.js, .NET)
- Containerization with Docker
- Orchestration with Docker Compose
- Health checks and dependency management
- Two-tier network design (frontend vs. backend)
- Deployment to **AWS EC2** as the hosting server

The core user flow:

1. Users open the **Vote UI** and submit a vote.
2. The vote is sent to **Redis** as a message.
3. The **Worker** consumes votes from Redis and stores aggregates in **PostgreSQL**.
4. The **Result UI** reads from PostgreSQL and updates the results view in real time.

---

## 2. Architecture

### 2.1 Services

- **Vote service** (`/vote`)
  - Python **Flask** web application
  - Serves the voting interface (option A vs. option B)
  - Exposed on port **8080**

- **Result service** (`/result`)
  - **Node.js** application
  - Shows aggregated votes with **real-time updates** (WebSockets)
  - Exposed on port **8081**

- **Worker service** (`/worker`)
  - **.NET** worker application
  - Listens to Redis, processes vote messages, updates PostgreSQL

- **Redis**
  - Acts as a **message broker** for votes

- **PostgreSQL**
  - Stores final vote counts and acts as the system of record

- **Seed service** (`/seed-data`)
  - Utility container used to generate **load/test data**
  - Uses Apache Bench (ab) and helper scripts to simulate votes

### 2.2 Network Topology (Two-Tier)

The project follows a **two-tier network architecture** for better isolation and security:

- **Frontend network**
  - `vote` service (port 8080)
  - `result` service (port 8081)
  - Accessible from outside the Docker host (via EC2’s public IP and security group rules)

- **Backend network**
  - `worker` service
  - `redis` service
  - `postgres` service
  - Internal-only communication; not exposed directly to the public internet

This ensures databases and queues are kept private, while only the web-facing services are reachable from users.

### 2.3 Hosting on AWS EC2

The entire stack (Docker Engine + Docker Compose + app services) runs on an **AWS EC2 instance**, which acts as the main server.

At a high level:

1. Provision an EC2 instance (e.g., Amazon Linux 2 / Ubuntu) with Docker & Docker Compose installed.
2. Clone this repository onto the EC2 instance.
3. Run `docker compose up` to start all services.
4. Expose ports **8080** and **8081** in the EC2 security group so clients can access the frontend services.

---

## 3. Project Structure

```text
├── docker-compose.yml        # Orchestration for all services
├── healthchecks/             # Shell scripts for service health checks
│   ├── postgres.sh
│   └── redis.sh
├── vote/                     # Python Flask vote UI
│   ├── app.py
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── static/
│   └── templates/
├── result/                   # Node.js result UI
│   ├── server.js
│   ├── Dockerfile
│   ├── package.json
│   └── views/
├── worker/                   # .NET worker service
│   ├── Program.cs
│   ├── Worker.csproj
│   └── Dockerfile
├── seed-data/                # Data generator & load testing
│   ├── make-data.py
│   ├── generate-votes.sh
│   └── Dockerfile
└── README.md
```

---

## 4. Tech Stack

- **Backend & Services**
  - Python 3 + Flask (`vote`)
  - Node.js (`result`)
  - .NET Worker (`worker`)
  - Redis (queue)
  - PostgreSQL (database)

- **Infrastructure & DevOps**
  - Docker & Docker Compose
  - Two-tier networking (frontend/backend)
  - Shell-based health checks
  - Hosted on **AWS EC2**

---

## 5. Running the Application Locally

> Prerequisites: Docker and Docker Compose installed.

From the project root:

```bash
docker compose up --build
```

Then access:

- Vote UI: `http://localhost:8080`
- Result UI: `http://localhost:8081`

To run in detached mode:

```bash
docker compose up -d --build
```

To stop and clean up containers:

```bash
docker compose down
```

---

## 6. Running on AWS EC2

Below is the high-level flow used to host this project on an EC2 instance:

1. **Provision EC2 instance**
   - Choose a Linux-based AMI (Ubuntu / Amazon Linux 2)
   - Open inbound ports **22** (SSH), **8080** and **8081** (HTTP to the app)

2. **Install Docker & Docker Compose** on EC2
   - Install Docker Engine
   - Add your user to the `docker` group
   - Install Docker Compose plugin or standalone binary

3. **Deploy the application**
   - SSH into the EC2 instance
   - Clone this repository
   - Run:
     ```bash
     docker compose up -d --build
     ```

4. **Access the app via EC2 public IP**
   - Vote UI: `http://<EC2_PUBLIC_IP>:8080`
   - Result UI: `http://<EC2_PUBLIC_IP>:8081`

The EC2 instance acts as the **single server** hosting all containers and networks for this project.

---

## 7. Health Checks

The project includes health check scripts under `healthchecks/`:

- `healthchecks/redis.sh` – checks Redis readiness
- `healthchecks/postgres.sh` – checks PostgreSQL readiness

These scripts are wired into the Docker Compose configuration to make sure:

- Dependent services only start when Redis/PostgreSQL are healthy
- The application starts in a predictable, reliable order

---

## 8. Seed Data & Load Testing

The `seed-data` service can be used to generate test traffic and pre-populate the database.

Components:

- `make-data.py` – generates URL-encoded payload files (`posta`, `postb`)
- `generate-votes.sh` – uses Apache Bench (ab) to send test votes to the vote service, e.g.:
  - ~2000 votes for option A
  - ~1000 votes for option B

Example usage (once the stack is up):

```bash
docker compose run --rm seed
```

Or, using a dedicated profile:

```bash
docker compose --profile seed up
```

After seeding, you should see skewed results in the result UI that reflect the generated traffic.

---

## 9. Development Notes

- The vote service limits each browser to a single vote via basic client-side logic.
- The result service uses WebSockets for real-time chart updates.
- The worker runs continuously and is designed to be stateless; scaling it horizontally is straightforward.
- Startup order and readiness are managed using Docker health checks and `depends_on` conditions.

---

## 10. Future Improvements

Some ideas to extend this project:

- Add CI/CD (e.g., GitHub Actions) to build and push images automatically.
- Use **Amazon RDS** for managed PostgreSQL instead of a container.
- Add monitoring and alerting (Prometheus/Grafana or CloudWatch).
- Add HTTPS termination using an AWS Load Balancer or Nginx reverse proxy.
- Parameterize configuration via environment variables / SSM Parameter Store.

---

## 11. Summary

This repository demonstrates a complete **DevOps-ready microservices application** deployed on an **AWS EC2 server**, featuring:

- Polyglot microservices (Python, Node.js, .NET)
- Docker and Docker Compose orchestration
- Two-tier network architecture
- Health checks and dependency management
- Seed data and load testing utilities

It’s a solid foundation for learning, experimenting, and extending real-world DevOps practices.