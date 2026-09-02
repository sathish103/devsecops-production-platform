🚀 DevSecOps Production Platform

Production-grade DevSecOps CI/CD platform on AWS EKS with GitHub Actions, Amazon ECR, Kubernetes Gateway API, AWS Load Balancer Controller, blue-green deployments, immutable releases, security gates, and automatic rollback.








📌 What Is This Project?

This project implements an end-to-end DevSecOps production delivery platform for a containerized microservices application running on Amazon EKS.

The platform automates the complete path from a developer commit to production:

```text
Developer
   │
   │ git push origin main
   ▼
GitHub
   │
   ▼
GitHub Actions CI
   │
   ├── Build & Test
   ├── Gitleaks Secret Scan
   ├── Docker Image Build
   ├── Trivy Vulnerability Scan
   ├── GitHub OIDC → AWS IAM
   └── Push immutable images to Amazon ECR
   │
   ▼
CI SUCCESS
   │
   ▼
GitHub Actions CD
   │
   ├── Detect active environment
   ├── Identify inactive environment
   ├── Deploy new release to inactive environment
   ├── Wait for Kubernetes rollout
   ├── Validate replicas
   ├── Validate image SHA
   ├── Switch production traffic: 100%
   ├── Verify production
   └── Roll back automatically if final verification fails
   │
   ▼
Amazon EKS Production
```
The key design principle is:

Deploy first → validate completely → switch 100% traffic → verify → rollback if required.

The current implementation uses blue-green deployment with a direct 100% traffic switch. It does not use gradual canary percentages such as 10%, 25%, 50%, or 75%.

🏗️ Architecture Overview

                                  ┌──────────────────────┐
                                  │      Developer       │
                                  └──────────┬───────────┘
                                             │
                                      git push origin main
                                             │
                                             ▼
                                  ┌──────────────────────┐
                                  │   GitHub Repository  │
                                  │       main branch    │
                                  └──────────┬───────────┘
                                             │
                                             ▼
                         ┌────────────────────────────────────┐
                         │          GitHub Actions CI          │
                         │                                    │
                         │  1. Build & Test                   │
                         │  2. Gitleaks                       │
                         │  3. Docker Build                   │
                         │  4. Trivy Scan                     │
                         │  5. GitHub OIDC → AWS IAM          │
                         │  6. Push images → Amazon ECR       │
                         └──────────────────┬─────────────────┘
                                            │
                                      CI SUCCESS
                                            │
                                            ▼
                         ┌────────────────────────────────────┐
                         │          GitHub Actions CD          │
                         │                                    │
                         │  Detect Active Environment         │
                         │             │                      │
                         │             ▼                      │
                         │  Deploy to Inactive Environment   │
                         │             │                      │
                         │             ▼                      │
                         │  Rollout + Image Validation       │
                         │             │                      │
                         │             ▼                      │
                         │       100% Traffic Switch          │
                         │             │                      │
                         │             ▼                      │
                         │     Production Verification        │
                         │             │                      │
                         │        ┌────┴────┐                 │
                         │        │         │                 │
                         │     SUCCESS    FAILURE              │
                         │        │         │                 │
                         │        ▼         ▼                 │
                         │      DONE     ROLLBACK              │
                         └──────────────────┬─────────────────┘
                                            │
                                            ▼
                         ┌────────────────────────────────────┐
                         │             Amazon EKS              │
                         │                                    │
                         │       Production Gateway            │
                         │              │                     │
                         │          HTTPRoutes                 │
                         │              │                     │
                         │       ┌──────┴──────┐              │
                         │       │             │              │
                         │       ▼             ▼              │
                         │     BLUE          GREEN            │
                         │  devsecops-blue devsecops-green   │
                         └────────────────────────────────────┘

## 🧩 Application Components

The platform contains four application workloads:

| Component | Container Image | Production Port |
| --- | --- | --- |
| Frontend | devsecops-frontend | 80 |
| User Service | devsecops-user-service | 8081 |
| Product Service | devsecops-product-service | 8082 |
| Order Service | devsecops-order-service | 8083 |

Each production workload exists in both environments:

```text
devsecops-blue
├── frontend-blue
├── user-service-blue
├── product-service-blue
└── order-service-blue

devsecops-green
├── frontend-green
├── user-service-green
├── product-service-green
└── order-service-green
```

## 🔐 Technology Stack

| Layer | Technology |
| --- | --- |
| Source Control | GitHub |
| CI/CD | GitHub Actions |
| Cloud | AWS |
| Kubernetes | Amazon EKS |
| Container Runtime | Docker |
| Container Registry | Amazon ECR |
| AWS Authentication | GitHub OIDC + IAM |
| Build | Maven |
| Java | Java 21 |
| Secret Detection | Gitleaks |
| Container Security | Trivy |
| Traffic Management | Kubernetes Gateway API |
| Gateway | Kubernetes Gateway |
| Routing | Kubernetes HTTPRoute |
| AWS Integration | AWS Load Balancer Controller |
| AWS Load Balancing | Application Load Balancer |
| Deployment Strategy | Blue-Green |
| Release Version | Git Commit SHA |

🌿 Branch Strategy

The production delivery flow is centered around the main branch.

```text
Developer
   │
   │ git push
   ▼
 main
   │
   ▼
 GitHub Actions CI
   │
   ▼
 ECR
   │
   ▼
 GitHub Actions CD
   │
   ▼
 Production
```
The important production rule is:

Only successful CI execution from a push to main
can trigger production CD.

The CD workflow explicitly checks:

CI conclusion = success
CI event       = push
CI branch      = main

This prevents other workflow executions from accidentally deploying to production.

**🔄 CI — Continuous Integration**

CI Objective

The CI pipeline answers:

"Is this source code safe, tested, buildable, and ready to become a production artifact?"

The pipeline performs:

```text
Source Code
    │
    ▼
Build & Test
    │
    ▼
Gitleaks
    │
    ▼
Docker Build
    │
    ▼
Trivy
    │
    ▼
AWS OIDC
    │
    ▼
Amazon ECR
```
**🧪 CI Stage 1 — Build & Test**

Three Java services are built using a matrix strategy:

```text
┌─────────────────┐
│  user-service   │
└────────┬────────┘
         │
┌─────────────────┐
│ product-service │
└────────┬────────┘
         │
┌─────────────────┐
│  order-service  │
└─────────────────┘
```

Each service runs:

mvn -B clean verify

Using:

Java 21
Maven

If any service fails:

```text
CI STOP
   ✕
No security stage
No Docker push
No production deployment
```
**🔎 CI Stage 2 — Gitleaks**

After successful tests, Gitleaks scans the repository for accidentally committed secrets.

```text
Build & Test
     │
     ▼
 Gitleaks
     │
 ┌───┴────┐
 │        │
FAIL     PASS
 │        │
 ▼        ▼
STOP    Continue
```
The objective is to prevent secrets such as credentials, tokens, or keys from progressing through the delivery pipeline.

**🐳 CI Stage 3 — Docker Build**

Four images are built:

devsecops-user-service
devsecops-product-service
devsecops-order-service
devsecops-frontend

Each image uses the Git commit SHA as its tag.

Example:

devsecops-user-service:<commit-sha>

This creates the relationship:

```text
Git Commit
    │
    ▼
Docker Image
    │
    ▼
ECR
    │
    ▼
EKS
    │
    ▼
Production
```
There is no dependency on a mutable latest tag.

**🛡️ CI Stage 4 — Trivy**

Each Docker image is scanned using Trivy.

The pipeline checks:

HIGH
CRITICAL

with:

--severity HIGH,CRITICAL
--exit-code 1
--ignore-unfixed

If the scan fails:

```text
Trivy FAIL
    │
    ▼
CI FAIL
    │
    X
No ECR push
No CD
```

This creates a security gate before the image becomes a production artifact.

**🔑 CI Stage 5 — GitHub OIDC → AWS IAM**

GitHub Actions does not require long-lived AWS access keys.

Instead:

```text
GitHub Actions
      │
      │ OIDC token
      ▼
GitHub OIDC Provider
      │
      │ AssumeRoleWithWebIdentity
      ▼
AWS IAM Role
      │
      ▼
AWS APIs
```
The GitHub Actions role is:

GitHubActions-DevSecOps-Production

This role provides the permissions required by the CI/CD workflows.

**📦 CI Stage 6 — Push to Amazon ECR**

After all previous stages succeed:

