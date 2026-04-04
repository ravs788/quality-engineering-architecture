# Test Architecture - Visual Diagrams

## 1. Testing Pyramid by Service

```mermaid
graph TB
    subgraph "Testing Pyramid - Microservices Platform"
        E2E[E2E Tests<br/>10-20 tests<br/>Complete user journeys]
        INT[Integration Tests<br/>50-100 tests<br/>Service interactions]
        COMP[Component Tests<br/>200-500 tests<br/>Service APIs + DB]
        UNIT[Unit Tests<br/>1000+ tests<br/>Functions & Classes]
        
        E2E --> INT
        INT --> COMP
        COMP --> UNIT
    end
    
    style E2E fill:#ff6b6b
    style INT fill:#ffa07a
    style COMP fill:#87ceeb
    style UNIT fill:#90ee90
```

## 2. Per-Service CI/CD Pipeline Flow

```mermaid
flowchart LR
    subgraph "Developer Workstation"
        DEV[Code Changes]
        PRE[Pre-commit Hooks<br/>Lint + Unit Tests]
    end
    
    subgraph "CI Pipeline - Per Service"
        UNIT[Unit Tests<br/>Coverage: 80%+]
        COMP[Component Tests<br/>Testcontainers]
        CON[Contract Tests<br/>Pact Consumer]
        BUILD[Build Docker<br/>Image]
    end
    
    subgraph "Cross-Service Verification"
        PACT[Pact Broker<br/>Verify Contracts]
        INTEG[Integration Tests<br/>Multi-service]
    end
    
    subgraph "Staging"
        DEPLOY_STG[Deploy to<br/>Staging K8s]
        E2E[E2E Tests<br/>Playwright]
        PERF[Performance<br/>Tests K6]
        SEC[Security<br/>Scans]
    end
    
    subgraph "Production"
        CANARY[Canary<br/>Deployment]
        SMOKE[Smoke Tests]
        MONITOR[Synthetic<br/>Monitoring]
    end
    
    DEV --> PRE
    PRE --> UNIT
    UNIT --> COMP
    COMP --> CON
    CON --> BUILD
    BUILD --> PACT
    PACT --> INTEG
    INTEG --> DEPLOY_STG
    DEPLOY_STG --> E2E
    E2E --> PERF
    PERF --> SEC
    SEC --> CANARY
    CANARY --> SMOKE
    SMOKE --> MONITOR
    
    style UNIT fill:#90ee90
    style COMP fill:#87ceeb
    style CON fill:#dda0dd
    style E2E fill:#ffa07a
    style CANARY fill:#ff6b6b
```

## 3. Contract Testing Flow (Pact)

```mermaid
sequenceDiagram
    participant OC as Order Service<br/>(Consumer)
    participant PB as Pact Broker
    participant RS as Restaurant Service<br/>(Provider)
    participant CI as CI Pipeline
    
    Note over OC: Developer writes test
    OC->>OC: Define contract expectation<br/>GET /restaurants/123
    OC->>OC: Run Pact test<br/>Generate contract JSON
    OC->>PB: Publish contract
    
    Note over RS: Provider CI triggered
    RS->>PB: Fetch contracts<br/>for Restaurant Service
    PB->>RS: Return all consumer contracts
    RS->>RS: Start real service
    RS->>RS: Replay contract requests<br/>against real API
    
    alt Contract Satisfied
        RS->>PB: Publish verification SUCCESS
        PB->>CI: ✅ Can deploy both services
    else Contract Broken
        RS->>PB: Publish verification FAILURE
        PB->>CI: ❌ Block deployment
        CI->>OC: Notify: Breaking change detected
    end
```

## 4. Test Environment Progression

```mermaid
graph LR
    subgraph "Local Dev"
        L1[Unit Tests<br/>Watch Mode]
        L2[Component Tests<br/>Docker Compose]
        L3[Contract Tests<br/>Mock Provider]
    end
    
    subgraph "CI Environment"
        C1[All Per-Service Tests<br/>Isolated]
        C2[Contract Verification<br/>Pact Broker]
        C3[Integration Tests<br/>Limited Services]
    end
    
    subgraph "Staging"
        S1[All Services<br/>Production Config]
        S2[E2E Tests<br/>Critical Flows]
        S3[Performance Tests<br/>Load/Stress]
        S4[Security Scans<br/>OWASP/Snyk]
    end
    
    subgraph "Production"
        P1[Canary Deploy<br/>5% Traffic]
        P2[Smoke Tests<br/>Health Checks]
        P3[Synthetic Monitoring<br/>24/7]
    end
    
    L1 --> L2 --> L3
    L3 --> C1
    C1 --> C2 --> C3
    C3 --> S1
    S1 --> S2 --> S3 --> S4
    S4 --> P1
    P1 --> P2 --> P3
    
    style L1 fill:#e0ffe0
    style C1 fill:#e0f0ff
    style S1 fill:#fff0e0
    style P1 fill:#ffe0e0
```

