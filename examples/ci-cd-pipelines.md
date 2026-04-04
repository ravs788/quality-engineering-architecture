# CI/CD Pipeline Configuration Examples

## 1. Per-Service Pipeline (GitHub Actions)

### Order Service - Complete CI Pipeline

```yaml
# .github/workflows/order-service-ci.yml
name: Order Service CI

on:
  push:
    branches: [main, develop]
    paths:
      - 'services/order-service/**'
      - '.github/workflows/order-service-ci.yml'
  pull_request:
    branches: [main, develop]
    paths:
      - 'services/order-service/**'

env:
  SERVICE_NAME: order-service
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}/order-service

jobs:
  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: services/order-service
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
          cache-dependency-path: services/order-service/package-lock.json
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linter
        run: npm run lint
      
      - name: Run unit tests
        run: npm test -- --coverage --ci
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          files: ./services/order-service/coverage/coverage-final.json
          flags: order-service-unit
          name: order-service-unit-coverage
      
      - name: Check coverage threshold
        run: |
          COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "Coverage $COVERAGE% is below 80% threshold"
            exit 1
          fi

  component-tests:
    name: Component Tests
    runs-on: ubuntu-latest
    needs: unit-tests
    defaults:
      run:
        working-directory: services/order-service
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: orders_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      kafka:
        image: confluentinc/cp-kafka:7.4.0
        env:
          KAFKA_BROKER_ID: 1
          KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
          KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
        ports:
          - 9092:9092
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
          cache-dependency-path: services/order-service/package-lock.json
      
      - name: Install dependencies
        run: npm ci
      
      - name: Wait for services
        run: |
          npm install -g wait-on
          wait-on tcp:localhost:5432 -t 30000
          wait-on tcp:localhost:9092 -t 30000
      
      - name: Run database migrations
        run: npm run migrate:test
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/orders_test
      
      - name: Run component tests
        run: npm run test:component
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/orders_test
          KAFKA_BROKERS: localhost:9092
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: component-test-results
          path: services/order-service/test-results/

  contract-tests:
    name: Contract Tests (Consumer)
    runs-on: ubuntu-latest
    needs: component-tests
    defaults:
      run:
        working-directory: services/order-service
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
          cache-dependency-path: services/order-service/package-lock.json
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run Pact consumer tests
        run: npm run test:pact
      
      - name: Publish contracts to Pact Broker
        if: github.ref == 'refs/heads/main'
        run: npm run pact:publish
        env:
          PACT_BROKER_BASE_URL: ${{ secrets.PACT_BROKER_URL }}
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}
          GIT_COMMIT: ${{ github.sha }}
          GIT_BRANCH: ${{ github.ref_name }}

  build-and-push:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: [unit-tests, component-tests, contract-tests]
    if: github.event_name == 'push'
    
    permissions:
      contents: read
      packages: write
    
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Log in to Container Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=sha,prefix={{branch}}-
            type=semver,pattern={{version}}
      
      - name: Build and push Docker image
        id: build
        uses: docker/build-push-action@v4
        with:
          context: services/order-service
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
      
      - name: Sign image with Cosign
        run: |
          cosign sign --key env://COSIGN_KEY "${TAGS}@${DIGEST}"
        env:
          TAGS: ${{ steps.meta.outputs.tags }}
          DIGEST: ${{ steps.build.outputs.digest }}
          COSIGN_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}

  security-scan:
    name: Security Scanning
    runs-on: ubuntu-latest
    needs: build-and-push
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ needs.build-and-push.outputs.image-tag }}
          format: 'sarif'
          output: 'trivy-results.sarif'
      
      - name: Upload Trivy results to GitHub Security
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
      
      - name: Run Snyk security scan
        uses: snyk/actions/node@master
        continue-on-error: true
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          command: test
          args: --severity-threshold=high
      
      - name: Fail on critical vulnerabilities
        run: |
          CRITICAL=$(cat trivy-results.sarif | jq '[.runs[].results[] | select(.level=="error")] | length')
          if [ $CRITICAL -gt 0 ]; then
            echo "Found $CRITICAL critical vulnerabilities"
            exit 1
          fi

  notify:
    name: Notify Build Status
    runs-on: ubuntu-latest
    needs: [unit-tests, component-tests, contract-tests, build-and-push, security-scan]
    if: always()
    
    steps:
      - name: Send Slack notification
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: |
            Order Service CI Pipeline
            Branch: ${{ github.ref_name }}
            Commit: ${{ github.sha }}
            Author: ${{ github.actor }}
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
        if: always()
```