```text
GitHub Actions
      │
      ▼
AWS OIDC
      │
      ▼
Amazon ECR Login
      │
      ▼
Push 4 SHA-tagged images
```
Images:

devsecops-user-service:<SHA>
devsecops-product-service:<SHA>
devsecops-order-service:<SHA>
devsecops-frontend:<SHA>

At this point CI is complete.

```text
CI SUCCESS
    │
    ▼
CD automatically starts
```

**🚀 CD — Continuous Deployment**

CD Objective

The CD pipeline answers:

"How can the new version be safely introduced into production without replacing the currently serving environment until the candidate is ready?"

The answer is:

```text
Deploy new release to INACTIVE environment
                │
                ▼
          Validate completely
                │
                ▼
       Switch traffic 100%
                │
                ▼
        Verify production
                │
        ┌───────┴────────┐
        │                │
      PASS             FAIL
        │                │
        ▼                ▼
      DONE            ROLLBACK
```
**🔵🟢 Blue-Green Environment Model**

There are two production environments:

```text
                 PRODUCTION
                     │
             ┌───────┴───────┐
             │               │
             ▼               ▼
          BLUE             GREEN
     devsecops-blue    devsecops-green
```

Only one environment receives production traffic.

Example:

BLUE  = ACTIVE
GREEN = INACTIVE

or:

GREEN = ACTIVE
BLUE  = INACTIVE

The CD workflow detects this automatically.

**🧭 CD Stage 1 — Detect Active Environment**

The workflow first checks the production HTTPRoute state.

It reads:

frontend-route
user-route
product-route
order-route

For example:

frontend-route  → frontend-green
user-route      → user-service-green
product-route   → product-service-green
order-route     → order-service-green

Therefore:

ACTIVE   = GREEN
INACTIVE = BLUE

The workflow also verifies that all four routes agree.

If one route points to BLUE while the others point to GREEN:

```text
Production state inconsistent
          │
          ▼
       CD STOPS
```

This protects against deploying from an unexpected routing state.

**🆕 CD Stage 2 — Deploy to Inactive Environment**

Suppose:

GREEN = ACTIVE
BLUE  = INACTIVE

The new release is deployed to:

devsecops-blue

The workflow updates:

frontend-blue
user-service-blue
product-service-blue
order-service-blue

using the exact SHA generated by CI.

Important:

Production traffic has not changed yet.

The state is:

```text
                 Production Traffic
                       │
                       ▼
                    GREEN
                     100%

                    BLUE
                      0%
                  new release
```

**⏳ CD Stage 3 — Kubernetes Rollout Validation**

The workflow waits for every candidate deployment:

frontend-blue
user-service-blue
product-service-blue
order-service-blue

It waits using Kubernetes rollout status.

Then it validates:

Desired replicas
Updated replicas
Ready replicas
Available replicas

The candidate must be fully ready.

**🧾 CD Stage 4 — Image SHA Validation**

The workflow checks the actual image configured in each deployment.

Expected:

ECR image : <CI commit SHA>

Actual:

Kubernetes deployment image

If they do not match:

```text
Candidate validation FAIL
        │
        ▼
Production traffic remains unchanged
```

This prevents an unexpected image from becoming active.

**✅ Candidate Ready**

Only after all candidate checks pass:

```text
┌────────────────────────────────────┐
│       PRODUCTION CANDIDATE READY   │
├────────────────────────────────────┤
│                                    │
│ Current Active : GREEN             │
│ New Candidate  : BLUE              │
│ Version        : <commit-sha>      │
│                                    │
│ Production traffic: UNCHANGED      │
│                                    │
└────────────────────────────────────┘
```

Now the traffic switch can occur.

**🔀 CD Stage 5 — Where Does the Traffic Switch Actually Happen?**

This is the most important part of the architecture.

The application traffic path is:

```text
User
  │
  ▼
Route 53
  │
  ▼
AWS Application Load Balancer
  │
  ▼
AWS Load Balancer Controller
  │
  ▼
Kubernetes Gateway
  │
  ▼
HTTPRoute
  │
  ▼
Kubernetes Service
  │
  ▼
Target Group
  │
  ▼
Pod IP
  │
  ▼
Application Container
  │
  ▼
Response
```

The deployment workflow does not directly modify Route 53 or manually move ALB target groups.

The CD workflow changes the Kubernetes HTTPRoute.

The Kubernetes Gateway API configuration is the desired routing state.

The AWS Load Balancer Controller observes the Kubernetes Gateway/HTTPRoute resources and reconciles the AWS load balancer configuration.

🌐 Complete End-to-End User Request Flow

Step 1 — User Request

A user accesses the application:

https://your-domain.example.com

The request enters:

```text
User
  │
  ▼
Internet
```

Step 2 — Route 53

Route 53 provides DNS resolution.

Conceptually:

```text
your-domain.example.com
          │
          ▼
       Route 53
          │
          ▼
ALB DNS endpoint
```
Route 53's responsibility is DNS resolution.

It does not perform the blue-green application deployment switch in this architecture.

⚖️ Step 3 — Application Load Balancer

The request reaches the AWS Application Load Balancer.

```text
Route 53
    │
    ▼
AWS ALB
```

The ALB is the external load-balancing entry point.

The AWS Load Balancer Controller manages the AWS load-balancer configuration based on Kubernetes resources.

🎛️ Step 4 — AWS Load Balancer Controller

Inside the EKS cluster:

```text
Kubernetes Resources
        │
        ▼
AWS Load Balancer Controller
        │
        ▼
AWS Load Balancer configuration
```
The controller watches Kubernetes networking resources and reconciles the corresponding AWS infrastructure.

This creates the bridge between:

Kubernetes Gateway API

and:

AWS Application Load Balancer

🚪 Step 5 — Gateway

The Kubernetes Gateway represents the entry point for application traffic inside the Kubernetes networking model.

Conceptually:

```text
ALB
 │
 ▼
Gateway
 │
 ▼
HTTPRoute
```
The Gateway defines the listener/entry point.

The HTTPRoute defines how application requests are routed.

🛣️ Step 6 — HTTPRoute

This is where the blue-green application traffic decision is expressed.

Example:

```yaml
backendRefs:
  - name: frontend-blue
    namespace: devsecops-blue
    port: 80
    weight: 100
```

This means the route sends:

```text
frontend traffic
       │
       ▼
frontend-blue
       │
      100%
```

After the blue-green switch:

```yaml
backendRefs:
  - name: frontend-green
    namespace: devsecops-green
    port: 80
    weight: 100
```

Now:

```text
frontend traffic
       │
       ▼
frontend-green
       │
      100%
```
🎯 The Exact Traffic Switch Point

The logical application traffic switch in this implementation happens at the Kubernetes HTTPRoute backend reference.

Before:

```text
HTTPRoute
   │
   ▼
frontend-blue
   │
 100%
```

After:

```text
HTTPRoute
   │
   ▼
frontend-green
   │
 100%
```
The CD workflow performs this by patching the HTTPRoute.

For all four application routes:

frontend-route
user-route
product-route
order-route

the workflow replaces the backend reference with the new environment and sets:

weight = 100

So the production transition is:

OLD ENVIRONMENT
      │
      │ 100%
      ▼
    BLUE

        SWITCH

    GREEN
      │
      │ 100%
      ▼
NEW ENVIRONMENT

🎯 What About Target Groups?

The important distinction is:

The GitHub Actions CD workflow does not directly switch AWS Target Groups.

The CD workflow changes:

Kubernetes HTTPRoute

Then the networking controller reconciles the desired Kubernetes state into the AWS load-balancer configuration.

Conceptually:

```text
GitHub Actions
      │
      │ kubectl patch HTTPRoute
      ▼
Kubernetes HTTPRoute
      │
      ▼
Gateway API state
      │
      ▼
AWS Load Balancer Controller
      │
      ▼
AWS ALB configuration
      │
      ▼
Target Groups / routing
```
The exact AWS target-group structure depends on the Gateway implementation and controller configuration. Therefore, the safe architectural statement is:

HTTPRoute is the Kubernetes-level routing control; AWS Load Balancer Controller translates/reconciles the Kubernetes networking state into the AWS load-balancer configuration, including the required backend target configuration.

🎯 How Do Target Groups Reach Pod IPs?

At the backend, Kubernetes provides the service abstraction.

Conceptually:

```text
HTTPRoute
    │
    ▼
Kubernetes Service
    │
    ▼
Service endpoints
    │
    ▼
Pod IPs
```

