# Test Architecture Strategy: Food Ordering Platform

**Last Updated:** 2026-03-22  
**Platform:** Microservices-based Food Ordering System  
**Architecture:** Docker + Kubernetes + REST/Kafka

---

## Executive Summary

This document outlines the test architecture for a microservices-based online food ordering platform. The strategy supports **independent service deployment** while maintaining **system-level reliability** through layered testing, contract-based decoupling, and progressive test environments.

---

## 1. System Overview

### Microservices Components
- **User Service** - Registration, authentication, user profiles
- **Restaurant Service** - Restaurant catalog, menus, availability
- **Order Service** - Order management, status tracking, payments
- **Notification Service** - Email, SMS, push notifications
- **Analytics Service** - Data aggregation, user behavior insights

### Infrastructure
- **Containerization:** Docker
- **Orchestration:** Kubernetes
- **Communication:** REST APIs (sync) + Kafka (async events)
- **Frontend:** Mobile/Web via API Gateway

---

## 2. Test Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          PRODUCTION ENVIRONMENT                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Smoke Tests (Post-Deployment) + Synthetic Monitoring               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ▲
                                    │ Deploy if all tests pass
                                    │
┌─────────────────────────────────────────────────────────────────────────────┐
│                       STAGING / PRE-PROD ENVIRONMENT                         │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────────┐ │
│  │  E2E Tests       │  │ Performance      │  │ Security Scanning        │ │
│  │  (Full flows)    │  │ (Load/Stress)    │  │ (OWASP ZAP, Snyk)        │ │
│  │  - Selenium/     │  │ - JMeter/K6      │  │ - Dependency Check       │ │
│  │    Playwright    │  │ - Grafana K6     │  │ - Container Scanning     │ │
│  └──────────────────┘  └──────────────────┘  └──────────────────────────┘ │
│                                                                              │
│  All services deployed together (production-like configuration)             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ▲
                                    │ Triggered after integration tests pass
                                    │
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CI ENVIRONMENT (Per-Service)                          │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │              CROSS-SERVICE INTEGRATION TESTS                       │    │
│  │  ┌──────────────────────────────────────────────────────────┐     │    │
│  │  │  Contract Verification (Pact Broker)                     │     │    │
│  │  │  - Provider verifies contracts from all consumers        │     │    │
│  │  └──────────────────────────────────────────────────────────┘     │    │
│  │  ┌──────────────────────────────────────────────────────────┐     │    │
│  │  │  Service Integration Tests                               │     │    │
│  │  │  - Real service + Mocked dependencies                    │     │    │
│  │  │  - Testcontainers (DB, Kafka)                            │     │    │
│  │  └──────────────────────────────────────────────────────────┘     │    │
│  └────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ▲
                                    │ Triggered on PR/Commit
                                    │
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PER-SERVICE BUILD PIPELINE (CI)                           │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────┐        │
│  │                      USER SERVICE                              │        │
│  │  ┌─────────────────┐  ┌──────────────────┐  ┌───────────────┐ │        │
│  │  │ Unit Tests      │→ │ Component Tests  │→ │ Contract Tests│ │        │
│  │  │ (Jest/Pytest)   │  │ (In-Memory DB)   │  │ (Pact - Cons.)│ │        │
│  │  │ - Business      │  │ - API Endpoints  │  │ - Define      │ │        │
│  │  │   Logic         │  │ - With Mocks     │  │   Consumer    │ │        │
│  │  │ - 80%+ Cov.     │  │ - Without Deps   │  │   Contracts   │ │        │
│  │  └─────────────────┘  └──────────────────┘  └───────────────┘ │        │
│  └────────────────────────────────────────────────────────────────┘        │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────┐        │
│  │                   RESTAURANT SERVICE                           │        │
│  │  ┌─────────────────┐  ┌──────────────────┐  ┌───────────────┐ │        │
│  │  │ Unit Tests      │→ │ Component Tests  │→ │ Contract Tests│ │        │
│  │  └─────────────────┘  └──────────────────┘  └───────────────┘ │        │
│  └────────────────────────────────────────────────────────────────┘        │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────┐        │
│  │                      ORDER SERVICE                             │        │
│  │  ┌─────────────────┐  ┌──────────────────┐  ┌───────────────┐ │        │
│  │  │ Unit Tests      │→ │ Component Tests  │→ │ Contract Tests│ │        │
│  │  │                 │  │ + Event Tests    │  │ (Provider +   │ │        │
│  │  │                 │  │ (Kafka Events)   │  │  Consumer)    │ │        │
│  │  └─────────────────┘  └──────────────────┘  └───────────────┘ │        │
│  └────────────────────────────────────────────────────────────────┘        │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────┐        │
│  │                  NOTIFICATION SERVICE                          │        │
│  │  ┌─────────────────┐  ┌──────────────────┐  ┌───────────────┐ │        │
│  │  │ Unit Tests      │→ │ Component Tests  │→ │ Contract Tests│ │        │
│  │  │                 │  │ (Mock Email/SMS) │  │               │ │        │
│  │  └─────────────────┘  └──────────────────┘  └───────────────┘ │        │
│  └────────────────────────────────────────────────────────────────┘        │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────┐        │
│  │                   ANALYTICS SERVICE                            │        │
│  │  ┌─────────────────┐  ┌──────────────────┐  ┌───────────────┐ │        │
│  │  │ Unit Tests      │→ │ Component Tests  │→ │ Contract Tests│ │        │
│  │  │                 │  │ (Event Consumer) │  │               │ │        │
│  │  └─────────────────┘  └──────────────────┘  └───────────────┘ │        │
│  └────────────────────────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ▲
                                    │