---

## 2. Contract Verification Pipeline (Provider)

### Restaurant Service - Provider Verification

```yaml
# .github/workflows/restaurant-service-contract-verification.yml
name: Restaurant Service - Contract Verification

on:
  push:
    branches: [main, develop]
    paths:
      - 'services/restaurant-service/**'
  workflow_dispatch:
  # Triggered by Pact Broker webhook when new consumer contract is published
  repository_dispatch:
    types: [pact-changed]

jobs:
  verify-contracts:
    name: Verify Consumer Contracts
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: restaurants_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        working-directory: services/restaurant-service
        run: npm ci
      
      - name: Run database migrations
        working-directory: services/restaurant-service
        run: npm run migrate:test
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/restaurants_test
      
      - name: Start service
        working-directory: services/restaurant-service
        run: npm start &
        env:
          PORT: 8080
          DATABASE_URL: postgresql://test:test@localhost:5432/restaurants_test
      
      - name: Wait for service
        run: |
          npm install -g wait-on
          wait-on http://localhost:8080/health -t 30000
      
      - name: Verify Pact contracts
        working-directory: services/restaurant-service
        run: npm run pact:verify
        env:
          PACT_BROKER_BASE_URL: ${{ secrets.PACT_BROKER_URL }}
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}
          PROVIDER_BASE_URL: http://localhost:8080
          GIT_COMMIT: ${{ github.sha }}
          GIT_BRANCH: ${{ github.ref_name }}
      
      - name: Can-i-deploy check
        working-directory: services/restaurant-service
        run: |
          npx pact-broker can-i-deploy \
            --pacticipant RestaurantService \
            --version ${{ github.sha }} \
            --to-environment production
        env:
          PACT_BROKER_BASE_URL: ${{ secrets.PACT_BROKER_URL }}
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}
```

---

## 3. Integration Test Pipeline

### Cross-Service Integration Tests

```yaml
# .github/workflows/integration-tests.yml
name: Integration Tests

on:
  workflow_run:
    workflows: 
      - "Order Service CI"
      - "Restaurant Service CI"
      - "User Service CI"
    types:
      - completed
    branches: [main]
  workflow_dispatch:

jobs:
  integration-tests:
    name: Run Integration Tests
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
          cache: 'pip'
      
      - name: Install dependencies
        run: pip install -r tests/integration/requirements.txt
      
      - name: Create .env file
        run: |
          cat > .env << EOF
          USER_SERVICE_IMAGE=${{ env.REGISTRY }}/user-service:${{ github.sha }}
          RESTAURANT_SERVICE_IMAGE=${{ env.REGISTRY }}/restaurant-service:${{ github.sha }}
          ORDER_SERVICE_IMAGE=${{ env.REGISTRY }}/order-service:${{ github.sha }}
          NOTIFICATION_SERVICE_IMAGE=${{ env.REGISTRY }}/notification-service:${{ github.sha }}
          ANALYTICS_SERVICE_IMAGE=${{ env.REGISTRY }}/analytics-service:${{ github.sha }}
          EOF
      
      - name: Start services with Docker Compose
        run: docker-compose -f docker-compose.integration.yml up -d
      
      - name: Wait for services to be healthy
        run: |
          chmod +x scripts/wait-for-services.sh
          ./scripts/wait-for-services.sh
      
      - name: Run integration tests
        run: pytest tests/integration/ -v --tb=short
      
      - name: Collect logs on failure
        if: failure()
        run: |
          mkdir -p logs
          docker-compose -f docker-compose.integration.yml logs > logs/services.log
      
      - name: Upload logs
        if: failure()
        uses: actions/upload-artifact@v3
        with:
          name: integration-test-logs
          path: logs/
      
      - name: Cleanup
        if: always()
        run: docker-compose -f docker-compose.integration.yml down -v
```