With AWS load balancing, the AWS load balancer can use pod IPs as targets when configured with the appropriate target type.

The conceptual relationship is:

```text
Kubernetes Service
       │
       ▼
EndpointSlice
       │
       ▼
Pod IP
       │
       ▼
AWS Target Group
       │
       ▼
ALB
```

For example:

```text
frontend-green Service
        │
        ├── Pod IP 10.x.x.21
        ├── Pod IP 10.x.x.22
        └── Pod IP 10.x.x.23
```
The target registration/reconciliation is handled by Kubernetes/AWS integration components rather than by the GitHub Actions deployment script itself.

🔄 Full Request-to-Response Flow

The complete production request path can be visualized as:

                         USER
                          │
                          │ HTTPS Request
                          ▼
                     ┌──────────┐
                     │ Route 53 │
                     └────┬─────┘
                          │
                          │ DNS
                          ▼
              ┌──────────────────────┐
              │ Application Load     │
              │ Balancer (AWS ALB)   │
              └──────────┬───────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │ AWS Load Balancer    │
              │ Controller           │
              │                      │
              │ Reconciles AWS with  │
              │ Kubernetes Gateway   │
              │ configuration        │
              └──────────┬───────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │ Kubernetes Gateway   │
              └──────────┬───────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │ HTTPRoute            │
              │                      │
              │ BLUE 100%             │
              │       OR              │
              │ GREEN 100%            │
              └──────────┬───────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │ Kubernetes Service   │
              └──────────┬───────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │ EndpointSlice /      │
              │ Pod endpoints        │
              └──────────┬───────────┘
                         │
                         ▼
                  ┌──────────────┐
                  │   Pod IP     │
                  │              │
                  │ Application  │
                  └──────┬───────┘
                         │
                         │ Response
                         ▼
                     ALB → User

🔁 What Changes During a Blue-Green Deployment?

Suppose the current production environment is GREEN.

Before deployment:

```text
                     HTTPRoute
                         │
                         ▼
                 GREEN Service
                         │
                     100% traffic
                         │
                         ▼
                    GREEN Pods
```

BLUE is running separately but receives no production traffic.

```text
BLUE Pods
   │
   └── 0% production traffic
```

During deployment:

The new release is deployed to BLUE.

```text
                    PRODUCTION
                        │
                        ▼
                     GREEN
                      100%
                        │
                        ▼
                   GREEN Pods


                     BLUE
                       │
                       │ new version
                       ▼
                  BLUE Pods
```

Production traffic remains on GREEN.

After candidate validation:

CD patches the HTTPRoutes.

```text
BEFORE:

HTTPRoute
   │
   └── GREEN = 100%

AFTER:

HTTPRoute
   │
   └── BLUE = 100%
```

Now:

```text
GREEN = 0%
BLUE  = 100%
```
🚨 Automatic Rollback

The deployment contains a rollback path for failed final production verification.

Example:

```text
Before:

GREEN = 100%
BLUE  = 0%

New version:

BLUE candidate = healthy

Traffic switch:

GREEN = 0%
BLUE  = 100%

Final production verification fails:

ERROR
  │
  ▼
Rollback
  │
  ▼
HTTPRoutes restored
  │
  ▼
GREEN = 100%
BLUE  = 0%
```

🧯 Rollback Flow

```text
                 100% SWITCH
                      │
                      ▼
             New environment
                becomes active
                      │
                      ▼
             Final verification
                      │
               ┌──────┴──────┐
               │             │
             PASS          FAIL
               │             │
               ▼             ▼
             DONE         ROLLBACK
                              │
                              ▼
                    Restore old HTTPRoutes
                              │
                              ▼
                       OLD ENV = 100%
                              │
                              ▼
                         Production
```
The previous environment remains available because blue and green are maintained separately.

🔐 Why Keep the Old Environment Running?

The inactive environment is not immediately destroyed after a release.

This is important because:

```text
OLD ENVIRONMENT
       │
       └── remains available
```

If the new environment has a problem:

```text
Production
    │
    ▼
New environment
    │
    X
    │
    ▼
Old environment
   100%
```
This allows a fast traffic rollback without rebuilding the old release.

🔒 Production Safety Controls

The CD workflow includes several safeguards.

1. CI must succeed

```text
CI FAIL
  │
  X
No production deployment
```

2. Main push only

CD only continues for a successful CI run caused by a push to main.

3. Active environment detection

The workflow determines:

```text
ACTIVE
INACTIVE
```

before deploying.

4. Route consistency validation

All four application routes must agree on the active environment.

5. Candidate rollout validation

All candidate deployments must become ready.

6. Image SHA validation

The candidate must use the exact image generated by CI.

7. Traffic verification

After the switch:

```text
Expected environment = new environment
Expected traffic      = 100%
```

8. Production verification

The active deployments must:

```text
run expected image
have expected replicas ready
```

9. Automatic rollback

If final verification fails after the switch, the previous environment is restored.

10. Deployment concurrency protection

The workflow uses:

```yaml
concurrency:
  group: production-blue-green
  cancel-in-progress: false
```
This prevents multiple production blue-green workflows from modifying production traffic state simultaneously.

📊 Current Traffic Model

The current project uses:

```text
                 BLUE-GREEN
                     │
          ┌──────────┴──────────┐
          │                     │
       ACTIVE                INACTIVE
          │                     │
        100%                    0%
```

After deployment:

```text
                 BLUE-GREEN
                     │
          ┌──────────┴──────────┐
          │                     │
       OLD ENV              NEW ENV
          │                     │
         0%                    100%
```
This is not a gradual canary strategy.

There is no:

90/10
75/25
50/50
25/75

traffic progression in the current CD implementation.

## 🧠 Blue-Green vs Canary in This Project

| Feature | Current Implementation |
| --- | --- |
| Two production environments | ✅ |
| Blue environment | ✅ |
| Green environment | ✅ |
| New release deployed before switch | ✅ |
| Direct 100% switch | ✅ |
| Gradual traffic increase | ❌ |
| Canary percentages | ❌ |
| Candidate validation before switch | ✅ |
| Automatic rollback | ✅ |

The architecture can be extended to weighted/canary traffic later, but the current implementation intentionally keeps production traffic switching simple:

```text
100% OLD
   ↓
100% NEW
```

## 🗂️ Kubernetes Traffic Model

The production namespace contains the HTTPRoutes:

```text
devsecops
│
├── frontend-route
├── user-route
├── product-route
└── order-route
```

They point to services in the blue or green namespaces.

Example:

```text
frontend-route
      │
      ▼
frontend-green
      │
      ▼
devsecops-green
      │
      ▼
GREEN frontend pods
```
The same model applies to:

user-service
product-service
order-service

🧬 Release Traceability

Every release can be traced through the complete chain:

```text
Git Commit SHA
      │
      ▼
GitHub Actions CI
      │
      ▼
Docker Image SHA Tag
      │
      ▼
Amazon ECR
      │
      ▼
GitHub Actions CD
      │
      ▼
Inactive EKS Environment
      │
      ▼
HTTPRoute
      │
      ▼
Production
```

For example:

```text
commit abc123
     │
     ├── user-service:abc123
     ├── product-service:abc123
     ├── order-service:abc123
     └── frontend:abc123
              │
              ▼
       ECR
              │
              ▼
       EKS BLUE/GREEN
```
This makes production releases auditable and reproducible.

## 🔐 Security Architecture

```text
Developer
    │
    ▼
GitHub
    │
    ▼
Gitleaks
    │
    ▼
Docker Build
    │
    ▼
Trivy
    │
    ▼
GitHub OIDC
    │
    ▼
AWS IAM
    │
    ▼
Amazon ECR
    │
    ▼
Amazon EKS
```
Security is integrated directly into the CI pipeline instead of being treated as a separate post-deployment activity.

## ☁️ AWS Architecture

```text
AWS
│
├── ap-south-1
│
├── IAM
│   └── GitHubActions-DevSecOps-Production
│
├── Amazon ECR
│   ├── devsecops-user-service
│   ├── devsecops-product-service
│   ├── devsecops-order-service
│   └── devsecops-frontend
│
├── Amazon EKS
│   └── devsecops-eks
│       │
│       ├── Gateway
│       ├── HTTPRoutes
│       │
│       ├── devsecops-blue
│       │   ├── frontend
│       │   ├── user-service
│       │   ├── product-service
│       │   └── order-service
│       │
│       └── devsecops-green
│           ├── frontend
│           ├── user-service
│           ├── product-service
│           └── order-service
│
└── Application Load Balancer
```
## 📁 Repository Structure