## 5. Microservices Test Architecture - Complete View

```mermaid
graph TB
    subgraph "API Gateway"
        GW[API Gateway<br/>Kong/Nginx]
    end
    
    subgraph "Services Layer"
        US[User Service]
        RS[Restaurant Service]
        OS[Order Service]
        NS[Notification Service]
        AS[Analytics Service]
    end
    
    subgraph "Infrastructure"
        DB[(Databases<br/>Per Service)]
        KAFKA[Kafka<br/>Event Bus]
        CACHE[Redis Cache]
    end
    
    subgraph "Test Layers"
        direction TB
        
        subgraph "Per-Service Tests"
            UT[Unit Tests<br/>Fast, Isolated]
            CT[Component Tests<br/>Service + DB]
            PT[Pact Consumer<br/>Contracts]
        end
        
        subgraph "Cross-Service Tests"
            PV[Pact Provider<br/>Verification]
            IT[Integration Tests<br/>2-3 Services]
        end
        
        subgraph "System Tests"
            E2E[E2E Tests<br/>Full Flows]
            PERF[Performance<br/>Tests]
            SEC[Security<br/>Scans]
        end
    end
    
    GW --> US & RS & OS & NS & AS
    US & RS & OS & NS & AS --> DB
    OS & NS & AS --> KAFKA
    US & RS & OS --> CACHE
    
    UT -.Tests.-> US & RS & OS & NS & AS
    CT -.Tests.-> US & RS & OS & NS & AS
    PT -.Defines.-> US & RS & OS & NS & AS
    PV -.Verifies.-> US & RS & OS & NS & AS
    IT -.Tests.-> US & RS & OS & NS & AS
    E2E -.Tests.-> GW
    PERF -.Tests.-> GW
    SEC -.Scans.-> US & RS & OS & NS & AS
    
    style US fill:#a8e6cf
    style RS fill:#ffd3b6
    style OS fill:#ffaaa5
    style NS fill:#ff8b94
    style AS fill:#c9b6e4
```

## 6. Test Doubles Strategy

```mermaid
graph TB
    subgraph "Order Service Component Test"
        direction LR
        
        OT[Order Service<br/>Under Test]
        
        subgraph "Real Dependencies"
            DB[(PostgreSQL<br/>Testcontainer)]
            KAFKA_TC[Kafka<br/>Testcontainer]
        end
        
        subgraph "Mocked Dependencies"
            REST_MOCK[Restaurant Service<br/>WireMock]
            USER_MOCK[User Service<br/>WireMock]
            PAY_MOCK[Payment Gateway<br/>Stripe Test Mode]
        end
        
        OT --> DB
        OT --> KAFKA_TC
        OT -.HTTP.-> REST_MOCK
        OT -.HTTP.-> USER_MOCK
        OT -.HTTP.-> PAY_MOCK
    end
    
    style OT fill:#ff8b94
    style DB fill:#90ee90
    style KAFKA_TC fill:#90ee90
    style REST_MOCK fill:#ffaaa5
    style USER_MOCK fill:#ffaaa5
    style PAY_MOCK fill:#ffaaa5
```

## 7. Service Communication & Testing Points

```mermaid
graph LR
    subgraph "Order Service"
        direction TB
        O_API[REST API]
        O_KAFKA[Kafka Producer]
    end
    
    subgraph "Restaurant Service"
        direction TB
        R_API[REST API]
    end
    
    subgraph "Notification Service"
        direction TB
        N_KAFKA[Kafka Consumer]
        N_EMAIL[Email Provider]
    end
    
    subgraph "Analytics Service"
        direction TB
        A_KAFKA[Kafka Consumer]
    end
    
    O_API -->|REST Call<br/>🧪 Contract Test| R_API
    O_KAFKA -->|Event: OrderPlaced<br/>🧪 Schema Test| N_KAFKA
    O_KAFKA -->|Event: OrderPlaced<br/>🧪 Integration Test| A_KAFKA
    N_KAFKA -->|🧪 Mock in Tests| N_EMAIL
    
    style O_API fill:#ffaaa5
    style R_API fill:#ffd3b6
    style N_KAFKA fill:#ff8b94
    style A_KAFKA fill:#c9b6e4
```