┌─────────────────────────────────────────────────────────────────────────────┐
│                       LOCAL DEVELOPMENT ENVIRONMENT                          │
│  ┌──────────────────────────────────────────────────────────────┐          │
│  │  Developer Workstation                                       │          │
│  │  - Unit tests (watch mode)                                   │          │
│  │  - Component tests (docker-compose for dependencies)         │          │
│  │  - Contract tests (generate/verify locally)                  │          │
│  │  - Pre-commit hooks (lint, unit tests)                       │          │
│  └──────────────────────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Testing Layers Explained

### 3.1 Unit Tests
**Scope:** Individual functions, classes, methods  
**Execution:** Per-service build pipeline  
**Tools:** Jest (Node.js), Pytest (Python), JUnit (Java)  
**Coverage Target:** 80%+  

**Characteristics:**
- Fast execution (< 5 minutes per service)
- No external dependencies
- Mock all I/O operations
- Run on every commit

**Example (User Service):**
```python
def test_hash_password():
    """Test password hashing utility"""
    hashed = hash_password("test123")
    assert verify_password("test123", hashed) == True
```

---

### 3.2 Component Tests
**Scope:** Single service with all internal components  
**Execution:** Per-service build pipeline  
**Tools:** Testcontainers, In-memory databases  
**Test Doubles:** Mocked external services  

**Characteristics:**
- Service runs as black box
- Real database (containerized)
- External services mocked via WireMock/MockServer
- API contracts tested
- Event publishing/consumption verified

**Example (Restaurant Service):**
```python
def test_create_restaurant_api():
    """Test REST API endpoint with real DB, mocked dependencies"""
    # Real PostgreSQL via Testcontainers
    # Mock User Service for auth validation
    response = client.post("/restaurants", json={...}, headers=auth_header)
    assert response.status_code == 201
```

---

### 3.3 Contract Tests (Consumer-Driven Contracts)
**Scope:** API interactions between services  
**Execution:** Per-service pipeline + Cross-service verification  
**Tools:** Pact, Spring Cloud Contract  

**How it works:**
1. **Consumer defines contract:** Order Service defines expectations for Restaurant Service API
2. **Contract published:** Stored in Pact Broker
3. **Provider verifies:** Restaurant Service verifies it can satisfy the contract
4. **Decoupling achieved:** Services can deploy independently if contracts are satisfied

**Example Flow:**
```
Order Service (Consumer)          Pact Broker          Restaurant Service (Provider)
─────────────────────                                  ─────────────────────────────
1. Generate contract    ──────→   Store contract  ←────  2. Fetch contracts
                                                         3. Verify against real service
                                                         4. Publish verification result
```

**Benefits:**
- Prevents breaking changes
- Enables independent deployment
- Catches integration issues early

---

### 3.4 Integration Tests
**Scope:** Multiple services interacting  
**Execution:** Cross-service CI pipeline  
**Tools:** Docker Compose, Testcontainers  