```text
devsecops-production-platform/
│
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy-blue-green.yml
│
├── frontend/
│
├── user-service/
│
├── product-service/
│
├── order-service/
│
├── k8s/
│
├── mysql/
│
├── docker-compose.yml
├── eksctl-config.yaml
├── iam_policy.json
└── README.md
```

## ⚙️ CI Workflow

`.github/workflows/ci.yml`

Responsibilities:

- ✓ Build Java services
- ✓ Run tests
- ✓ Detect secrets with Gitleaks
- ✓ Build Docker images
- ✓ Scan images with Trivy
- ✓ Authenticate to AWS using OIDC
- ✓ Push images to ECR

## ⚙️ CD Workflow

`.github/workflows/deploy-blue-green.yml`

Responsibilities:

- ✓ Wait for successful CI
- ✓ Confirm main branch push
- ✓ Detect active environment
- ✓ Identify inactive environment
- ✓ Deploy release to inactive environment
- ✓ Wait for rollout
- ✓ Validate replicas
- ✓ Validate image SHA
- ✓ Switch HTTPRoutes to new environment
- ✓ Set production traffic to 100%
- ✓ Verify production
- ✓ Roll back to previous environment when final verification fails

## 🔬 End-to-End Deployment Example

Assume the current state is:

```text
GREEN = ACTIVE
BLUE  = INACTIVE
```

A developer runs:

```bash
git push origin main
```

### CI

```text
main
 │
 ▼
Build & Test
 │
 ▼
Gitleaks
 │
 ▼
Docker Build
 │
 ▼
Trivy
 │
 ▼
OIDC → AWS
 │
 ▼
ECR Push
 │
 ▼
CI SUCCESS
```

### CD

```text
Detect:

ACTIVE   = GREEN
INACTIVE = BLUE

Deploy:

new SHA → BLUE

Validate:

BLUE replicas      = READY
BLUE image         = expected SHA
BLUE deployments   = healthy

Traffic before switch:

GREEN = 100%
BLUE  = 0%

Switch:

HTTPRoute
   │
   ▼
BLUE = 100%

Traffic after switch:

GREEN = 0%
BLUE  = 100%

Final verification:

BLUE image SHA correct
BLUE replicas ready
HTTPRoutes → BLUE
Traffic → 100%

Release complete.
```

## 🔁 Next Deployment

On the next release:

```text
BLUE  = ACTIVE
GREEN = INACTIVE
```

The new release is deployed to GREEN:

```text
new SHA → GREEN
```

After validation:

```text
BLUE  = 0%
GREEN = 100%
```

Therefore the deployment cycle continuously alternates:

```text
Release 1
GREEN → BLUE

Release 2
BLUE → GREEN

Release 3
GREEN → BLUE

Release 4
BLUE → GREEN
```
## 🏁 Final Architecture Summary

The complete production delivery chain is:

```text
                         SOURCE
                           │
                           ▼
                    GitHub main branch
                           │
                           ▼
                    GitHub Actions CI
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
      Build/Test       Gitleaks          Docker
                                           │
                                           ▼
                                         Trivy
                                           │
                                           ▼
                                    GitHub OIDC → AWS
                                           │
                                           ▼
                                      Amazon ECR
                                           │
                                           ▼
                              GitHub Actions CD
                                           │
                              ┌────────────┴────────────┐
                              │                         │
                              ▼                         ▼
                       ACTIVE ENV                 INACTIVE ENV
                              │                         │
                              │                    New release
                              │                         │
                              │                    Rollout check
                              │                         │
                              │                    Image check
                              │                         │
                              └────────────┬────────────┘
                                           │
                                      Candidate Ready
                                           │
                                           ▼
                                  HTTPRoute PATCH
                                           │
                                           ▼
                                    100% traffic
                                           │
                                           ▼
                              AWS LB Controller
                                           │
                                           ▼
                                  AWS ALB / Targets
                                           │
                                           ▼
                                  Kubernetes Service
                                           │
                                           ▼
                                      Pod IPs
                                           │
                                           ▼
                                    Application
                                           │
                                           ▼
                                      RESPONSE
```

## 🎯 Core Principle

```text
┌──────────────────────────────────────────────────────────┐
│                                                          │
│   BUILD → TEST → SCAN → PACKAGE → DEPLOY → VALIDATE     │
│                                                          │
│                     ↓                                    │
│                                                          │
│              SWITCH TRAFFIC 100%                         │
│                                                          │
│                     ↓                                    │
│                                                          │
│                VERIFY PRODUCTION                         │
│                                                          │
│                     ↓                                    │
│                                                          │
│             FAILURE → AUTOMATIC ROLLBACK                 │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

This project demonstrates a complete DevSecOps production workflow where security, immutable artifacts, Kubernetes deployment validation, blue-green environments, controlled traffic switching, production verification, and rollback are integrated into a single automated delivery process.

## 👨‍💻 Project Focus

The platform demonstrates practical implementation of:

- DevSecOps CI/CD
- GitHub Actions
- AWS EKS
- Amazon ECR
- GitHub OIDC
- Docker
- Maven / Java 21
- Gitleaks
- Trivy
- Kubernetes Gateway API
- Gateway / HTTPRoute
- AWS Load Balancer Controller

AWS Load Balancer Controller

Application Load Balancer

Blue-Green Deployment

Immutable SHA-based Releases

100% Production Traffic Switching

Production Validation

Automatic Rollback

End-to-End Request Routing

Deployment strategy: Blue-Green with direct 100% traffic switching.
Canary/gradual traffic shifting is intentionally not part of the current implementation.

==========================================================================================




# AWS ALB Controller + Gateway API + Blue-Green Traffic Switching

## Overview

This document explains how **AWS Load Balancer Controller**, **Kubernetes Gateway API**, **GatewayClass**, **Gateway**, **HTTPRoute**, **TargetGroupConfiguration (TGC)**, **TargetGroupBinding (TGB)**, Kubernetes Services, EndpointSlices, AWS ALB Target Groups, and Pod IPs work together in a blue-green deployment.

It also explains exactly what happens when the **GitHub Actions CD pipeline switches production traffic from Blue to Green**.

---

# 1. Complete Production Traffic Flow

The complete request path is:

```text
                         INTERNET
                            |
                            v
                    +----------------+
                    |    Route 53    |
                    | DNS / Domain   |
                    +-------+--------+
                            |
                            | DNS
                            v
                 +-----------------------+
                 |     AWS ALB           |
                 | Application LB        |
                 +----------+------------+
                            |
                            | Listener :80/:443
                            v
                 +-----------------------+
                 | ALB Listener Rules    |
                 | created/managed by    |
                 | AWS Load Balancer     |
                 | Controller            |
                 +----------+------------+
                            |
                            | HTTP host/path
                            v
                 +-----------------------+
                 |      HTTPRoute        |
                 | Kubernetes Gateway API |
                 +----------+------------+
                            |
                    backendRefs
                            |
                +-----------+-----------+
                |                       |
                v                       v
        user-service-blue       user-service-green
        Service                 Service
                |                       |
                v                       v
        Target Group Blue       Target Group Green
                |                       |
                v                       v
          Pod IPs Blue           Pod IPs Green
```

The key point is that **HTTPRoute is the Kubernetes traffic-routing intent**, while the **AWS Load Balancer Controller converts that intent into AWS ALB configuration**.

The controller continuously reconciles Kubernetes Gateway API objects with AWS resources.

---

# 2. The Components and Their Responsibilities

## 2.1 Route 53

Route 53 is responsible for DNS.

For example:

```text
api.example.com
      |
      v
Route 53
      |
      v
ALB DNS name
```

Route 53 does **not** know about:

* Kubernetes Pods
* Services
* HTTPRoutes
* TargetGroups
* Blue/Green namespaces

It simply resolves:

```text
api.example.com
        ↓