---

## 4. Staging Deployment Pipeline

```yaml
# .github/workflows/deploy-staging.yml
name: Deploy to Staging

on:
  workflow_run:
    workflows: ["Integration Tests"]
    types: [completed]
    branches: [main]
  workflow_dispatch:

env:
  CLUSTER_NAME: food-ordering-staging
  NAMESPACE: staging

jobs:
  deploy:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Update kubeconfig
        run: |
          aws eks update-kubeconfig \
            --name ${{ env.CLUSTER_NAME }} \
            --region us-east-1
      
      - name: Install Helm
        uses: azure/setup-helm@v3
        with:
          version: 'v3.12.0'
      
      - name: Deploy services
        run: |
          helm upgrade --install food-ordering ./helm/food-ordering \
            --namespace ${{ env.NAMESPACE }} \
            --create-namespace \
            --set image.tag=${{ github.sha }} \
            --set environment=staging \
            --values helm/values-staging.yaml \
            --wait \
            --timeout 10m
      
      - name: Wait for rollout
        run: |
          kubectl rollout status deployment/user-service -n ${{ env.NAMESPACE }} --timeout=5m
          kubectl rollout status deployment/restaurant-service -n ${{ env.NAMESPACE }} --timeout=5m
          kubectl rollout status deployment/order-service -n ${{ env.NAMESPACE }} --timeout=5m
          kubectl rollout status deployment/notification-service -n ${{ env.NAMESPACE }} --timeout=5m
          kubectl rollout status deployment/analytics-service -n ${{ env.NAMESPACE }} --timeout=5m
      
      - name: Run smoke tests
        run: |
          chmod +x scripts/smoke-tests.sh
          ./scripts/smoke-tests.sh ${{ secrets.STAGING_URL }}
      
      - name: Record deployment in Pact Broker
        run: |
          npx pact-broker record-deployment \
            --pacticipant OrderService \
            --version ${{ github.sha }} \
            --environment staging
        env:
          PACT_BROKER_BASE_URL: ${{ secrets.PACT_BROKER_URL }}
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}

  e2e-tests:
    name: End-to-End Tests
    runs-on: ubuntu-latest
    needs: deploy
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install Playwright
        run: |
          cd tests/e2e
          npm ci
          npx playwright install --with-deps
      
      - name: Run E2E tests
        run: |
          cd tests/e2e
          npx playwright test
        env:
          BASE_URL: ${{ secrets.STAGING_URL }}
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: playwright-report
          path: tests/e2e/playwright-report/
      
      - name: Upload videos
        if: failure()
        uses: actions/upload-artifact@v3
        with:
          name: playwright-videos
          path: tests/e2e/test-results/

  performance-tests:
    name: Performance Tests
    runs-on: ubuntu-latest
    needs: e2e-tests
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup K6
        run: |
          sudo gpg -k
          sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
          echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
          sudo apt-get update
          sudo apt-get install k6
      
      - name: Run K6 performance tests
        run: k6 run tests/performance/load-test.js
        env:
          BASE_URL: ${{ secrets.STAGING_URL }}
          K6_CLOUD_TOKEN: ${{ secrets.K6_CLOUD_TOKEN }}
      
      - name: Upload K6 results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: k6-results
          path: k6-results.json

  security-tests:
    name: Security Tests (DAST)
    runs-on: ubuntu-latest
    needs: deploy
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: OWASP ZAP Full Scan
        uses: zaproxy/action-full-scan@v0.7.0
        with:
          target: ${{ secrets.STAGING_URL }}
          rules_file_name: '.zap/rules.tsv'
          cmd_options: '-a'
      
      - name: Upload ZAP report
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: zap-report
          path: zap-report.html
```

---

## 5. Production Deployment Pipeline (Canary)