**Test Scenarios:**
- User creates order → Order Service calls Restaurant Service
- Order placed → Kafka event → Notification Service sends email
- Payment processed → Analytics Service receives event

**Characteristics:**
- Subset of services run together
- Critical paths tested (not all combinations)
- Real message queue (Kafka in Docker)
- Database per service

---

### 3.5 End-to-End (E2E) Tests
**Scope:** Complete user journeys through UI/API  
**Execution:** Staging environment  
**Tools:** Playwright, Selenium, Cypress  

**Critical Flows:**
1. User registration → Login → Browse restaurants → Place order → Receive notification
2. Restaurant updates menu → User sees updated menu
3. Order cancellation flow

**Characteristics:**
- Run against full system
- Test through API Gateway
- Slow but high confidence
- Minimal set (10-20 critical paths)

---

### 3.6 Performance Tests
**Scope:** Load, stress, endurance  
**Execution:** Staging environment (production-like)  
**Tools:** K6, JMeter, Gatling  

**Test Types:**
- **Load Tests:** Expected traffic (1000 orders/min)
- **Stress Tests:** Breaking point identification
- **Spike Tests:** Sudden traffic surge (lunch rush)
- **Endurance Tests:** 24-hour sustained load

**Key Metrics:**
- Response times (p50, p95, p99)
- Throughput (requests/sec)
- Error rates
- Resource utilization (CPU, memory)

---

### 3.7 Security Tests
**Scope:** Vulnerabilities, compliance  
**Execution:** CI + Staging  
**Tools:** OWASP ZAP, Snyk, Trivy  

**Test Areas:**
- **SAST:** Static code analysis (SonarQube)
- **DAST:** Dynamic application testing (OWASP ZAP)
- **Dependency Scanning:** Known vulnerabilities (Snyk, Dependabot)
- **Container Scanning:** Image vulnerabilities (Trivy)
- **Secrets Scanning:** Prevent credential leaks (GitGuardian)

---

## 4. Test Environments

### 4.1 Local Development
**Purpose:** Rapid feedback for developers  
**Setup:** Docker Compose  
**Characteristics:**
- Full service stack optional
- Lightweight mocks for external services
- Hot reload enabled
- Unit + Component tests run locally

---

### 4.2 CI Environment
**Purpose:** Automated verification on every commit  
**Infrastructure:** GitHub Actions / Jenkins / GitLab CI  
**Characteristics:**
- Ephemeral environments (created per build)
- Isolated per service
- Parallel execution
- Fast feedback (< 10 minutes)

---

### 4.3 Staging / Pre-Production
**Purpose:** Production-like validation  
**Infrastructure:** Kubernetes cluster (mirrored from prod)  
**Characteristics:**
- All services deployed
- Production-like data volume
- Same infrastructure as prod (scaled down)
- Blue-green deployment testing

---

### 4.4 Production
**Purpose:** Smoke tests + Monitoring  
**Tools:** Synthetic monitoring, Datadog, New Relic  
**Characteristics:**
- Post-deployment smoke tests
- Continuous synthetic transactions
- Real user monitoring (RUM)
- Canary deployments

---

## 5. Test Orchestration & CI/CD Pipeline

### 5.1 Per-Service Pipeline (on every commit)

```yaml
# Example: Order Service CI Pipeline
name: Order Service CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Run Unit Tests
        run: npm test -- --coverage
      
      - name: Run Component Tests
        run: |
          docker-compose up -d postgres kafka
          npm run test:component
      
      - name: Generate Consumer Contracts
        run: npm run test:pact
      
      - name: Publish Contracts to Pact Broker
        run: npm run pact:publish
      
      - name: Build Docker Image
        run: docker build -t order-service:${{ github.sha }} .
      
      - name: Push to Registry
        run: docker push order-service:${{ github.sha }}
```

---

### 5.2 Cross-Service Integration Pipeline

**Trigger:** After individual service pipelines pass  
**Execution:**
1. Deploy latest versions of all services
2. Run integration tests
3. Verify contracts (Pact)
4. Run API integration tests

---

### 5.3 Staging Deployment Pipeline