<ALB DNS name>
```

After DNS resolution, the request goes to the ALB.

Therefore:

> **Blue/Green switching does NOT normally happen in Route 53.**

The DNS record continues pointing to the same ALB.

The traffic switch happens **inside the ALB routing configuration**, driven by the Kubernetes `HTTPRoute`.

---

# 3. AWS Application Load Balancer

The AWS ALB is the Layer 7 entry point.

It understands:

* HTTP
* HTTPS
* Host headers
* Paths
* Listener rules
* Target Groups
* Health checks

For example:

```text
https://api.example.com/users

             |
             v

             ALB
              |
              v
       HTTPS Listener :443
              |
              v
       Listener Rule
       Host = api.example.com
       Path = /users
              |
              v
       Target Group
              |
              v
        Pod IP:Port
```

With Gateway API, you don't manually create those listener rules every time.

The AWS Load Balancer Controller derives them from Kubernetes Gateway API resources.

AWS documents that L7 `HTTPRoute` resources are implemented using an ALB.

---

# 4. AWS Load Balancer Controller

The **AWS Load Balancer Controller (ALBC)** is the reconciliation engine connecting:

```text
Kubernetes
    |
    | Desired state
    v
AWS Load Balancer Controller
    |
    | AWS APIs
    v
AWS
```

It watches Kubernetes resources such as:

```text
GatewayClass
Gateway
HTTPRoute
Service
EndpointSlice
TargetGroupConfiguration
TargetGroupBinding
```

and reconciles AWS resources such as:

```text
ALB
Listeners
Listener Rules
Target Groups
Target Registrations
Security Groups
```

The controller runs continuously.

It is not a one-time deployment script.

Conceptually:

```text
Kubernetes Desired State
          |
          v
+-------------------------+
| AWS Load Balancer       |
| Controller              |
|                         |
| Observe                 |
| Compare                 |
| Reconcile               |
+------------+------------+
             |
             v
      AWS Actual State
```

If Kubernetes says:

```text
HTTPRoute → green
```

but ALB currently routes to:

```text
blue
```

the controller detects the difference and updates the ALB.

---

# 5. Gateway API

Gateway API is the Kubernetes-native API model for expressing traffic management.

Instead of putting everything into one large Ingress object, responsibilities are separated.

The important resources are:

```text
GatewayClass
      |
      v
Gateway
      |
      v
HTTPRoute
      |
      v
Service
      |
      v
Pods
```

This separation is one of the biggest advantages of Gateway API.

---

# 6. GatewayClass

`GatewayClass` defines:

> **Which controller is responsible for managing this Gateway?**

Example:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: alb
spec:
  controllerName: gateway.k8s.aws/alb
```

The important field is:

```yaml
controllerName: gateway.k8s.aws/alb
```

This tells the AWS Load Balancer Controller:

```text
"This GatewayClass belongs to me."
```

Therefore:

```text
GatewayClass
     |
     | controllerName
     v
AWS Load Balancer Controller
```

For ALB Gateway API support, the AWS controller manages `GatewayClass` objects using the `gateway.k8s.aws/alb` controller name.

---

# 7. Gateway

A `Gateway` represents the actual traffic entry point.

Example:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: devsecops-gateway
  namespace: devsecops
spec:
  gatewayClassName: alb

  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      hostname: "*.example.com"
```

Conceptually:

```text
GatewayClass
     |
     | "Use ALB controller"
     v
Gateway
     |
     | "Create/manage this ALB"
     v
AWS ALB
```

The Gateway describes things such as:

* which GatewayClass to use
* listeners
* ports
* protocols
* hostnames
* TLS configuration
* which routes may attach

The controller turns this desired state into an AWS ALB.

---

# 8. HTTPRoute

`HTTPRoute` is where your application traffic-routing rules are defined.

For example:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: user-route
  namespace: devsecops
spec:
  parentRefs:
    - name: devsecops-gateway

  hostnames:
    - "api.example.com"

  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /users

      backendRefs:
        - name: user-service-green
          port: 8080
```

This means:

```text
Request:

https://api.example.com/users
             |
             v
        HTTPRoute
             |
             v
user-service-green:8080
```

The `HTTPRoute` API defines `parentRefs`, hostnames, matches, filters and backend references.

---

# 9. HTTPRoute Is the Traffic-Switching Point

This is the most important concept for your blue-green deployment.

Suppose production currently has:

```text
HTTPRoute
    |
    v
user-service-blue
```

and you deploy Green.

The CD workflow changes the HTTPRoute:

```text
BEFORE

HTTPRoute
    |
    v
user-service-blue
```

After the traffic switch:

```text
AFTER

HTTPRoute
    |
    v
user-service-green
```

The GitHub Actions runner does not directly execute:

```text
Modify ALB
Modify Target Group
Register Pod IP
```

Instead, it does something conceptually like:

```text
kubectl apply / kubectl patch HTTPRoute
                  |
                  v
           Kubernetes API
                  |
                  v
          HTTPRoute changed
                  |
                  v
       AWS Load Balancer Controller
                  |
                  v
           AWS API calls
                  |
                  v
              ALB changed
```

---

# 10. Kubernetes Service

The HTTPRoute normally points to a Kubernetes Service.

Example:

```yaml
backendRefs:
  - name: user-service-green
    port: 8080
```

The Service provides a stable Kubernetes abstraction over changing Pods.

For example:

```text
user-service-green
        |
        +----------------+
        |                |
        v                v
Pod Green #1       Pod Green #2
10.0.3.21:8080     10.0.4.17:8080
```

The Service itself does not become an AWS Target Group.

The AWS Load Balancer Controller uses the Service/backend relationship to build the AWS target-group configuration.

---

# 11. EndpointSlices

This is an important piece people often forget.

Kubernetes maintains EndpointSlices containing the current endpoints behind a Service.

Example:

```text
Service:
user-service-green

EndpointSlice:

10.0.3.21:8080
10.0.4.17:8080
10.0.5.32:8080
```

If a Pod is deleted:

```text
10.0.3.21
```

the EndpointSlice changes.

If a new Pod appears:

```text
10.0.8.44
```

the EndpointSlice changes.

The controller watches these changes and reconciles AWS target registration.

So the chain is:

```text
Deployment
    |
    v
Pods
    |
    v
EndpointSlice
    |
    v
AWS Load Balancer Controller
    |
    v
AWS Target Group
```

---

# 12. Target Group

An AWS Target Group is the collection of backend targets that receive traffic from the ALB.

For IP mode:

```text
Target Group Green

Targets:

10.0.3.21:8080
10.0.4.17:8080
10.0.5.32:8080
```

These are Pod IPs.

The ALB sends traffic directly to those Pod IPs.

AWS documents that IP mode registers Pods directly as ALB targets, while instance mode registers Kubernetes nodes and forwards through NodePort.

For your architecture, the important model is:

```text
ALB
 |
 v
Target Group
 |
 +---- Pod IP 10.0.3.21
 |
 +---- Pod IP 10.0.4.17
 |
 +---- Pod IP 10.0.5.32
```

---

# 13. Why IP Target Type Matters

With:

```text
targetType = ip
```

the ALB directly targets Pods.

Traffic:

```text
ALB
 |
 v
Pod IP
```

Instead of:

```text
ALB
 |
 v
Node
 |
 v
NodePort
 |
 v
Pod
```

For EKS VPC networking, Pod IPs are directly reachable by the ALB when the networking prerequisites are satisfied.

---

# 14. TargetGroupConfiguration — TGC

`TargetGroupConfiguration` is an AWS Load Balancer Controller CRD used to customize Target Group behavior.

It does **not mean "create the target group manually."**

Instead, it tells the controller:

> "When you create/manage the Target Group for this backend, use these properties."

For example:

```yaml
apiVersion: gateway.k8s.aws/v1
kind: TargetGroupConfiguration
metadata:
  name: user-service-tgc
  namespace: devsecops-green

spec:
  targetReference:
    name: user-service-green

  defaultConfiguration:
    targetType: ip

    healthCheckConfig:
      healthCheckPath: /health

    targetGroupAttributes:
      - key: deregistration_delay.timeout_seconds
        value: "30"
```

This can control things such as:

```text
Target type
Health checks
Target Group attributes
Tags
Other target-group properties
```

AWS documents three levels of TGC customization:

```text
GatewayClass defaults
        |
        v
Gateway defaults
        |
        v
Service-specific configuration
```

and route-specific overrides can also be defined.

---

# 15. TargetGroupBinding — TGB

This is another important concept.

`TargetGroupBinding` represents the relationship between:

```text
Kubernetes Service
        +
AWS Target Group
```

Example:

```yaml
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: user-service-green
spec:
  serviceRef:
    name: user-service-green
    port: 8080

  targetGroupARN:
    arn:aws:elasticloadbalancing:...
```