```yaml
# .github/workflows/deploy-production.yml
name: Deploy to Production

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to deploy (git SHA)'
        required: true
      canary-percentage:
        description: 'Initial canary percentage'
        required: false
        default: '5'

env:
  CLUSTER_NAME: food-ordering-prod
  NAMESPACE: production

jobs:
  pre-deployment-checks:
    name: Pre-Deployment Checks
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
        with:
          ref: ${{ github.event.inputs.version }}
      
      - name: Verify all tests passed
        run: |
          # Check that all CI pipelines passed for this commit
          gh api \
            /repos/${{ github.repository }}/commits/${{ github.event.inputs.version }}/check-runs \
            --jq '.check_runs[] | select(.conclusion != "success") | .name' \
            > failed_checks.txt
          
          if [ -s failed_checks.txt ]; then
            echo "Some checks failed:"
            cat failed_checks.txt
            exit 1
          fi
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Check Pact can-i-deploy
        run: |
          npx pact-broker can-i-deploy \
            --pacticipant OrderService \
            --version ${{ github.event.inputs.version }} \
            --to-environment production
        env:
          PACT_BROKER_BASE_URL: ${{ secrets.PACT_BROKER_URL }}
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}

  canary-deployment:
    name: Canary Deployment (${{ github.event.inputs.canary-percentage }}%)
    runs-on: ubuntu-latest
    needs: pre-deployment-checks
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Update kubeconfig
        run: |
          aws eks update-kubeconfig \
            --name ${{ env.CLUSTER_NAME }} \
            --region us-east-1
      
      - name: Deploy canary
        run: |
          helm upgrade food-ordering ./helm/food-ordering \
            --namespace ${{ env.NAMESPACE }} \
            --set image.tag=${{ github.event.inputs.version }} \
            --set canary.enabled=true \
            --set canary.weight=${{ github.event.inputs.canary-percentage }} \
            --values helm/values-production.yaml \
            --wait
      
      - name: Wait for canary rollout
        run: |
          kubectl rollout status deployment/order-service-canary -n ${{ env.NAMESPACE }}
      
      - name: Run smoke tests on canary
        run: |
          chmod +x scripts/smoke-tests.sh
          ./scripts/smoke-tests.sh ${{ secrets.PROD_CANARY_URL }}
      
      - name: Monitor canary metrics
        run: |
          chmod +x scripts/monitor-canary.sh
          ./scripts/monitor-canary.sh 5m
        env:
          GRAFANA_URL: ${{ secrets.GRAFANA_URL }}
          GRAFANA_TOKEN: ${{ secrets.GRAFANA_TOKEN }}

  gradual-rollout:
    name: Gradual Rollout
    runs-on: ubuntu-latest
    needs: canary-deployment
    
    strategy:
      matrix:
        weight: [25, 50, 100]
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Update kubeconfig
        run: |
          aws eks update-kubeconfig \
            --name ${{ env.CLUSTER_NAME }} \
            --region us-east-1
      
      - name: Increase canary to ${{ matrix.weight }}%
        run: |
          helm upgrade food-ordering ./helm/food-ordering \
            --namespace ${{ env.NAMESPACE }} \
            --set canary.weight=${{ matrix.weight }} \
            --reuse-values \
            --wait
      
      - name: Monitor for 10 minutes
        run: |
          chmod +x scripts/monitor-canary.sh
          ./scripts/monitor-canary.sh 10m
        env:
          GRAFANA_URL: ${{ secrets.GRAFANA_URL }}
          GRAFANA_TOKEN: ${{ secrets.GRAFANA_TOKEN }}
          ERROR_RATE_THRESHOLD: 0.01
          LATENCY_P99_THRESHOLD: 1000
      
      - name: Rollback on error
        if: failure()
        run: |
          helm rollback food-ordering -n ${{ env.NAMESPACE }}
          kubectl rollout undo deployment/order-service-canary -n ${{ env.NAMESPACE }}

  finalize-deployment:
    name: Finalize Deployment
    runs-on: ubuntu-latest
    needs: gradual-rollout
    
    steps:
      - name: Remove canary
        run: |
          helm upgrade food-ordering ./helm/food-ordering \
            --namespace ${{ env.NAMESPACE }} \
            --set canary.enabled=false \
            --reuse-values \
            --wait
      
      - name: Record deployment in Pact Broker
        run: |
          npx pact-broker record-deployment \
            --pacticipant OrderService \
            --version ${{ github.event.inputs.version }} \
            --environment production
        env:
          PACT_BROKER_BASE_URL: ${{ secrets.PACT_BROKER_URL }}
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}
      
      - name: Create GitHub release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: v${{ github.event.inputs.version }}
          release_name: Release ${{ github.event.inputs.version }}
          body: |
            Production deployment completed successfully.
            Commit: ${{ github.event.inputs.version }}
          draft: false
          prerelease: false
      
      - name: Notify team
        uses: 8398a7/action-slack@v3
        with:
          status: success
          text: |
            🚀 Production Deployment Successful
            Version: ${{ github.event.inputs.version }}
            Environment: Production
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

---

## 6. Supporting Scripts

### Smoke Tests Script

```bash
#!/bin/bash
# scripts/smoke-tests.sh