**Trigger:** Manual or after integration tests pass  
**Execution:**
1. Deploy to staging Kubernetes cluster
2. Run E2E tests (Playwright)
3. Run performance tests (K6)
4. Run security scans (OWASP ZAP)
5. Manual approval gate

---

### 5.4 Production Deployment Pipeline

**Trigger:** Manual approval after staging validation  
**Execution:**
1. Canary deployment (5% traffic)
2. Smoke tests
3. Monitoring validation (5 minutes)
4. Gradual rollout (25% → 50% → 100%)
5. Automatic rollback on failure

---

## 6. Test Doubles Strategy

### 6.1 When to Use Test Doubles

| Test Level | Real Components | Mocked/Stubbed |
|-----------|----------------|----------------|
| Unit | Code under test | Everything else |
| Component | Service + DB | External services, message queues |
| Contract | N/A | Contracts define interactions |
| Integration | 2-3 services + infrastructure | External APIs |
| E2E | All services | External payment gateways (sandbox) |

---

### 6.2 Mocking Strategies by Service

#### User Service
**Mocks:**
- Email provider (SendGrid → MockServer)
- SMS gateway (Twilio → Stub)

#### Restaurant Service
**Mocks:**
- User Service authentication (WireMock)
- Image storage (S3 → LocalStack)

#### Order Service
**Mocks:**
- Payment gateway (Stripe → Stripe Test Mode)
- Restaurant Service (for component tests)
- Notification Service (event consumers)

#### Notification Service
**Mocks:**
- Email/SMS providers (all mocked in component tests)

#### Analytics Service
**Mocks:**
- All event producers (use test event generators)

---

### 6.3 Contract Testing with Pact

**Consumer Side (Order Service):**
```javascript
// Define expectation for Restaurant Service
const provider = new Pact.Provider({
  consumer: 'OrderService',
  provider: 'RestaurantService'
});

it('should get restaurant details', async () => {
  await provider.addInteraction({
    state: 'restaurant with ID 123 exists',
    uponReceiving: 'a request for restaurant 123',
    withRequest: {
      method: 'GET',
      path: '/restaurants/123'
    },
    willRespondWith: {
      status: 200,
      body: { id: 123, name: 'Pizza Place' }
    }
  });
  
  // Test Order Service code that calls Restaurant Service
});
```

**Provider Side (Restaurant Service):**
```javascript
// Verify Restaurant Service satisfies contracts
Pact.verifyPacts({
  provider: 'RestaurantService',
  pactBrokerUrl: 'https://pact-broker.example.com',
  providerBaseUrl: 'http://localhost:8080'
});
```

---

## 7. Test Execution Matrix

| Test Type | Frequency | Duration | Environment | Blocking? |
|-----------|-----------|----------|-------------|-----------|
| Unit | Every commit | 2-5 min | CI | Yes |
| Component | Every commit | 5-10 min | CI | Yes |
| Contract (Consumer) | Every commit | 2 min | CI | Yes |
| Contract (Provider) | Every commit | 3 min | CI | Yes |
| Integration | Post-merge | 15 min | CI | Yes |
| E2E | Pre-deployment | 30 min | Staging | Yes |
| Performance | Nightly + Pre-release | 2 hours | Staging | No (report only) |
| Security | Daily + Pre-release | 1 hour | Staging | Yes (critical vulns) |
| Smoke | Post-deployment | 5 min | Production | Yes |

---

## 8. Reporting & Observability

### 8.1 Test Reporting Tools
- **Unit/Component:** JUnit XML → Jenkins/GitHub Actions
- **Coverage:** Codecov, SonarQube
- **Contract:** Pact Broker (contract verification history)
- **E2E:** Playwright HTML reports, video recordings
- **Performance:** Grafana dashboards (K6 metrics)

### 8.2 Test Metrics Tracked
- Test execution time trends
- Flaky test rate (< 1% target)
- Code coverage per service (80%+ target)
- Contract verification success rate
- Deployment frequency (DORA metric)
- Change failure rate (DORA metric)

---

## 9. How This Architecture Supports Independent Deployment

### 9.1 Service Autonomy
✅ **Each service has its own test suite** → Can be tested independently  
✅ **Contract tests prevent breaking changes** → No need to test all service combinations  
✅ **Mocks/stubs decouple services** → Component tests don't require full system  

