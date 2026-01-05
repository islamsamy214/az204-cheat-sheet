# AZ-204 Certification Study Guide

> **Comprehensive study materials for the AZ-204: Developing Solutions for Microsoft Azure certification exam**

[![Azure](https://img.shields.io/badge/Azure-0078D4?style=flat&logo=microsoft-azure&logoColor=white)](https://azure.microsoft.com/)
[![AZ-204](https://img.shields.io/badge/Certification-AZ--204-blue)](https://learn.microsoft.com/en-us/credentials/certifications/azure-developer/)
[![Status](https://img.shields.io/badge/Status-Complete-success)](https://github.com/islamsamy214/az204-cheat-sheet)

## 📚 About This Repository

This repository contains **comprehensive study materials** for the Microsoft AZ-204 certification exam. All 11 topics from the official exam outline are covered with detailed explanations, code examples, Azure CLI commands, hands-on exercises, and exam preparation tips.

**Total Content:** ~236,450 words across 11 major topics

**Last Updated:** January 5, 2026

---

## ✅ Course Completion Status

**All 11 topics complete!** 🎉

| # | Topic | Word Count | Status |
|---|-------|-----------|--------|
| 1 | [Implement Azure App Service Web Apps](#topic-1-azure-app-service-web-apps) | 19,143 | ✅ Complete |
| 2 | [Implement Azure Functions](#topic-2-azure-functions) | 10,004 | ✅ Complete |
| 3 | [Work with Azure Blob Storage](#topic-3-azure-blob-storage) | 19,943 | ✅ Complete |
| 4 | [Develop Solutions with Azure Cosmos DB](#topic-4-azure-cosmos-db) | 22,202 | ✅ Complete |
| 5 | [Implement Containerized Solutions](#topic-5-containerized-solutions) | 18,745 | ✅ Complete |
| 6 | [Implement User Authentication & Authorization](#topic-6-authentication--authorization) | 30,230 | ✅ Complete |
| 7 | [Implement Secure Azure Solutions](#topic-7-secure-azure-solutions) | 23,793 | ✅ Complete |
| 8 | [Implement API Management](#topic-8-api-management) | 15,797 | ✅ Complete |
| 9 | [Develop Event-Based Solutions](#topic-9-event-based-solutions) | 35,470 | ✅ Complete |
| 10 | [Develop Message-Based Solutions](#topic-10-message-based-solutions) | 21,588 | ✅ Complete |
| 11 | [Monitor, Troubleshoot, and Optimize Solutions](#topic-11-monitor-troubleshoot-optimize) | 17,732 | ✅ Complete |

---

## 📖 Topics Overview

### Topic 1: Azure App Service Web Apps
**Directory:** [`01-implement-azure-app-service-web-apps/`](./01-implement-azure-app-service-web-apps/)

Learn to create, configure, and deploy web applications using Azure App Service.

**Key Concepts:**
- App Service plans and pricing tiers
- Deployment methods (Git, ZIP, ARM templates)
- Configuration and app settings
- Scaling (vertical and horizontal)
- Deployment slots and slot swapping
- Custom domains and SSL certificates
- Networking features (VNET integration, private endpoints)

---

### Topic 2: Azure Functions
**Directory:** [`02-implement-azure-functions/`](./02-implement-azure-functions/)

Master serverless computing with Azure Functions.

**Key Concepts:**
- Hosting plans (Consumption, Premium, Dedicated)
- Triggers and bindings (HTTP, Timer, Blob, Queue, Event Hub, Cosmos DB)
- Durable Functions (orchestration, activity, entity patterns)
- Function development (in-portal, VS Code, Visual Studio)
- Configuration and monitoring
- Security and authentication

---

### Topic 3: Azure Blob Storage
**Directory:** [`03-work-with-azure-blob-storage/`](./03-work-with-azure-blob-storage/)

Work with Azure's scalable object storage solution.

**Key Concepts:**
- Storage account types and performance tiers
- Blob types (Block, Append, Page)
- Access tiers (Hot, Cool, Cold, Archive)
- Lifecycle management policies
- Azure Storage SDK operations
- SAS tokens and access control
- Blob versioning and soft delete
- Static website hosting

---

### Topic 4: Azure Cosmos DB
**Directory:** [`04-develop-solutions-that-use-azure-cosmos-db/`](./04-develop-solutions-that-use-azure-cosmos-db/)

Build globally distributed, multi-model database applications.

**Key Concepts:**
- API options (SQL, MongoDB, Cassandra, Gremlin, Table)
- Consistency levels (Strong, Bounded Staleness, Session, Consistent Prefix, Eventual)
- Partition strategies and partition keys
- Request Units (RU/s) and throughput
- Change Feed processing
- Stored procedures, triggers, and UDFs
- Global distribution and multi-region writes
- SDK operations and best practices

---

### Topic 5: Containerized Solutions
**Directory:** [`05-implement-containerized-solutions/`](./05-implement-containerized-solutions/)

Deploy and manage containerized applications on Azure.

**Key Concepts:**
- Azure Container Instances (ACI)
- Azure Container Apps (ACA)
- Azure Kubernetes Service (AKS)
- Docker image creation and management
- Azure Container Registry (ACR)
- Container networking and scaling
- Microservices architecture
- Dapr integration

---

### Topic 6: Authentication & Authorization
**Directory:** [`06-implement-user-authentication-authorization/`](./06-implement-user-authentication-authorization/)

Secure applications with Microsoft Identity Platform.

**Key Concepts:**
- Microsoft Entra ID (Azure AD)
- OAuth 2.0 and OpenID Connect
- Application registration
- Authentication flows (Authorization Code, Client Credentials, On-Behalf-Of)
- MSAL (Microsoft Authentication Library)
- Token acquisition and validation
- Delegated vs application permissions
- Multi-tenant applications
- Conditional Access

---

### Topic 7: Secure Azure Solutions
**Directory:** [`07-implement-secure-azure-solutions/`](./07-implement-secure-azure-solutions/)

Implement security best practices for Azure solutions.

**Key Concepts:**
- Azure Key Vault (secrets, keys, certificates)
- Managed Identity (System-assigned, User-assigned)
- Azure App Configuration
- Secure connection strings and secrets
- Certificate management
- Key rotation strategies
- RBAC and access policies
- Encryption at rest and in transit

---

### Topic 8: API Management
**Directory:** [`08-implement-api-management/`](./08-implement-api-management/)

Create, publish, and manage APIs at scale.

**Key Concepts:**
- API Management service tiers
- Products, APIs, and operations
- Policies (inbound, outbound, backend, on-error)
- Rate limiting and quotas
- Request/response transformation
- Backend services configuration
- Developer portal
- API versioning and revisions
- Authentication (subscription keys, OAuth, JWT validation)

---

### Topic 9: Event-Based Solutions
**Directory:** [`09-develop-event-based-solutions/`](./09-develop-event-based-solutions/)

Build reactive applications with Azure event services.

**Key Concepts:**
- Azure Event Grid (event routing, system/custom topics)
- Azure Event Hubs (big data streaming, partitions, consumer groups)
- Event Grid vs Event Hubs comparison
- Event schemas and CloudEvents
- Event handlers and subscriptions
- Event filtering and delivery
- Capture and processing patterns
- Dead-letter handling

---

### Topic 10: Message-Based Solutions
**Directory:** [`10-develop-message-based-solutions/`](./10-develop-message-based-solutions/)

Implement reliable messaging patterns with Azure Service Bus and Queue Storage.

**Key Concepts:**
- Azure Service Bus (queues, topics, subscriptions)
- Azure Queue Storage
- Message sessions and correlation
- Dead-letter queues
- Scheduled messages and deferrals
- Transactions and duplicate detection
- FIFO guarantee
- Filtering and routing
- Comparison: Service Bus vs Queue Storage

---

### Topic 11: Monitor, Troubleshoot, Optimize
**Directory:** [`11-monitor-troubleshoot-optimize-solutions/`](./11-monitor-troubleshoot-optimize-solutions/)

Monitor and optimize Azure applications with Application Insights.

**Key Concepts:**
- Azure Monitor ecosystem
- Application Insights (APM)
- Telemetry types (requests, dependencies, exceptions, traces)
- Log-based vs standard metrics
- Autoinstrumentation vs SDK
- Live Metrics Stream
- Application Map and distributed tracing
- Availability tests (Standard, Custom TrackAvailability)
- KQL (Kusto Query Language)
- Smart Detection and alerts
- Performance optimization

---

## 🎯 What You'll Learn

This study guide covers all exam objectives for AZ-204:

### Core Azure Services
- **Compute:** App Service, Functions, Containers (ACI, ACA, AKS)
- **Storage:** Blob Storage, Cosmos DB, Queue Storage, Table Storage
- **Messaging:** Event Grid, Event Hubs, Service Bus
- **Security:** Key Vault, Managed Identity, Entra ID
- **Integration:** API Management, Logic Apps
- **Monitoring:** Application Insights, Azure Monitor

### Development Skills
- ✅ Azure SDK usage (.NET, Python, JavaScript)
- ✅ Azure CLI commands
- ✅ ARM templates and Bicep
- ✅ Authentication and authorization flows
- ✅ Distributed tracing and monitoring
- ✅ Serverless architectures
- ✅ Microservices patterns
- ✅ Security best practices

---

## 📂 Repository Structure

```
az204-cheat-sheet/
├── 01-implement-azure-app-service-web-apps/
│   ├── 01-introduction.md
│   ├── 02-explore-app-service.md
│   ├── 03-configure-app-settings.md
│   └── ...
├── 02-implement-azure-functions/
│   ├── 01-introduction.md
│   ├── 02-explore-azure-functions.md
│   └── ...
├── 03-work-with-azure-blob-storage/
├── 04-develop-solutions-that-use-azure-cosmos-db/
├── 05-implement-containerized-solutions/
├── 06-implement-user-authentication-authorization/
├── 07-implement-secure-azure-solutions/
├── 08-implement-api-management/
├── 09-develop-event-based-solutions/
├── 10-develop-message-based-solutions/
├── 11-monitor-troubleshoot-optimize-solutions/
│   ├── 01-introduction-to-monitoring.md
│   ├── 02-explore-application-insights.md
│   ├── 03-log-based-metrics-standard-metrics.md
│   ├── 04-instrument-applications-for-monitoring.md
│   ├── 05-select-configure-availability-tests.md
│   ├── 06-troubleshoot-with-application-map.md
│   ├── 07-monitor-analyze-metrics-logs-traces.md
│   ├── 08-exercise-monitor-application.md
│   └── 09-summary-exam-preparation.md
└── README.md
```

---

## 🚀 How to Use This Guide

### Study Approach

1. **Sequential Learning:** Work through topics 1-11 in order
2. **Hands-On Practice:** Complete all exercises (Azure free tier available)
3. **Review Summaries:** Each topic ends with exam preparation tips
4. **Practice Labs:** Use Azure Portal, CLI, and SDKs
5. **Take Notes:** Personalize your study experience

### Study Schedule Recommendation

- **Week 1-2:** Topics 1-3 (Compute and Storage fundamentals)
- **Week 3-4:** Topics 4-5 (Databases and Containers)
- **Week 5-6:** Topics 6-7 (Security and Identity)
- **Week 7-8:** Topics 8-9 (API Management and Events)
- **Week 9-10:** Topics 10-11 (Messaging and Monitoring)
- **Week 11:** Review all topics and take practice exams

### Each Topic Contains

✅ **Comprehensive explanations** with real-world context  
✅ **ASCII diagrams** for visual learning  
✅ **Azure CLI commands** ready to run  
✅ **Code examples** in C#, Python, and JavaScript  
✅ **Comparison tables** for decision-making  
✅ **Best practices** and common pitfalls  
✅ **Hands-on exercises** with step-by-step instructions  
✅ **Exam tips** aligned with AZ-204 objectives  
✅ **Quick reference guides** for review  

---

## 🎓 Exam Information

### AZ-204: Developing Solutions for Microsoft Azure

**Exam Duration:** 120 minutes  
**Number of Questions:** 40-60  
**Passing Score:** 700/1000  
**Question Types:** Multiple choice, drag-and-drop, case studies  
**Cost:** $165 USD  

### Exam Domains

| Domain | Weight |
|--------|--------|
| Develop Azure compute solutions | 25-30% |
| Develop for Azure storage | 15-20% |
| Implement Azure security | 20-25% |
| Monitor, troubleshoot, and optimize Azure solutions | 15-20% |
| Connect to and consume Azure services and third-party services | 15-20% |

### Official Resources

- **Exam Page:** [AZ-204 Certification](https://learn.microsoft.com/en-us/credentials/certifications/azure-developer/)
- **Study Guide:** [Official Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-204)
- **Microsoft Learn:** [Learning Paths](https://learn.microsoft.com/en-us/training/browse/?products=azure&roles=developer)
- **Practice Assessment:** Available on Microsoft Learn
- **Exam Sandbox:** Experience the exam interface

---

## 💡 Study Tips

### Before the Exam

1. ✅ Complete all 11 topics in this repository
2. ✅ Practice hands-on labs in Azure Portal
3. ✅ Take at least 2-3 practice exams
4. ✅ Review all "Exam Tips" sections
5. ✅ Understand service limits and quotas
6. ✅ Know when to use which service (decision matrices)
7. ✅ Memorize Azure CLI common commands
8. ✅ Understand pricing models

### During the Exam

- 📝 Read questions carefully (watch for "NOT", "EXCEPT")
- ⏱️ Manage time (1.5-2 minutes per question)
- 🎯 Answer what you know first, flag difficult questions
- 🔍 Look for keywords that hint at specific services
- 💭 Eliminate obviously wrong answers
- 🤔 In case studies, take notes on requirements

### Key Services to Master

**High Priority:**
- App Service, Functions, Container Apps
- Blob Storage, Cosmos DB
- Key Vault, Managed Identity
- Application Insights, Azure Monitor
- Service Bus, Event Grid

**Medium Priority:**
- API Management
- Azure Container Registry
- Event Hubs
- Microsoft Entra ID (OAuth flows)

**Lower Priority:**
- Logic Apps
- Azure CDN
- Azure Cache for Redis

---

## 🛠️ Prerequisites

### Required Knowledge
- C#, Python, or JavaScript programming
- REST API concepts
- HTTP protocol basics
- JSON data format
- Git version control
- Basic cloud computing concepts

### Azure Account
- [Azure Free Account](https://azure.microsoft.com/free/) (12 months free services)
- Credit card required (won't be charged for free tier)
- $200 credit for first 30 days

### Tools
- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)
- [VS Code](https://code.visualstudio.com/) with Azure extensions
- [.NET SDK](https://dotnet.microsoft.com/download) (for C# examples)
- [Node.js](https://nodejs.org/) (for JavaScript examples)
- [Python 3.8+](https://www.python.org/downloads/) (for Python examples)

---

## 🤝 Contributing

This is a personal study guide, but feedback is welcome!

If you find errors or have suggestions:
1. Open an issue
2. Submit a pull request
3. Share your feedback

---

## 📜 License

This repository is for educational purposes. All Azure documentation references are property of Microsoft.

---

## 🙏 Acknowledgments

- Microsoft Learn documentation
- Azure documentation team
- AZ-204 community and study groups

---

## 📞 Contact

**Repository Owner:** [@islamsamy214](https://github.com/islamsamy214)

---

## 🎯 Final Checklist

Ready to take the exam? Verify you can:

- [ ] Create and configure App Service and Functions
- [ ] Work with Blob Storage and Cosmos DB using SDKs
- [ ] Implement authentication with Microsoft Identity Platform
- [ ] Secure applications with Key Vault and Managed Identity
- [ ] Deploy containerized solutions (ACI, ACA)
- [ ] Configure API Management policies
- [ ] Implement event-driven solutions (Event Grid, Event Hubs)
- [ ] Use Service Bus for reliable messaging
- [ ] Monitor applications with Application Insights
- [ ] Write KQL queries for log analysis
- [ ] Configure availability tests and alerts
- [ ] Use Azure CLI for all major services

---

**⭐ If this repository helped you pass AZ-204, please star it!**

**🎉 Good luck with your certification!**

---

*Last Updated: January 5, 2026*  
*Status: All 11 topics complete (236,450+ words)*  
*Ready for AZ-204 exam preparation*