set -e

BASE_URL=$1

if [ -z "$BASE_URL" ]; then
  echo "Usage: $0 <base-url>"
  exit 1
fi

echo "Running smoke tests against $BASE_URL"

# Health checks
echo "Checking service health..."
services=("user-service" "restaurant-service" "order-service" "notification-service" "analytics-service")

for service in "${services[@]}"; do
  response=$(curl -s -o /dev/null -w "%{http_code}" "$BASE_URL/$service/health")
  if [ "$response" -ne 200 ]; then
    echo "❌ $service health check failed (HTTP $response)"
    exit 1
  fi
  echo "✅ $service is healthy"
done

# Basic functionality tests
echo "Testing basic functionality..."

# Test user registration
echo "Testing user registration..."
user_response=$(curl -s -X POST "$BASE_URL/users" \
  -H "Content-Type: application/json" \
  -d '{"email":"smoke-test@example.com","password":"Test123!","name":"Smoke Test"}' \
  -w "%{http_code}" -o /tmp/user_response.json)

if [ "${user_response: -3}" -ne 201 ]; then
  echo "❌ User registration failed"
  exit 1
fi
echo "✅ User registration successful"

# Test restaurant listing
echo "Testing restaurant listing..."
rest_response=$(curl -s -o /dev/null -w "%{http_code}" "$BASE_URL/restaurants")
if [ "$rest_response" -ne 200 ]; then
  echo "❌ Restaurant listing failed"
  exit 1
fi
echo "✅ Restaurant listing successful"

echo "All smoke tests passed! ✅"
```

### Canary Monitoring Script

```bash
#!/bin/bash
# scripts/monitor-canary.sh

set -e

DURATION=$1
ERROR_RATE_THRESHOLD=${ERROR_RATE_THRESHOLD:-0.01}
LATENCY_P99_THRESHOLD=${LATENCY_P99_THRESHOLD:-1000}

if [ -z "$DURATION" ]; then
  echo "Usage: $0 <duration> (e.g., 5m, 10m)"
  exit 1
fi

echo "Monitoring canary for $DURATION"
echo "Error rate threshold: ${ERROR_RATE_THRESHOLD}%"
echo "P99 latency threshold: ${LATENCY_P99_THRESHOLD}ms"

# Convert duration to seconds
DURATION_SECONDS=$(echo "$DURATION" | sed 's/m/*60/g' | bc)
END_TIME=$(($(date +%s) + DURATION_SECONDS))