### 9.2 Fast Feedback Loops
✅ **Unit + Component tests run in < 10 min** → Developers get quick feedback  
✅ **Contract tests verify compatibility** → Integration issues caught before merge  
✅ **Per-service pipelines** → No waiting for other teams  

### 9.3 Confidence Without Full E2E
✅ **Testing pyramid approach** → Heavy on unit/component, light on E2E  
✅ **Contract tests replace most integration tests** → Faster, more reliable  
✅ **Staging E2E only for critical paths** → Safety net without bottleneck  

### 9.4 Deployment Safety
✅ **Per-service pipelines gate deployments** → Bad code never reaches staging  
✅ **Contract verification prevents incompatible changes** → Breaking changes blocked  
✅ **Canary deployments + smoke tests** → Production issues caught early  
✅ **Independent rollback capability** → Failed service doesn't block others  

---

## 10. Key Principles & Best Practices

1. **Test Pyramid:** Majority of tests at unit level, fewer at integration, minimal E2E
2. **Shift Left:** Run tests as early as possible in development cycle
3. **Fast Feedback:** Keep pipeline under 15 minutes for per-service builds
4. **Contract-First:** Use consumer-driven contracts to enable independent deployment
5. **Isolate External Dependencies:** Mock/stub external services, use Testcontainers for infrastructure
6. **Production-Like Staging:** Staging should mirror production configuration
7. **Monitor Everything:** Tests are not enough; synthetic monitoring in production is critical
8. **Fail Fast:** Block deployment on test failures; never skip tests
9. **Parallelize:** Run tests in parallel to reduce pipeline time
10. **Meaningful Tests:** Avoid flaky tests; maintain test quality as rigorously as code quality

---

## 11. Tooling Summary

| Category | Tools |
|----------|-------|
| **Unit Testing** | Jest, Pytest, JUnit, Mocha |
| **API Testing** | RestAssured, Postman/Newman, SuperTest |
| **Contract Testing** | Pact, Spring Cloud Contract |
| **Mocking** | WireMock, MockServer, Mockito |
| **Containers** | Testcontainers, Docker Compose |
| **E2E Testing** | Playwright, Cypress, Selenium |
| **Performance** | K6, JMeter, Gatling |
| **Security** | OWASP ZAP, Snyk, Trivy, SonarQube |
| **CI/CD** | GitHub Actions, Jenkins, GitLab CI |
| **Orchestration** | Kubernetes, Helm |
| **Monitoring** | Datadog, New Relic, Prometheus, Grafana |
| **Contract Broker** | Pact Broker, Pactflow |
| **Test Reporting** | Allure, ReportPortal, TestRail |

---

## 12. Implementation Roadmap

### Phase 1: Foundation (Weeks 1-2)
- [ ] Set up per-service CI pipelines
- [ ] Implement unit tests (80% coverage target)
- [ ] Configure Testcontainers for component tests
- [ ] Establish code coverage reporting

### Phase 2: Service Decoupling (Weeks 3-4)
- [ ] Set up Pact Broker
- [ ] Implement consumer contracts for all services
- [ ] Add provider verification to pipelines
- [ ] Train teams on contract testing

### Phase 3: Integration & E2E (Weeks 5-6)
- [ ] Build cross-service integration test suite
- [ ] Set up staging environment
- [ ] Implement critical E2E flows (10-15 tests)
- [ ] Configure E2E test execution in staging pipeline

### Phase 4: Performance & Security (Weeks 7-8)
- [ ] Design performance test scenarios
- [ ] Set up K6 with Grafana dashboards
- [ ] Integrate security scanning tools
- [ ] Establish performance baselines

### Phase 5: Production Readiness (Weeks 9-10)
- [ ] Implement canary deployment strategy
- [ ] Set up synthetic monitoring
- [ ] Configure automated rollbacks
- [ ] Document runbooks for test failures

---

## Conclusion

This test architecture provides a **comprehensive, layered approach** to testing microservices while maintaining **service autonomy** and **deployment independence**. By emphasizing contract testing, test isolation, and progressive environments, teams can deploy services independently with confidence.

The key to success is balancing **speed** (fast feedback through unit/component tests) with **reliability** (contract tests + critical E2E tests), avoiding the trap of slow, brittle end-to-end test suites that block deployments.