## 8. Test Execution Timeline (CI/CD)

```mermaid
gantt
    title Test Execution in CI/CD Pipeline
    dateFormat mm:ss
    axisFormat %M:%S
    
    section Per-Service
    Unit Tests           :00:00, 05:00
    Component Tests      :05:00, 05:00
    Contract Tests       :10:00, 02:00
    Build & Push         :12:00, 03:00
    
    section Cross-Service
    Contract Verification:15:00, 03:00
    Integration Tests    :18:00, 10:00
    
    section Staging
    Deploy to Staging    :28:00, 05:00
    E2E Tests           :33:00, 30:00
    Performance Tests    :63:00, 60:00
    Security Scans       :123:00, 30:00
    
    section Production
    Canary Deploy        :153:00, 10:00
    Smoke Tests          :163:00, 05:00
```

## 9. Feedback Loop Speed

```mermaid
graph LR
    subgraph "Fast Feedback (< 15 min)"
        U[Unit Tests<br/>2-5 min]
        C[Component Tests<br/>5-10 min]
        P[Contract Tests<br/>2-3 min]
    end
    
    subgraph "Medium Feedback (15-60 min)"
        I[Integration Tests<br/>10-20 min]
        E[E2E Tests<br/>20-40 min]
    end
    
    subgraph "Slow Feedback (1-4 hours)"
        PERF[Performance Tests<br/>1-2 hours]
        SEC[Security Scans<br/>30-60 min]
    end
    
    subgraph "Continuous (24/7)"
        MON[Production Monitoring<br/>Real-time]
    end
    
    U --> C --> P --> I --> E --> PERF --> SEC --> MON
    
    style U fill:#90ee90
    style C fill:#87ceeb
    style I fill:#ffa07a
    style PERF fill:#ff6b6b
    style MON fill:#dda0dd
```

## 10. Independent Deployment with Safety Nets

```mermaid
flowchart TB
    subgraph "Service A Development"
        A1[Code Change<br/>Service A]
        A2[Unit/Component<br/>Tests Pass]
        A3[Contract Tests<br/>Define Expectations]
        A4[Deploy Service A]
    end
    
    subgraph "Service B Development"
        B1[Code Change<br/>Service B]
        B2[Unit/Component<br/>Tests Pass]
        B3[Contract Verification<br/>Satisfies Service A]
        B4[Deploy Service B]
    end
    
    subgraph "Safety Mechanisms"
        PACT[Pact Broker<br/>Contract Registry]
        VERSION[API Versioning<br/>Backward Compatible]
        FEATURE[Feature Flags<br/>Runtime Control]
    end
    
    A1 --> A2
    A2 --> A3
    A3 --> PACT
    PACT --> A4
    
    B1 --> B2
    B2 --> B3
    B3 --> PACT
    PACT --> B4
    
    A4 --> VERSION
    B4 --> VERSION
    VERSION --> FEATURE
    
    style A4 fill:#90ee90
    style B4 fill:#87ceeb
    style PACT fill:#dda0dd
```

---

## Diagram Annotations

### Key Principles Illustrated

1. **Testing Pyramid:** Emphasizes more unit tests, fewer E2E tests
2. **Fast Feedback:** Per-service tests complete in < 15 minutes
3. **Contract Testing:** Enables independent deployment without full integration tests
4. **Progressive Environments:** Each environment adds confidence layers
5. **Test Doubles:** Strategic use of mocks/stubs to isolate services
6. **Parallel Execution:** Services tested independently, only critical paths tested together

### Color Coding

- 🟢 **Green:** Fast, reliable tests (Unit, Component)
- 🔵 **Blue:** Integration points (Cross-service tests)
- 🟠 **Orange:** Slower, higher-level tests (E2E, Performance)
- 🔴 **Red:** Production/Critical paths
- 🟣 **Purple:** Infrastructure/Coordination (Pact Broker, Monitoring)

### Critical Success Factors

✅ **Independent Pipelines:** Each service has its own pipeline  
✅ **Contract-Driven:** Pact prevents breaking changes  
✅ **Fast Unit/Component Tests:** Developers get feedback in minutes  
✅ **Minimal E2E:** Only critical user journeys tested end-to-end  
✅ **Production Monitoring:** Tests don't end at deployment