Conceptually:

```text
AWS Target Group
       |
       | TargetGroupBinding
       |
       v
Kubernetes Service
       |
       v
EndpointSlices
       |
       v
Pod IPs
```

But here is the important detail:

> **You normally do NOT create TGB manually for ordinary Gateway API traffic.**

The AWS Load Balancer Controller internally uses `TargetGroupBinding` for Ingress, Service, and Gateway resources and automatically creates the relevant TGB in the Service namespace.

Therefore your architecture may contain TGB objects even though your CD workflow never creates them.

---

# 16. TGC vs TGB

These two are easy to confuse.

| Resource                   | Main purpose                                     |
| -------------------------- | ------------------------------------------------ |
| `TargetGroupConfiguration` | Configure how the AWS Target Group should behave |
| `TargetGroupBinding`       | Bind an AWS Target Group to a Kubernetes Service |
| `Service`                  | Stable Kubernetes backend abstraction            |
| `EndpointSlice`            | Current Pod endpoints                            |
| `HTTPRoute`                | HTTP traffic routing decision                    |
| `Gateway`                  | Traffic entry point / ALB                        |
| `GatewayClass`             | Selects the controller                           |

Think of it this way:

```text
HTTPRoute
    |
    | "Send traffic to"
    v
Service
    |
    | "These are the current Pods"
    v
EndpointSlice
    |
    v
TGB
    |
    | "This AWS TG represents this backend"
    v
AWS Target Group
    |
    v
Pod IPs
```

While TGC is configuration applied to the Target Group:

```text
TGC
 |
 +---- targetType = ip
 |
 +---- health check = /health
 |
 +---- deregistration delay
 |
 +---- target group attributes
```

---

# 17. How Pod IPs Enter the AWS Target Group

This is the part you were specifically asking about.

Suppose Green has:

```text
Deployment:

user-service-green

Pods:

Pod 1 → 10.0.3.21
Pod 2 → 10.0.4.17
Pod 3 → 10.0.5.32
```

Kubernetes creates/updates EndpointSlices:

```text
user-service-green
        |
        v
EndpointSlice
        |
        +---- 10.0.3.21:8080
        +---- 10.0.4.17:8080
        +---- 10.0.5.32:8080
```

The AWS Load Balancer Controller reconciles the target group:

```text
Target Group Green

Registered Targets:

10.0.3.21:8080
10.0.4.17:8080
10.0.5.32:8080
```

If Pod 2 disappears:

```text
Pod 2
10.0.4.17
     X
```

EndpointSlice changes:

```text
10.0.3.21
10.0.5.32
```

Controller reconciles:

```text
Target Group

REMOVE:
10.0.4.17

KEEP:
10.0.3.21
10.0.5.32
```

If Kubernetes creates a new Pod:

```text
10.0.8.44
```

the controller eventually registers:

```text
10.0.8.44
```

Therefore:

> **Pod IP registration is continuously reconciled. It is not a one-time operation performed during deployment.**

---

# 18. Blue-Green Architecture

Assume:

```text
Namespace: devsecops-blue

user-service-blue
frontend-blue
product-service-blue
order-service-blue
```

and:

```text
Namespace: devsecops-green

user-service-green
frontend-green
product-service-green
order-service-green
```

Both environments can exist simultaneously.

```text
                         ALB
                          |
                    HTTPRoute
                          |
              +-----------+-----------+
              |                       |
              v                       v
          BLUE TG                 GREEN TG
              |                       |
        +-----+-----+           +-----+-----+
        |     |     |           |     |     |
       Pod   Pod   Pod          Pod   Pod   Pod
```

Initially:

```text
HTTPRoute
     |
     v
BLUE
```

Green is deployed but receives no production traffic.

---

# 19. Initial State — Blue Is Active

```text
                    Route 53
                        |
                        v
                       ALB
                        |
                        v
                 HTTPS Listener
                        |
                        v
                 HTTPRoute
                        |
                        v
               user-service-blue
                        |
                        v
                  Target Group
                     BLUE
                        |
              +---------+---------+
              |         |         |
              v         v         v
           Pod Blue  Pod Blue  Pod Blue
```

Green exists separately:

```text
user-service-green
        |
        v
Green Target Group
        |
        v
Green Pods
```

but production HTTPRoute is not pointing to Green.

---

# 20. GitHub Actions CD Deployment

Your CD workflow first deploys Green.

Conceptually:

```text
GitHub Actions
      |
      v
Deploy Green manifests
      |
      v
Kubernetes
      |
      v
Green Pods created
      |
      v
Services / EndpointSlices
      |
      v
AWS Load Balancer Controller
      |
      v
Green Target Group
      |
      v
Green Pod IPs registered
```

At this point:

```text
BLUE = ACTIVE
GREEN = INACTIVE
```

but both are healthy.

---

# 21. Green Validation

Before switching production traffic, CD should verify:

```text
Green Deployment
       |
       v
Pods Running
       |
       v
Readiness checks
       |
       v
Service endpoints
       |
       v
ALB Target Group
       |
       v
Targets healthy
       |
       v
Application smoke tests
```

Example:

```text
Green Pods
   |
   +---- Ready
   |
   +---- Health check OK
   |
   +---- ALB target healthy
   |
   +---- Application test successful
```

Only after validation should the traffic switch occur.

---

# 22. The Actual Blue → Green Traffic Switch

This is the critical part.

Before:

```yaml
backendRefs:
  - name: user-service-blue
    port: 8080
```

GitHub Actions changes the Kubernetes HTTPRoute to:

```yaml
backendRefs:
  - name: user-service-green
    port: 8080
```

The change is:

```text
GitHub Actions
      |
      v
kubectl patch/apply HTTPRoute
      |
      v
Kubernetes API Server
      |
      v
HTTPRoute updated
      |
      v
AWS Load Balancer Controller
      |
      v
Reconciliation
      |
      v
AWS ALB Listener Rule updated
      |
      v
Traffic now forwarded to
GREEN Target Group
```

The ALB itself is not necessarily recreated.

The important change is the routing configuration associated with the existing ALB.

---

# 23. What Happens Inside the Controller

Conceptually:

```text
HTTPRoute changed

backendRefs:
    BLUE
      ↓
    GREEN

        |
        v

AWS Load Balancer Controller
        |
        +-----------------------------+
        |                             |
        v                             v
Find Gateway                   Find backend Service
        |                             |
        v                             v
Find ALB                       Find EndpointSlices
        |                             |
        +-------------+---------------+
                      |
                      v
              Build desired state
                      |
                      v
             Compare AWS state
                      |
                      v
              AWS API operations
```

The controller then makes AWS state match the Kubernetes desired state.

---

# 24. Does the Controller Change Pod IPs?

**No.**

This distinction is extremely important.

The controller does **not** assign Pod IPs.

Pod IP lifecycle belongs to:

```text
EKS
Kubernetes
VPC CNI
Scheduler
CNI/IP allocation
```

The controller discovers/reconciles which Pod IPs should be registered as ALB targets.

Therefore:

```text
Who creates Pod?

Kubernetes Deployment / ReplicaSet
        |
        v
Pod

Who assigns Pod IP?

EKS networking / VPC CNI

Who exposes Pod through Service?

Kubernetes Service

Who maintains endpoint list?

Kubernetes EndpointSlice

Who registers targets into AWS Target Group?

AWS Load Balancer Controller
```

That is the clean separation of responsibilities.

---

# 25. Does the Controller Change the Target Group?

**It can create, configure, update, and reconcile Target Groups as required by the desired Gateway/route/backend state.**

But don't think:

```text
HTTPRoute changed
      =
Target Group destroyed
      =
new Target Group created
```

That is not the correct general mental model.

A better model is:

```text
Kubernetes desired state
        |
        v
Controller determines
required AWS resources
        |
        v
Create/update/reuse resources
        |
        v
Reconcile target registrations
```

Depending on the resource graph and changes, the controller may reuse existing AWS resources or create/update resources.

---

# 26. Blue-Green Traffic Switch — Full Flow

Here is the complete flow:

