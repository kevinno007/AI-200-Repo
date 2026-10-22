# AI-200: Develop AI Cloud Solutions on Azure

> **Certification track:** Microsoft Certified: Azure AI Cloud Developer Associate  
> **Course:** AI-200T00-A — Develop AI cloud solutions on Azure  
> **Level:** Intermediate | **Role:** Developer | **Platform:** Microsoft Azure

This repository is my learning journal and hands-on workspace for **AI-200**. I will use it to track learning-path progress, capture implementation notes, add labs/code samples, and record certification-preparation updates.

## What is AI-200?

AI-200 focuses on the **cloud-development foundation around production AI applications on Azure**. Rather than focusing only on model building, the course teaches developers how to host AI applications, work with AI-oriented data stores, connect backend services, protect configuration and secrets, and monitor/troubleshoot solutions.

Microsoft describes the course as covering:

- Azure compute and containerization patterns
- Azure Container Apps and Azure Kubernetes Service (AKS)
- Serverless APIs with Azure Functions
- Event-driven and message-based integration with Azure Service Bus and Azure Event Grid
- AI-ready data solutions with Azure Cosmos DB for NoSQL
- Azure Database for PostgreSQL with `pgvector`
- Azure Managed Redis for caching, streaming, and vector search
- Security, configuration, observability, monitoring, and troubleshooting

The official AI-200T00-A course is listed as a **5-day, intermediate developer course**.

## Who should take this course?

AI-200 is best suited to **developers who build backend and AI-driven applications on Azure** and want practical experience with production cloud architecture.

You are a strong fit if you:

- Build or plan to build AI-enabled backend services on Azure.
- Need to deploy workloads using containers, Azure Container Apps, or AKS.
- Work with Azure data services used by AI applications, including vector-enabled stores.
- Need asynchronous/event-driven integration using Service Bus or Event Grid.
- Want practical experience securing, monitoring, and troubleshooting distributed Azure applications.
- Plan to earn the **Microsoft Certified: Azure AI Cloud Developer Associate** certification.

### Recommended background

For certification preparation, Microsoft expects candidates to be proficient with:

- Azure SDKs and third-party SDKs used with Azure
- Azure data-management services
- Azure monitoring and troubleshooting
- Azure messaging and eventing
- Vector databases
- Python programming
- Containerized applications on Azure

> This is an **intermediate** path. Basic Azure knowledge plus software-development experience will make the learning paths significantly easier to follow.

---

## Learning Paths

The official AI-200T00-A syllabus currently contains **9 learning paths and 27 modules**.

| # | Learning Path | Modules | Main focus |
|---|---|---:|---|
| 1 | Implement container application hosting on Azure | 3 | Container images and Azure Container Registry foundations |
| 2 | Deploy and manage apps on Azure Container Apps | 4 | Running and managing containerized applications |
| 3 | Deploy and monitor applications on Azure Kubernetes Service | 3 | AKS deployment and monitoring |
| 4 | Develop AI solutions with Azure Cosmos DB for NoSQL | 3 | NoSQL data patterns for AI solutions |
| 5 | Develop AI solutions with Azure Database for PostgreSQL | 3 | PostgreSQL and vector-enabled AI data scenarios |
| 6 | Enhance AI solutions with Azure Managed Redis | 3 | Caching, streaming, and vector-search scenarios |
| 7 | Integrate backend services for AI solutions | 4 | Service integration, messaging, and event-driven patterns |
| 8 | Manage application secrets and configuration for AI solutions | 2 | Azure Key Vault and secure configuration |
| 9 | Observe and troubleshoot apps on Azure | 2 | Monitoring, diagnostics, and troubleshooting |

## Recommended Learning Flow

The learning paths form a practical journey from **hosting → data → integration → security → operations**:

