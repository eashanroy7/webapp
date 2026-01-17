# Cloud-Native Web Application Platform

A production-ready, cloud-native web application built with Spring Boot and deployed on Google Cloud Platform (GCP). This project demonstrates enterprise-grade architecture with automated CI/CD pipelines, infrastructure as code, autoscaling, and comprehensive observability.

## 🏗️ Architecture Overview

This application implements a scalable, highly available cloud-native architecture including:

- **RESTful API** with Spring Boot backend
- **PostgreSQL** database with automated bootstrapping
- **Custom Machine Images** built with Packer
- **Infrastructure as Code** using Terraform
- **CI/CD Pipeline** with GitHub Actions
- **Auto-scaling** with Google Compute Engine Instance Groups
- **Load Balancing** for high availability
- **Serverless Functions** for event-driven workflows
- **Cloud Observability** with structured logging and monitoring
- **Domain Management** with Cloud DNS

## ✨ Features

### Health Monitoring & Database Connectivity
- Health check endpoint (`/healthz`) with database connectivity validation
- Optimized connection pooling with HikariCP (5-second timeout for rapid failure detection)
- Service returns appropriate HTTP status codes:
  - `200 OK` - Application and database healthy
  - `400 Bad Request` - Invalid query parameters or payload
  - `405 Method Not Allowed` - Unsupported HTTP methods
  - `503 Service Unavailable` - Database connectivity issues

**Testing Health Endpoint:**
```bash
# Valid health check
curl -vvvv http://localhost:8081/healthz

# Invalid request with query params (returns 400)
curl -vvvv http://localhost:8081/healthz?key=value

# Invalid request with payload (returns 400)
curl -vvvv -X GET http://localhost:8081/healthz --data-binary '{"key":"value"}' -H "Content-Type: application/json"

# Unsupported method (returns 405)
curl -vvvv -XPUT http://localhost:8081/healthz
```

### User Management API

Complete user lifecycle management with secure authentication:

#### Create User Account
**Endpoint:** `POST /v1/user`

**Request:**
```json
{
  "first_name": "Jane",
  "last_name": "Doe",
  "password": "skdjfhskdfjhg@1A",
  "username": "jane.doe@example.com"
}
```

**Response (201 Created):**
```json
{
  "id": "d290f1ee-6c54-4b01-90e6-d701748f0851",
  "first_name": "Jane",
  "last_name": "Doe",
  "username": "jane.doe@example.com",
  "account_created": "2016-08-29T09:12:33.001Z",
  "account_updated": "2016-08-29T09:12:33.001Z"
}
```

#### Get User Details
**Endpoint:** `GET /v1/user/self`
- Requires Basic Authentication
- Returns user profile information
- **Responses:** `200 OK`, `401 Unauthorized`

#### Update User Profile
**Endpoint:** `PUT /v1/user/self`
- Requires Basic Authentication
- Allows updates to first name, last name, and password
- **Responses:** `204 No Content`, `400 Bad Request`, `401 Unauthorized`

### Email Verification System