```text
                    GitHub Actions
                         |
                         |
                  CD deployment
                         |
                         v
               Deploy GREEN version
                         |
                         v
                 Kubernetes API
                         |
                         v
                  Green Deployment
                         |
                         v
                    Green Pods
                         |
                         v
                   Pod IPs created
                         |
                         v
                    Service
                         |
                         v
                  EndpointSlice
                         |
                         v
             AWS Load Balancer Controller
                         |
                         v
                 Green Target Group
                         |
                         v
                Green targets healthy
                         |
                         v
              Smoke / health validation
                         |
                         v
              Update HTTPRoute
              BLUE → GREEN
                         |
                         v
             Kubernetes API Server
                         |
                         v
             AWS Load Balancer Controller
                         |
                         v
                  Reconciliation
                         |
                         v
                 AWS ALB Listener
                         |
                         v
               Green Target Group
                         |
                         v
                    Green Pods
```

---

# 27. What Does NOT Change During the Switch

This is just as important.

Normally the following remain unchanged:

```text
Route 53
   |
   v
ALB DNS name
```

The ALB itself can remain the same.

The Gateway can remain the same.

The listeners can remain the same.

The production hostname remains the same.

For example:

```text
api.example.com
```

still resolves to the same ALB.

The switch occurs further downstream:

```text
ALB
 |
 v
Listener
 |
 v
Listener Rule
 |
 v
Target Group
```

The route's backend changes from:

```text
BLUE
```

to:

```text
GREEN
```

---

# 28. Before and After

## Before Switch

```text
Route 53
   |
   v
ALB
   |
   v
Listener :443
   |
   v
HTTPRoute
   |
   v
user-service-blue
   |
   v
Blue Target Group
   |
   +---- 10.0.1.10
   +---- 10.0.2.20
   +---- 10.0.3.30
```

Green:

```text
user-service-green
       |
       v
Green Target Group
       |
       +---- 10.0.4.10
       +---- 10.0.5.20
       +---- 10.0.6.30

No production traffic
```

---

## After Switch

```text
Route 53
   |
   v
ALB
   |
   v
Listener :443
   |
   v
HTTPRoute
   |
   v
user-service-green
   |
   v
Green Target Group
   |
   +---- 10.0.4.10
   +---- 10.0.5.20
   +---- 10.0.6.30
```

Blue remains available:

```text
Blue Target Group
       |
       +---- 10.0.1.10
       +---- 10.0.2.20
       +---- 10.0.3.30
```

This is what gives you the ability to roll back quickly.

---

# 29. Rollback

Suppose Green becomes unhealthy after the switch.

The CD workflow can change:

```text
GREEN
  ↓
BLUE
```

by changing the HTTPRoute backend reference back.

```text
GitHub Actions
      |
      v
HTTPRoute
      |
      | backendRefs
      v
GREEN → BLUE
      |
      v
Kubernetes API
      |
      v
AWS Load Balancer Controller
      |
      v
ALB listener rule reconciliation
      |
      v
BLUE Target Group
      |
      v
Blue Pods
```

This is much faster than redeploying Blue.

Blue is already running.

---

# 30. Automatic Rollback Logic

A good CD pipeline should conceptually look like this:

```text
                    START
                      |
                      v
             Deploy GREEN
                      |
                      v
             Wait for rollout
                      |
                      v
            Pods Ready?
             /        \
           NO          YES
           |             |
           v             v
       ROLLBACK      Validate ALB
                         |
                         v
                  Targets Healthy?
                    /          \
                  NO            YES
                  |               |
                  v               v
              ROLLBACK       Smoke Test
                                |
                                v
                         Smoke Test OK?
                          /          \
                        NO            YES
                        |               |
                        v               v
                    ROLLBACK       SWITCH
                                      |
                                      v
                               BLUE → GREEN
                                      |
                                      v
                              Monitor / Verify
                                      |
                                      v
                              Production OK?
                               /          \
                             NO            YES
                             |               |
                             v               v
                          ROLLBACK          DONE
                             |
                             v
                         GREEN → BLUE
```

---

# 31. Important: What Does "Rollback" Actually Mean?

In your architecture, rollback should primarily mean:

```text
Change HTTPRoute backend

GREEN
  ↓
BLUE
```

It does not necessarily mean:

```text
Delete Green
Redeploy Blue
Recreate ALB
Change Route53
```

That would defeat much of the purpose of blue-green deployment.

The fastest rollback is:

```text
HTTPRoute:

GREEN → BLUE
```

provided Blue is still healthy and available.

---

# 32. Full Resource Relationship

The complete Kubernetes/AWS relationship can be visualized as:

```text
                    GatewayClass
                         |
                         | controllerName
                         v
              AWS Load Balancer Controller
                         |
                         v
                      Gateway
                         |
                         | creates/manages
                         v
                        ALB
                         |
                         | Listener
                         v
                     HTTPRoute
                         |
                         | backendRefs
                         v
                      Service
                         |
                         v
                    EndpointSlice
                         |
                         v
                       TGB
                         |
                         v
                  AWS Target Group
                         |
                         v
                     Pod IPs
```

And alongside this:

```text
TGC
 |
 +---- targetType
 |
 +---- health checks
 |
 +---- target group attributes
 |
 +---- target group customization
```

---

# 33. More Accurate Internal Model

It is useful to think about two separate control planes.

## Kubernetes control plane

```text
GitHub Actions
      |
      v
Kubernetes API
      |
      +---- GatewayClass
      |
      +---- Gateway
      |
      +---- HTTPRoute
      |
      +---- Service
      |
      +---- EndpointSlice
      |
      +---- TGC
      |
      +---- TGB
```

## AWS control plane

```text
AWS Load Balancer Controller
             |
             v
          AWS APIs
             |
      +------+------+
      |             |
      v             v
     ALB       Target Groups
      |             |
      |             v
      |        Registered Pods
      |
      v
Listeners
      |
      v
Listener Rules
```

The AWS Load Balancer Controller is the bridge between these two worlds.

---

# 34. The Reconciliation Loop

The controller's job is essentially:

```text
           Desired State
                |
                v
       Kubernetes Resources
                |
                v
       AWS Load Balancer
          Controller
                |
                v
         Current AWS State
                |
                v
       Compare desired vs
          actual state
                |
          +-----+-----+
          |           |
       Same        Different
          |           |
          v           v
        Wait       Reconcile
                      |
                      v
                  AWS APIs
                      |
                      v
               New AWS state
```

Then it repeats.

This is why manually changing AWS resources behind the controller is a bad idea.

For example:

```text
AWS Console
   |
   v
Manually change listener
```

while Kubernetes says:

```text
HTTPRoute → GREEN
```

The controller can reconcile AWS back toward the Kubernetes desired state.

**Kubernetes resources should be treated as the source of truth.**

---

# 35. What GitHub Actions Actually Controls

Your CD pipeline should primarily control:

```text
Application deployment
        |
        v
Kubernetes manifests
        |
        v
Green environment
        |
        v
Validation
        |
        v
HTTPRoute backendRef
        |
        v
Traffic switch
```

It should NOT need to directly control:

```text
AWS ALB listener APIs
AWS TargetGroup APIs
AWS RegisterTargets API
```

The AWS Load Balancer Controller handles that infrastructure reconciliation.

This is cleaner and much safer.

---

# 36. Complete Blue-Green Sequence

For your DevSecOps platform, the end-to-end sequence can be represented as:

```text
                    USER
                     |
                     v
              api.example.com
                     |
                     v
                  Route53
                     |
                     v
                    ALB
                     |
                     v
              ALB HTTPS Listener
                     |
                     v
              HTTPRoute Rule
                     |
              +------+------+
              |             |
           BLUE active    GREEN inactive
              |             |
              v             v
        Blue Target      Green Target
          Group            Group
              |             |
              v             v
          Blue Pods      Green Pods
```

Then CD starts:

```text
GitHub Actions
      |
      v
Build image
      |
      v
Push image to ECR
      |
      v
Deploy GREEN
      |
      v
Green Pods
      |
      v
EndpointSlices
      |
      v
ALB Controller
      |
      v
Green Target Group
      |
      v
Health checks
      |
      v
Smoke tests
      |
      v
UPDATE HTTPRoute
      |
      v
BLUE → GREEN
      |
      v
ALB Controller
      |
      v
ALB Listener Rule
      |
      v
GREEN Target Group
      |
      v
GREEN Pods
      |
      v
Production traffic
```

---

# 37. Where TGC Fits in This Flow

TGC does not decide:

```text
Blue or Green?
```

That is the job of the route/backend relationship.

Instead, TGC answers questions such as:

```text
What target type should this Target Group use?

How should health checks behave?

What Target Group attributes should be configured?

What custom Target Group behavior should apply?
```