```text
[1. Container hosting foundations]
             |
             v
[2. Azure Container Apps]
             |
             v
[3. Azure Kubernetes Service]
             |
             v
[4. Cosmos DB for NoSQL]
             |
             v
[5. PostgreSQL + vector scenarios]
             |
             v
[6. Azure Managed Redis]
             |
             v
[7. Backend service integration]
             |
             v
[8. Secrets & configuration]
             |
             v
[9. Observe & troubleshoot]
             |
             v
[AI-200 Exam Preparation]
```

### Phase 1 — Host and run applications

Start with container fundamentals, then move through **Azure Container Apps** and **AKS**. The goal is to understand where and how an AI-backed service runs before adding data and integrations.

### Phase 2 — Build the AI data layer

Work through **Azure Cosmos DB for NoSQL**, **Azure Database for PostgreSQL**, and **Azure Managed Redis**. Together these paths cover persistence plus patterns relevant to modern AI workloads, including vector-aware data scenarios and caching.

### Phase 3 — Connect the architecture

Use backend Azure services to create loosely coupled workflows. This is where messaging and event-driven services such as **Azure Service Bus** and **Azure Event Grid** fit into the application architecture.

### Phase 4 — Secure the application

Move secrets and configuration out of application code. The syllabus specifically includes **Azure Key Vault** as part of the secrets/configuration learning path.

### Phase 5 — Operate in production

Finish with observability and troubleshooting using **Azure Monitor**. This closes the development lifecycle by covering what happens after deployment.

---

## Certification Details

Successful preparation leads toward:

### Microsoft Certified: Azure AI Cloud Developer Associate

The certification validates the ability to **design, build, and implement AI solutions on Azure**, with emphasis on backend services, scalable architectures, and the full development lifecycle.

| Item | Details |
|---|---|
| Exam | **AI-200** |
| Certification | **Microsoft Certified: Azure AI Cloud Developer Associate** |
| Level | Intermediate |
| Role | Developer |
| Subjects | Application development and artificial intelligence |
| Exam duration | **120 minutes** |
| Delivery | Proctored; interactive components may be included |
| Exam price shown by Microsoft | **US$165**, with actual pricing based on the country/region where the exam is proctored |
| Practice assessment | Microsoft currently states that it is **not yet available** and is generally released after the exam exits beta/GA timing conditions |

### Skills assessed

Microsoft currently groups the AI-200 exam into four major areas:

1. **Develop containerized solutions on Azure**
2. **Develop AI solutions by using Azure data management services**
3. **Connect to and consume Azure services**
4. **Secure, monitor, troubleshoot Azure solutions**

> **Important:** Certification pages and exam objectives can change. Always compare this repository against the latest Microsoft AI-200 Study Guide before scheduling the exam.

---


## Suggested Repository Structure

```text
.
├── README.md
├── 01-container-hosting/
├── 02-container-apps/
├── 03-aks/
├── 04-cosmos-db/
├── 05-postgresql/
├── 06-managed-redis/
├── 07-backend-integration/
├── 08-security-configuration/
├── 09-observability/
├── labs/
└── exam-prep/
```

For every learning path, I plan to capture:

- Key concepts and Azure services
- Architecture/design notes
- Useful CLI, SDK, and configuration examples
- Hands-on exercises and lessons learned
- Common errors and troubleshooting notes
- Certification-focused revision points

---

## Official Resources

- [AI-200T00-A: Develop AI cloud solutions on Azure](https://learn.microsoft.com/en-us/training/courses/ai-200t00/)
- [Microsoft Certified: Azure AI Cloud Developer Associate](https://learn.microsoft.com/en-us/credentials/certifications/azure-ai-cloud-developer-associate/)
- [AI-200 Study Guide](https://aka.ms/AI200-StudyGuide)
- [Microsoft Exam Sandbox](https://go.microsoft.com/fwlink/?linkid=2226877)

## Notes

This repository is a **personal learning and revision resource** and is not an official Microsoft repository. Certification requirements, exam content, availability, and pricing can change. Use the official Microsoft Learn pages above as the source of truth.

---

_Last reviewed against Microsoft Learn course and certification pages: **22 October 2026**._