Email verification workflow leveraging serverless architecture ([serverless repository](https://github.com/eashanroy7/serverless)):

**Architecture:**
1. User registration triggers a Pub/Sub message
2. Cloud Function receives the event and generates a time-limited verification token
3. Verification email sent via Mailgun API with unique link
4. User clicks link → redirects to `/verify` endpoint
5. Token validated (2-minute expiration window)
6. User account marked as verified

**Security Features:**
- Time-bound verification tokens (2-minute expiry)
- All API calls require verified email status
- Token stored securely in Cloud SQL database

### Database Management

**Automated Bootstrapping:**
- Utilizes Spring Boot's `CommandLineRunner` interface for automatic database initialization
- Self-healing: Database recreated automatically if deleted
- JPA entity classes with Spring Data JPA for automatic schema generation
- No manual SQL execution required for table restoration

**Database Architecture:**
- Development: Local PostgreSQL installation
- Production: Google Cloud SQL (PostgreSQL 14)
- Private Services Access for secure connectivity
- Network isolation - Cloud SQL not exposed to internet

## 🚀 Infrastructure & Deployment

### Custom Machine Images (Packer)

Automated machine image creation with comprehensive provisioning:

**Image Components:**
- Base OS updates and security patches
- Java Development Kit (JDK 17)
- Maven build tool
- Application artifacts deployed to `/opt/webapp`
- Systemd service configuration for automatic startup
- Google Cloud Ops Agent for logging and monitoring
- Dedicated non-login user (`csye6225`) for security

**Packer Provisioning Pipeline:**
1. System updates and dependency installation
2. Copy compiled application (.jar) to image
3. Configure application directory structure with proper ownership
4. Install and configure systemd service
5. Set up logging infrastructure (`/var/log/webapp/webapp.log`)
6. Install and configure Ops Agent with custom config
7. Enable and start all required services

### Infrastructure as Code (Terraform)

Comprehensive cloud infrastructure managed entirely through Terraform ([tf-gcp-infra repository](https://github.com/eashanroy7/tf-gcp-infra)):

**Network Infrastructure:**
- Custom VPC with public and private subnets
- Public subnet for web application tier
- Private subnet for database tier
- Internet Gateway with explicit routes
- Firewall rules for controlled traffic (port 8081)

**Compute Resources:**
- Google Compute Engine instances from custom images
- Instance Templates for autoscaling
- Managed Instance Groups with health checks
- Application Load Balancer for traffic distribution
- Autoscaling policies for dynamic resource allocation

**Database Infrastructure:**
- Cloud SQL PostgreSQL 14 instance
- Private Services Access configuration
- Database and user provisioning
- Secure credentials management

**Startup Scripts:**
- Dynamic application.properties generation with Cloud SQL credentials
- Automatic service restart on instance launch

**DNS Configuration:**
- Cloud DNS public zone management
- A record configuration pointing to load balancer IP
- Automated record updates on infrastructure changes

### CI/CD Pipeline (GitHub Actions)

Multi-stage continuous integration and deployment workflows:

**Pull Request Validation:**
- Code compilation checks
- Packer template formatting (`packer fmt`)
- Packer template validation
- Integration test execution
- Branch protection rules enforced

**Integration Tests:**
- Test Suite 1: Account creation and retrieval validation
- Test Suite 2: Account update and verification
- All tests run in isolated environment

**Build Pipeline:**
- Maven build with dependency caching
- JAR artifact generation
- Artifact upload to workflow

**Packer Image Creation (on merge to main):**
- Triggered automatically on PR merge
- Uses GCP Service Account credentials (stored as GitHub secrets)
- Validates Packer templates
- Builds custom machine image
- Deploys to Google Cloud Platform

**Rolling Update Deployment:**
- Creates new Instance Template with latest machine image
- Updates Managed Instance Group configuration
- Initiates rolling update (zero-downtime deployment)
- Monitors instance refresh completion
- Workflow reflects deployment status

**Security:**
- GitHub organization secrets for sensitive data
- Fork repository secret access enabled
- Encrypted environment variables

### Domain Configuration

**Custom Domain Setup:**
- Registered domain: `eashanroy.me` (Namecheap)
- Cloud DNS public zone configuration
- Custom nameserver delegation to GCP
- DNS propagation verification:
  ```bash
  dig NS eashanroy.me
  ```

## 📊 Observability & Monitoring

### Structured Logging

**Implementation:**
- SLF4J with Logback for logging framework
- JSON structured log format
- Custom Logback configuration
- Centralized log storage: `/var/log/webapp/webapp.log`

**Cloud Integration:**
- Google Cloud Ops Agent installed on all instances
- Custom Ops Agent configuration (`/etc/google-cloud-ops-agent/config.yaml`)
- Real-time log streaming to Google Cloud Logging
- Log Explorer integration for searching and analysis

**Service Account Permissions:**
- `Logging Admin` - Full logging operations
- `Monitoring Metric Writer` - Custom metrics publishing

### Monitoring

- VM instance health checks
- Application performance metrics
- Database connectivity monitoring
- Autoscaling metrics and triggers

## 🛠️ Technology Stack

### Backend
- **Framework:** Spring Boot
- **Language:** Java 17
- **Build Tool:** Maven
- **ORM:** Spring Data JPA
- **Database:** PostgreSQL 14
- **Connection Pool:** HikariCP

### Infrastructure
- **Cloud Provider:** Google Cloud Platform (GCP)
- **IaC:** Terraform
- **Image Building:** Packer
- **CI/CD:** GitHub Actions
- **Compute:** Google Compute Engine
- **Database:** Google Cloud SQL
- **Load Balancing:** GCP Application Load Balancer
- **Serverless:** Google Cloud Functions
- **Messaging:** Google Pub/Sub
- **DNS:** Google Cloud DNS
- **Logging:** Google Cloud Logging
- **Monitoring:** Google Cloud Monitoring

### Serverless
- **Runtime:** Java
- **Event Source:** Google Pub/Sub
- **Email Service:** Mailgun API
- **Database:** Cloud SQL (via VPC Connector)

### DevOps & Automation
- **Version Control:** Git, GitHub
- **CI/CD:** GitHub Actions
- **Secrets Management:** GitHub Secrets
- **Service Management:** systemd
- **Process Management:** Non-root user execution

## 🔒 Security Features

- **Authentication:** Basic Auth for API endpoints
- **Password Security:** Bcrypt hashing
- **Database Isolation:** Private subnet, no internet exposure
- **Least Privilege:** Dedicated service accounts with minimal IAM roles
- **Non-root Execution:** Application runs as non-login user
- **Network Security:** Firewall rules limiting traffic
- **Secrets Management:** Encrypted GitHub organization secrets
- **Email Verification:** Time-bound tokens for account activation

## 📦 Deployment

The application follows a fully automated deployment workflow:

1. **Code Push:** Developer pushes code to feature branch
2. **PR Creation:** Pull request raised to main branch
3. **Automated Testing:** CI pipeline runs tests and validations
4. **Code Review:** Manual review process
5. **Merge:** PR merged to main branch
6. **Image Build:** Packer automatically builds new machine image
7. **Instance Template:** New template version created
8. **Rolling Update:** Managed Instance Group updated with zero downtime
9. **Health Checks:** New instances validated before old ones terminated
10. **Complete:** New version serving production traffic

## 🌐 Access

- **Domain:** eashanroy.me
- **API Base URL:** http://eashanroy.me:8081
- **Health Check:** http://eashanroy.me:8081/healthz
- **User API:** http://eashanroy.me:8081/v1/user

## 📝 Development Setup

### Prerequisites
- JDK 17
- Maven
- PostgreSQL 14+
- Git

### Local Development

**Database Setup:**
```bash
# Start PostgreSQL (Windows)
net start postgresql-x64-16

# Stop PostgreSQL (Windows)
net stop postgresql-x64-16
```

**Application Configuration:**
Create `src/main/resources/application.properties` with database configuration.

**Build & Run:**
```bash
# Build
mvn clean install

# Run
mvn spring-boot:run
```

**Testing:**
```bash
# Run all tests
mvn test

# Run integration tests
mvn verify
```

## 🔄 Autoscaling & High Availability

- **Instance Groups:** Managed Instance Groups with autoscaling policies
- **Load Balancer:** Application Load Balancer for traffic distribution
- **Health Checks:** Regular health monitoring for instance viability
- **Rolling Updates:** Zero-downtime deployments with progressive rollout
- **Auto-healing:** Automatic instance replacement on health check failures

## 📂 Repository Structure

This project is organized across three repositories:

**Main Application Repository** (Current)
```
webapp/
├── src/
│   ├── main/
│   │   ├── java/               # Spring Boot application code
│   │   └── resources/          # Configuration files
│   └── test/                   # Integration tests
├── packer/                     # Packer templates
├── .github/
│   └── workflows/              # CI/CD pipeline definitions
└── pom.xml                     # Maven configuration
```

**Infrastructure Repository** → [tf-gcp-infra](https://github.com/eashanroy7/tf-gcp-infra)
- Terraform configurations for all GCP infrastructure
- Network, compute, database, and security resources
- Load balancer and autoscaling configurations
- DNS and SSL certificate management

**Serverless Repository** → [serverless](https://github.com/eashanroy7/serverless)
- Google Cloud Function code (Java)
- Email verification logic
- Pub/Sub event handlers
- Mailgun API integration

## 🎯 Project Highlights

- **Production-Ready:** Enterprise-grade architecture with security best practices
- **Fully Automated:** Zero-touch deployment from code commit to production
- **Cloud-Native:** Leverages managed services for scalability and reliability
- **Observable:** Comprehensive logging and monitoring for operational excellence
- **Scalable:** Auto-scaling infrastructure handles variable load
- **Secure:** Multiple layers of security from network to application
- **Event-Driven:** Serverless functions for asynchronous workflows
- **Infrastructure as Code:** 100% reproducible infrastructure

---

**Note:** This project demonstrates proficiency in cloud-native application development, DevOps practices, infrastructure automation, and modern software engineering principles.