For example:

```text
HTTPRoute
    |
    | "send traffic to"
    v
user-service-green
    |
    v
Target Group Green
    ^
    |
    | configuration
    |
   TGC
```

---

# 38. Where TGB Fits

TGB is the binding layer:

```text
Kubernetes Service
       |
       |
       v
TargetGroupBinding
       |
       |
       v
AWS Target Group
```

The controller uses this relationship to manage target registration.

For normal ALBC-managed Gateway resources, you generally don't need to manually write TGB resources because the controller internally uses/creates the binding.

---

# 39. One Critical Distinction

Do not mix these three concepts:

```text
HTTPRoute
TargetGroupConfiguration
TargetGroupBinding
```

They answer three different questions.

### HTTPRoute

> **Where should HTTP traffic go?**

```text
/api → user-service-green
```

### TGC

> **How should the Target Group behave?**

```text
targetType = ip
healthCheckPath = /health
deregistrationDelay = 30
```

### TGB

> **Which Kubernetes Service is associated with which AWS Target Group?**

```text
user-service-green
        ↕
Green Target Group
```

---

# 40. Final Architecture

```text
                              INTERNET
                                  |
                                  v
                            +-----------+
                            | Route 53  |
                            +-----+-----+
                                  |
                                  | DNS
                                  v
                         +-------------------+
                         |       AWS ALB     |
                         +---------+---------+
                                   |
                              HTTPS :443
                                   |
                                   v
                         +-------------------+
                         | ALB Listener      |
                         | / Listener Rules  |
                         +---------+---------+
                                   |
                                   | derived from
                                   v
                         +-------------------+
                         |    HTTPRoute      |
                         |                   |
                         | host/path rules   |
                         | backendRefs       |
                         +---------+---------+
                                   |
                         backendRefs
                                   |
                    +--------------+--------------+
                    |                             |
                    v                             v
             BLUE SERVICE                  GREEN SERVICE
                    |                             |
                    v                             v
             BLUE TGB/TG                    GREEN TGB/TG
                    |                             |
                    v                             v
             BLUE TARGETS                   GREEN TARGETS
                    |                             |
             +------+------+                +-----+------+
             |      |      |                |     |      |
             v      v      v                v     v      v
           Pod    Pod    Pod              Pod   Pod    Pod
           IP     IP     IP               IP    IP     IP


        Gateway API / AWS Controller Control Plane
        --------------------------------------------

                    GatewayClass
                         |
                         v
                      Gateway
                         |
                         v
             AWS Load Balancer Controller
                         |
              +----------+----------+
              |                     |
              v                     v
        ALB reconciliation    Target reconciliation
              |                     |
              v                     v
        Listeners/rules       Target Groups
                                    |
                                    v
                               Pod IPs


        Target Group Configuration
        ---------------------------

                  TGC
                   |
                   +---- targetType
                   +---- health checks
                   +---- TG attributes
                   +---- TG customization


        Traffic Switch
        ---------------

             GitHub Actions CD
                    |
                    v
              HTTPRoute patch
                    |
                    v
             BLUE → GREEN
                    |
                    v
          Kubernetes API Server
                    |
                    v
          AWS Load Balancer Controller
                    |
                    v
             ALB reconciliation
                    |
                    v
             GREEN Target Group
                    |
                    v
                Green Pods
```

---

# 41. The Most Important Mental Model

If you remember only one thing, remember this:

```text
GitHub Actions
      |
      | changes Kubernetes desired state
      v
HTTPRoute
      |
      | backendRef
      v
Service
      |
      | EndpointSlices
      v
Pod IPs
```

And:

```text
AWS Load Balancer Controller
      |
      | watches Kubernetes
      |
      +--------------------------+
      |                          |
      v                          v
Gateway / HTTPRoute          Service/Endpoints
      |                          |
      v                          v
ALB Listener/Rules          Target Groups
                                 |
                                 v
                              Pod IPs
```

Therefore the complete relationship is:

```text
              GITHUB ACTIONS
                     |
                     v
             Kubernetes API
                     |
          +----------+----------+
          |                     |
          v                     v
      HTTPRoute             Deployment
          |                     |
          v                     v
       Service                 Pods
          |                     |
          v                     v
    EndpointSlice           Pod IPs
          |                     |
          +----------+----------+
                     |
                     v
          AWS Load Balancer
             Controller
                     |
          +----------+----------+
          |                     |
          v                     v
         ALB              Target Groups
          |                     |
          |                     v
          |                  Pod IPs
          |
          v
      Listener Rules
          |
          v
      HTTP routing
```

---

# 42. Blue-Green in One Sentence

Your production blue-green mechanism is essentially:

> **GitHub Actions deploys and validates Green, then changes the Kubernetes `HTTPRoute` backend from Blue Service to Green Service; the AWS Load Balancer Controller observes that desired-state change and reconciles the AWS ALB listener/routing configuration so production traffic reaches the Green Target Group and its healthy Pod IPs.**

That is the core mechanism.

And when rollback is required:

```text
HTTPRoute

GREEN
  ↓
BLUE

        ↓

AWS Load Balancer Controller
        ↓
ALB listener/rule reconciliation
        ↓
Blue Target Group
        ↓
Blue Pod IPs
```

No Route 53 change is required, and no ALB recreation is inherently required.

---

# 43. Operational Debugging Chain

When something goes wrong, troubleshoot from left to right:

```text
Route53
   ↓
ALB
   ↓
Listener
   ↓
Listener Rule
   ↓
Gateway
   ↓
HTTPRoute
   ↓
Service
   ↓
EndpointSlice
   ↓
TargetGroupBinding
   ↓
Target Group
   ↓
Target health
   ↓
Pod IP
   ↓
Pod
```

Useful commands:

```bash
kubectl get gatewayclass

kubectl get gateway -A

kubectl get httproute -A

kubectl describe httproute <name> -n <namespace>

kubectl get svc -n devsecops-blue

kubectl get svc -n devsecops-green

kubectl get endpointslice -n devsecops-blue

kubectl get endpointslice -n devsecops-green

kubectl get targetgroupbindings -A

kubectl describe targetgroupbinding <name> -n <namespace>

kubectl get pods -n devsecops-blue -o wide

kubectl get pods -n devsecops-green -o wide
```

For the controller:

```bash
kubectl get pods -n kube-system \
  -l app.kubernetes.io/name=aws-load-balancer-controller

kubectl logs -n kube-system \
  -l app.kubernetes.io/name=aws-load-balancer-controller
```

Then verify AWS:

```text
ALB
 ├── Listeners
 ├── Listener Rules
 ├── Target Groups
 └── Target Health
```

---

# 44. Final Responsibility Matrix

| Component                    | Responsibility                                                             |
| ---------------------------- | -------------------------------------------------------------------------- |
| Route 53                     | DNS resolution                                                             |
| ALB                          | Layer-7 entry point                                                        |
| ALB Listener                 | Accepts HTTP/HTTPS traffic                                                 |
| ALB Listener Rule            | Determines which Target Group receives request                             |
| GatewayClass                 | Selects the Gateway controller                                             |
| Gateway                      | Defines the ALB/Gateway entry point and listeners                          |
| HTTPRoute                    | Defines HTTP host/path/backend routing                                     |
| Service                      | Stable Kubernetes backend abstraction                                      |
| Deployment                   | Creates/manages Pods                                                       |
| Pod                          | Runs application                                                           |
| VPC CNI                      | Provides Pod networking/IPs                                                |
| EndpointSlice                | Tracks current Service endpoints                                           |
| TGC                          | Configures Target Group behavior                                           |
| TGB                          | Binds AWS Target Group to Kubernetes Service                               |
| AWS Load Balancer Controller | Reconciles Kubernetes Gateway/Service/endpoint state with AWS ALB/TG state |
| GitHub Actions               | Deploys application and changes desired traffic state                      |
| ECR                          | Stores application container images                                        |

---

# 45. The One-Line Architecture

```text
Route53
  → ALB
  → Listener
  → HTTPRoute-derived Listener Rule
  → Target Group
  → Pod IP
  → Application
```

And the control path is:

```text
GitHub Actions
  → Kubernetes API
  → HTTPRoute / Deployment
  → AWS Load Balancer Controller
  → AWS APIs
  → ALB / Listener Rules / Target Groups / Target Registrations
```

**That separation is the key to understanding the entire architecture.**