while [ $(date +%s) -lt $END_TIME ]; do
  # Query Prometheus for error rate
  ERROR_RATE=$(curl -s "$GRAFANA_URL/api/datasources/proxy/1/api/v1/query" \
    -H "Authorization: Bearer $GRAFANA_TOKEN" \
    --data-urlencode 'query=sum(rate(http_requests_total{status=~"5.."}[1m])) / sum(rate(http_requests_total[1m]))' \
    | jq -r '.data.result[0].value[1]')
  
  # Query for P99 latency
  P99_LATENCY=$(curl -s "$GRAFANA_URL/api/datasources/proxy/1/api/v1/query" \
    -H "Authorization: Bearer $GRAFANA_TOKEN" \
    --data-urlencode 'query=histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[1m]))' \
    | jq -r '.data.result[0].value[1]')
  
  echo "Current metrics - Error Rate: ${ERROR_RATE}%, P99 Latency: ${P99_LATENCY}ms"
  
  # Check thresholds
  if (( $(echo "$ERROR_RATE > $ERROR_RATE_THRESHOLD" | bc -l) )); then
    echo "❌ Error rate exceeded threshold!"
    exit 1
  fi
  
  if (( $(echo "$P99_LATENCY > $LATENCY_P99_THRESHOLD" | bc -l) )); then
    echo "❌ P99 latency exceeded threshold!"
    exit 1
  fi
  
  sleep 30
done

echo "✅ Canary monitoring completed successfully"
```

---

## 7. Docker Compose for Integration Tests

```yaml
# docker-compose.integration.yml
version: '3.8'

services:
  postgres-user:
    image: postgres:15
    environment:
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
      POSTGRES_DB: users_test
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U test"]
      interval: 5s
      timeout: 5s
      retries: 5

  postgres-restaurant:
    image: postgres:15
    environment:
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
      POSTGRES_DB: restaurants_test
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U test"]
      interval: 5s
      timeout: 5s
      retries: 5

  postgres-order:
    image: postgres:15
    environment:
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
      POSTGRES_DB: orders_test
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U test"]
      interval: 5s
      timeout: 5s
      retries: 5

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    depends_on:
      - zookeeper
    healthcheck:
      test: ["CMD-SHELL", "kafka-broker-api-versions --bootstrap-server localhost:9092"]
      interval: 10s
      timeout: 10s
      retries: 5

  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  user-service:
    image: ${USER_SERVICE_IMAGE}
    environment:
      DATABASE_URL: postgresql://test:test@postgres-user:5432/users_test
      PORT: 8081
    depends_on:
      postgres-user:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:8081/health || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 3

  restaurant-service:
    image: ${RESTAURANT_SERVICE_IMAGE}
    environment:
      DATABASE_URL: postgresql://test:test@postgres-restaurant:5432/restaurants_test
      PORT: 8082
    depends_on:
      postgres-restaurant:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:8082/health || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 3

  order-service:
    image: ${ORDER_SERVICE_IMAGE}
    environment:
      DATABASE_URL: postgresql://test:test@postgres-order:5432/orders_test
      KAFKA_BROKERS: kafka:9092
      RESTAURANT_SERVICE_URL: http://restaurant-service:8082
      USER_SERVICE_URL: http://user-service:8081
      PORT: 8083
    depends_on:
      postgres-order:
        condition: service_healthy
      kafka:
        condition: service_healthy
      restaurant-service:
        condition: service_healthy
      user-service:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:8083/health || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 3

  notification-service:
    image: ${NOTIFICATION_SERVICE_IMAGE}
    environment:
      KAFKA_BROKERS: kafka:9092
      SENDGRID_API_KEY: test-key
      PORT: 8084
    depends_on:
      kafka:
        condition: service_healthy

  analytics-service:
    image: ${ANALYTICS_SERVICE_IMAGE}
    environment:
      KAFKA_BROKERS: kafka:9092
      PORT: 8085
    depends_on:
      kafka:
        condition: service_healthy
```

---

## Summary

This comprehensive CI/CD pipeline setup provides:

1. **Per-Service Pipelines:** Independent build, test, and deployment for each microservice
2. **Contract Testing:** Automated contract verification to prevent breaking changes
3. **Integration Testing:** Cross-service validation before deployment
4. **Progressive Deployment:** Canary releases with automated monitoring and rollback
5. **Security Scanning:** Automated vulnerability detection at multiple stages
6. **Quality Gates:** Coverage thresholds, test results, and security checks block bad deployments

Key features:
- ✅ Fast feedback (< 15 min per service)
- ✅ Independent deployment capability
- ✅ Automated rollback on failure
- ✅ Comprehensive test coverage at all levels
- ✅ Security-first approach with SAST/DAST
- ✅ Production monitoring and canary validation
