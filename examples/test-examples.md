# Test Implementation Examples

## 1. Unit Test Examples

### User Service - Password Hashing (Python/Pytest)

```python
# tests/unit/test_password_utils.py
import pytest
from user_service.utils import hash_password, verify_password

def test_hash_password_creates_valid_hash():
    """Test password hashing creates a valid bcrypt hash"""
    password = "SecurePass123!"
    hashed = hash_password(password)
    
    assert hashed is not None
    assert hashed != password
    assert len(hashed) > 50  # bcrypt hashes are typically 60 chars

def test_verify_password_with_correct_password():
    """Test password verification succeeds with correct password"""
    password = "SecurePass123!"
    hashed = hash_password(password)
    
    assert verify_password(password, hashed) is True

def test_verify_password_with_incorrect_password():
    """Test password verification fails with incorrect password"""
    password = "SecurePass123!"
    hashed = hash_password(password)
    
    assert verify_password("WrongPassword", hashed) is False

def test_hash_password_with_empty_string_raises_error():
    """Test hashing empty password raises ValueError"""
    with pytest.raises(ValueError, match="Password cannot be empty"):
        hash_password("")
```

### Order Service - Order Validation (Node.js/Jest)

```javascript
// tests/unit/orderValidator.test.js
const { validateOrder } = require('../../src/validators/orderValidator');

describe('Order Validator', () => {
  test('should validate a valid order', () => {
    const order = {
      userId: 'user123',
      restaurantId: 'rest456',
      items: [
        { menuItemId: 'item1', quantity: 2, price: 15.99 },
        { menuItemId: 'item2', quantity: 1, price: 8.50 }
      ],
      totalAmount: 40.48
    };

    const result = validateOrder(order);
    expect(result.isValid).toBe(true);
    expect(result.errors).toHaveLength(0);
  });

  test('should reject order with negative quantity', () => {
    const order = {
      userId: 'user123',
      restaurantId: 'rest456',
      items: [
        { menuItemId: 'item1', quantity: -1, price: 15.99 }
      ],
      totalAmount: 15.99
    };

    const result = validateOrder(order);
    expect(result.isValid).toBe(false);
    expect(result.errors).toContain('Quantity must be positive');
  });

  test('should reject order with incorrect total calculation', () => {
    const order = {
      userId: 'user123',
      restaurantId: 'rest456',
      items: [
        { menuItemId: 'item1', quantity: 2, price: 15.99 }
      ],
      totalAmount: 100.00  // Wrong total
    };

    const result = validateOrder(order);
    expect(result.isValid).toBe(false);
    expect(result.errors).toContain('Total amount mismatch');
  });
});
```

---

## 2. Component Test Examples

### Restaurant Service - API Tests with Testcontainers (Python)

```python
# tests/component/test_restaurant_api.py
import pytest
from fastapi.testclient import TestClient
from testcontainers.postgres import PostgresContainer
from restaurant_service.main import app
from restaurant_service.database import get_db

@pytest.fixture(scope="module")
def postgres_container():
    """Spin up PostgreSQL container for tests"""
    with PostgresContainer("postgres:15") as postgres:
        yield postgres

@pytest.fixture(scope="module")
def test_client(postgres_container):
    """Create test client with real database"""
    # Override database connection
    db_url = postgres_container.get_connection_url()
    app.dependency_overrides[get_db] = lambda: get_test_db(db_url)
    
    client = TestClient(app)
    yield client
    
    app.dependency_overrides.clear()

def test_create_restaurant(test_client):
    """Test creating a new restaurant"""
    restaurant_data = {
        "name": "Pizza Palace",
        "cuisine": "Italian",
        "address": "123 Main St",
        "phone": "+1-555-0123"
    }
    
    response = test_client.post("/restaurants", json=restaurant_data)
    
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "Pizza Palace"
    assert data["cuisine"] == "Italian"
    assert "id" in data
    assert "createdAt" in data

def test_get_restaurant_by_id(test_client):
    """Test retrieving restaurant by ID"""
    # First create a restaurant
    create_response = test_client.post("/restaurants", json={
        "name": "Burger Barn",
        "cuisine": "American",
        "address": "456 Oak Ave",
        "phone": "+1-555-0456"
    })
    restaurant_id = create_response.json()["id"]
    
    # Then retrieve it
    response = test_client.get(f"/restaurants/{restaurant_id}")
    
    assert response.status_code == 200
    data = response.json()
    assert data["name"] == "Burger Barn"
    assert data["id"] == restaurant_id

def test_get_nonexistent_restaurant_returns_404(test_client):
    """Test retrieving non-existent restaurant returns 404"""
    response = test_client.get("/restaurants/99999")
    
    assert response.status_code == 404
    assert "not found" in response.json()["detail"].lower()

def test_list_restaurants_with_pagination(test_client):
    """Test listing restaurants with pagination"""
    # Create multiple restaurants
    for i in range(5):
        test_client.post("/restaurants", json={
            "name": f"Restaurant {i}",
            "cuisine": "Various",
            "address": f"{i} Street",
            "phone": f"+1-555-{i:04d}"
        })
    
    response = test_client.get("/restaurants?page=1&limit=3")
    
    assert response.status_code == 200
    data = response.json()
    assert len(data["items"]) == 3
    assert data["total"] >= 5
    assert data["page"] == 1
```

### Order Service - Component Tests with Mocked External Services (Node.js)

```javascript
// tests/component/orderService.test.js
const request = require('supertest');
const nock = require('nock');
const { startTestApp, stopTestApp } = require('../helpers/testApp');

describe('Order Service Component Tests', () => {
  let app;
  
  beforeAll(async () => {
    app = await startTestApp(); // Starts service with real DB (Testcontainers)
  });
  
  afterAll(async () => {
    await stopTestApp();
  });
  
  beforeEach(() => {
    // Mock Restaurant Service
    nock('http://restaurant-service:8080')
      .get('/restaurants/rest123')
      .reply(200, {
        id: 'rest123',
        name: 'Pizza Palace',
        isOpen: true
      });
    
    // Mock User Service
    nock('http://user-service:8080')
      .get('/users/user456')
      .reply(200, {
        id: 'user456',
        email: 'test@example.com',
        verified: true
      });
    
    // Mock Payment Gateway (Stripe Test Mode)
    nock('https://api.stripe.com')
      .post('/v1/payment_intents')
      .reply(200, {
        id: 'pi_test123',
        status: 'succeeded'
      });
  });
  
  afterEach(() => {
    nock.cleanAll();
  });
  
  test('POST /orders - creates order successfully', async () => {
    const orderData = {
      userId: 'user456',
      restaurantId: 'rest123',
      items: [
        { menuItemId: 'item1', quantity: 2, price: 15.99 }
      ],
      totalAmount: 31.98,
      paymentMethod: 'card'
    };
    
    const response = await request(app)
      .post('/orders')
      .send(orderData)
      .expect(201);
    
    expect(response.body).toMatchObject({
      userId: 'user456',
      restaurantId: 'rest123',
      status: 'confirmed',
      totalAmount: 31.98
    });
    expect(response.body).toHaveProperty('id');
    expect(response.body).toHaveProperty('createdAt');
  });
  
  test('POST /orders - rejects order when restaurant is closed', async () => {
    nock.cleanAll();
    nock('http://restaurant-service:8080')
      .get('/restaurants/rest123')
      .reply(200, {
        id: 'rest123',
        name: 'Pizza Palace',
        isOpen: false // Restaurant closed
      });
    
    const orderData = {
      userId: 'user456',
      restaurantId: 'rest123',
      items: [{ menuItemId: 'item1', quantity: 1, price: 10.00 }],
      totalAmount: 10.00
    };
    
    const response = await request(app)
      .post('/orders')
      .send(orderData)
      .expect(400);
    
    expect(response.body.error).toContain('Restaurant is currently closed');
  });
});
```

---

## 3. Contract Test Examples (Pact)

### Consumer Contract - Order Service expects Restaurant Service API

```javascript
// tests/contract/consumer/restaurantServiceContract.test.js
const { Pact } = require('@pact-foundation/pact');
const path = require('path');
const { getRestaurantDetails } = require('../../../src/clients/restaurantClient');

describe('Order Service -> Restaurant Service Contract', () => {
  const provider = new Pact({
    consumer: 'OrderService',
    provider: 'RestaurantService',
    port: 8989,
    log: path.resolve(process.cwd(), 'logs', 'pact.log'),
    dir: path.resolve(process.cwd(), 'pacts'),
    logLevel: 'warn'
  });
  
  beforeAll(() => provider.setup());
  afterEach(() => provider.verify());
  afterAll(() => provider.finalize());
  
  describe('GET /restaurants/:id', () => {
    test('when restaurant exists', async () => {
      await provider.addInteraction({
        state: 'restaurant with ID rest123 exists',
        uponReceiving: 'a request for restaurant rest123',
        withRequest: {
          method: 'GET',
          path: '/restaurants/rest123',
          headers: {
            'Accept': 'application/json'
          }
        },
        willRespondWith: {
          status: 200,
          headers: {
            'Content-Type': 'application/json'
          },
          body: {
            id: 'rest123',
            name: 'Pizza Palace',
            cuisine: 'Italian',
            isOpen: true,
            averageRating: 4.5
          }
        }
      });
      
      // Make actual request using our client
      const restaurant = await getRestaurantDetails('rest123', 'http://localhost:8989');
      
      expect(restaurant.id).toBe('rest123');
      expect(restaurant.name).toBe('Pizza Palace');
      expect(restaurant.isOpen).toBe(true);
    });
    
    test('when restaurant does not exist', async () => {
      await provider.addInteraction({
        state: 'restaurant with ID rest999 does not exist',
        uponReceiving: 'a request for non-existent restaurant rest999',
        withRequest: {
          method: 'GET',
          path: '/restaurants/rest999',
          headers: {
            'Accept': 'application/json'
          }
        },
        willRespondWith: {
          status: 404,
          headers: {
            'Content-Type': 'application/json'
          },
          body: {
            error: 'Restaurant not found'
          }
        }
      });
      
      await expect(
        getRestaurantDetails('rest999', 'http://localhost:8989')
      ).rejects.toThrow('Restaurant not found');
    });
  });
});
```

### Provider Verification - Restaurant Service verifies contracts

```javascript
// tests/contract/provider/restaurantServiceVerification.test.js
const { Verifier } = require('@pact-foundation/pact');
const path = require('path');
const { startServer, stopServer } = require('../../../src/server');

describe('Restaurant Service Provider Verification', () => {
  let server;
  const PORT = 8080;
  
  beforeAll(async () => {
    server = await startServer(PORT);
  });
  
  afterAll(async () => {
    await stopServer(server);
  });
  
  test('validates contracts from Pact Broker', async () => {
    const options = {
      provider: 'RestaurantService',
      providerBaseUrl: `http://localhost:${PORT}`,
      pactBrokerUrl: process.env.PACT_BROKER_URL || 'http://pact-broker:9292',
      pactBrokerToken: process.env.PACT_BROKER_TOKEN,
      publishVerificationResult: true,
      providerVersion: process.env.GIT_COMMIT || '1.0.0',
      
      // State handlers - set up test data for specific states
      stateHandlers: {
        'restaurant with ID rest123 exists': async () => {
          // Set up test data in database
          await createTestRestaurant({
            id: 'rest123',
            name: 'Pizza Palace',
            cuisine: 'Italian',
            isOpen: true,
            averageRating: 4.5
          });
        },
        'restaurant with ID rest999 does not exist': async () => {
          // Ensure restaurant doesn't exist
          await deleteRestaurantIfExists('rest999');
        }
      }
    };
    
    const verifier = new Verifier(options);
    await verifier.verifyProvider();
  });
});
```

---

## 4. Integration Test Examples

### Multi-Service Integration Test (Docker Compose)

```python
# tests/integration/test_order_flow.py
import pytest
import requests
import time
from kafka import KafkaConsumer
import json

@pytest.fixture(scope="module")
def services():
    """
    Assumes docker-compose has started:
    - User Service
    - Restaurant Service
    - Order Service
    - Kafka
    - PostgreSQL instances
    """
    # Wait for services to be ready
    wait_for_service("http://localhost:8081/health")  # User Service
    wait_for_service("http://localhost:8082/health")  # Restaurant Service
    wait_for_service("http://localhost:8083/health")  # Order Service
    yield
    # Cleanup happens in docker-compose down

def test_complete_order_flow(services):
    """Test complete flow: User -> Restaurant -> Order -> Event"""
    
    # 1. Create user
    user_response = requests.post("http://localhost:8081/users", json={
        "email": "customer@example.com",
        "name": "John Doe",
        "password": "SecurePass123!"
    })
    assert user_response.status_code == 201
    user_id = user_response.json()["id"]
    
    # 2. Create restaurant
    restaurant_response = requests.post("http://localhost:8082/restaurants", json={
        "name": "Test Restaurant",
        "cuisine": "Italian",
        "isOpen": True
    })
    assert restaurant_response.status_code == 201
    restaurant_id = restaurant_response.json()["id"]
    
    # 3. Create order
    order_response = requests.post("http://localhost:8083/orders", json={
        "userId": user_id,
        "restaurantId": restaurant_id,
        "items": [
            {"menuItemId": "item1", "quantity": 2, "price": 15.99}
        ],
        "totalAmount": 31.98
    })
    assert order_response.status_code == 201
    order_data = order_response.json()
    order_id = order_data["id"]
    assert order_data["status"] == "confirmed"
    
    # 4. Verify Kafka event was published
    consumer = KafkaConsumer(
        'order-events',
        bootstrap_servers=['localhost:9092'],
        auto_offset_reset='earliest',
        value_deserializer=lambda m: json.loads(m.decode('utf-8'))
    )
    
    # Poll for events (with timeout)
    event_found = False
    timeout = time.time() + 10  # 10 second timeout
    
    for message in consumer:
        event = message.value
        if event.get('orderId') == order_id and event.get('eventType') == 'ORDER_PLACED':
            event_found = True
            assert event['userId'] == user_id
            assert event['restaurantId'] == restaurant_id
            assert event['totalAmount'] == 31.98
            break
        
        if time.time() > timeout:
            break
    
    consumer.close()
    assert event_found, "ORDER_PLACED event not found in Kafka"

def test_order_with_invalid_restaurant_fails(services):
    """Test order fails gracefully with invalid restaurant"""
    
    order_response = requests.post("http://localhost:8083/orders", json={
        "userId": "user123",
        "restaurantId": "nonexistent",
        "items": [{"menuItemId": "item1", "quantity": 1, "price": 10.00}],
        "totalAmount": 10.00
    })
    
    assert order_response.status_code == 400
    assert "restaurant" in order_response.json()["error"].lower()
```

---

## 5. End-to-End Test Examples (Playwright)

```javascript
// tests/e2e/orderFlow.spec.js
const { test, expect } = require('@playwright/test');

test.describe('Complete Order Journey', () => {
  test('user can register, login, browse menu, and place order', async ({ page }) => {
    // 1. Register new user
    await page.goto('http://localhost:3000/register');
    await page.fill('[data-testid="email-input"]', 'test@example.com');
    await page.fill('[data-testid="password-input"]', 'SecurePass123!');
    await page.fill('[data-testid="name-input"]', 'Test User');
    await page.click('[data-testid="register-button"]');
    
    await expect(page).toHaveURL(/.*login/);
    
    // 2. Login
    await page.fill('[data-testid="email-input"]', 'test@example.com');
    await page.fill('[data-testid="password-input"]', 'SecurePass123!');
    await page.click('[data-testid="login-button"]');
    
    await expect(page).toHaveURL(/.*dashboard/);
    
    // 3. Browse restaurants
    await page.click('[data-testid="restaurants-link"]');
    await expect(page.locator('[data-testid="restaurant-card"]')).toHaveCount({ minimum: 1 });
    
    // 4. Select a restaurant
    await page.click('[data-testid="restaurant-card"]:first-child');
    await expect(page.locator('[data-testid="menu-items"]')).toBeVisible();
    
    // 5. Add items to cart
    await page.click('[data-testid="menu-item-add"]:first-child');
    await page.click('[data-testid="menu-item-add"]:first-child'); // Add 2
    
    await expect(page.locator('[data-testid="cart-count"]')).toHaveText('2');
    
    // 6. View cart
    await page.click('[data-testid="cart-button"]');
    await expect(page.locator('[data-testid="cart-item"]')).toHaveCount(1);
    
    // 7. Checkout
    await page.click('[data-testid="checkout-button"]');
    
    // 8. Enter payment details (test mode)
    await page.fill('[data-testid="card-number"]', '4242424242424242');
    await page.fill('[data-testid="card-expiry"]', '12/25');
    await page.fill('[data-testid="card-cvc"]', '123');
    
    // 9. Place order
    await page.click('[data-testid="place-order-button"]');
    
    // 10. Verify order confirmation
    await expect(page.locator('[data-testid="order-success"]')).toBeVisible();
    await expect(page.locator('[data-testid="order-id"]')).toContainText(/ORD-/);
    
    // 11. Verify notification (check for toast/email confirmation message)
    await expect(page.locator('[data-testid="notification-toast"]')).toContainText('Order confirmed');
  });
  
  test('user sees error when ordering from closed restaurant', async ({ page }) => {
    await page.goto('http://localhost:3000/login');
    await page.fill('[data-testid="email-input"]', 'test@example.com');
    await page.fill('[data-testid="password-input"]', 'SecurePass123!');
    await page.click('[data-testid="login-button"]');
    
    // Navigate to closed restaurant (set up via test data)
    await page.goto('http://localhost:3000/restaurants/closed-restaurant-id');
    
    await expect(page.locator('[data-testid="restaurant-closed-banner"]')).toBeVisible();
    await expect(page.locator('[data-testid="checkout-button"]')).toBeDisabled();
  });
});
```

---

## 6. Performance Test Examples (K6)

```javascript
// tests/performance/orderServiceLoad.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('errors');

// Test configuration
export const options = {
  stages: [
    { duration: '2m', target: 100 },   // Ramp up to 100 users
    { duration: '5m', target: 100 },   // Stay at 100 users
    { duration: '2m', target: 200 },   // Ramp up to 200 users
    { duration: '5m', target: 200 },   // Stay at 200 users
    { duration: '2m', target: 0 },     // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'], // 95% < 500ms, 99% < 1s
    http_req_failed: ['rate<0.01'],                  // Error rate < 1%
    errors: ['rate<0.05'],                           // Custom error rate < 5%
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:8083';

export function setup() {
  // Create test data (users, restaurants)
  const user = http.post(`${BASE_URL}/test/users`, JSON.stringify({
    email: `loadtest-${Date.now()}@example.com`,
    password: 'TestPass123!'
  }), {
    headers: { 'Content-Type': 'application/json' }
  });
  
  return {
    userId: JSON.parse(user.body).id,
    restaurantId: 'test-restaurant-123'
  };
}

export default function (data) {
  // Simulate placing an order
  const orderPayload = JSON.stringify({
    userId: data.userId,
    restaurantId: data.restaurantId,
    items: [
      { menuItemId: 'item1', quantity: 2, price: 15.99 },
      { menuItemId: 'item2', quantity: 1, price: 8.50 }
    ],
    totalAmount: 40.48
  });
  
  const params = {
    headers: { 'Content-Type': 'application/json' },
    tags: { name: 'CreateOrder' }
  };
  
  const response = http.post(`${BASE_URL}/orders`, orderPayload, params);
  
  // Validate response
  const success = check(response, {
    'status is 201': (r) => r.status === 201,
    'order has ID': (r) => JSON.parse(r.body).id !== undefined,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });
  
  errorRate.add(!success);
  
  // Think time (user delay between actions)
  sleep(Math.random() * 3 + 1); // 1-4 seconds
}

export function teardown(data) {
  // Cleanup test data if needed
}
```

---

## 7. Security Test Examples

### OWASP ZAP Automated Scan (Python)

```python
# tests/security/zap_scan.py
from zapv2 import ZAPv2
import time
import sys

# ZAP Proxy configuration
ZAP_URL = 'http://localhost:8090'
API_KEY = 'your-zap-api-key'
TARGET = 'http://localhost:8083'  # Order Service

zap = ZAPv2(apikey=API_KEY, proxies={'http': ZAP_URL, 'https': ZAP_URL})

def run_security_scan():
    print(f'Starting security scan on {TARGET}')
    
    # Spider the target
    print('Spidering target...')
    scan_id = zap.spider.scan(TARGET)
    
    while int(zap.spider.status(scan_id)) < 100:
        print(f'Spider progress: {zap.spider.status(scan_id)}%')
        time.sleep(2)
    
    print('Spider completed')
    
    # Active scan
    print('Starting active scan...')
    scan_id = zap.ascan.scan(TARGET)
    
    while int(zap.ascan.status(scan_id)) < 100:
        print(f'Active scan progress: {zap.ascan.status(scan_id)}%')
        time.sleep(5)
    
    print('Active scan completed')
    
    # Get alerts
    alerts = zap.core.alerts(baseurl=TARGET)
    
    # Categorize by risk
    high_risk = [a for a in alerts if a['risk'] == 'High']
    medium_risk = [a for a in alerts if a['risk'] == 'Medium']
    low_risk = [a for a in alerts if a['risk'] == 'Low']
    
    print(f'\n=== Security Scan Results ===')
    print(f'High Risk: {len(high_risk)}')
    print(f'Medium Risk: {len(medium_risk)}')
    print(f'Low Risk: {len(low_risk)}')
    
    # Print high-risk issues
    if high_risk:
        print('\nHigh Risk Issues:')
        for alert in high_risk:
            print(f"  - {alert['alert']}: {alert['url']}")
    
    # Generate HTML report
    with open('zap-report.html', 'w') as f:
        f.write(zap.core.htmlreport())
    
    print('\nReport saved to zap-report.html')
    
    # Fail build if high-risk issues found
    if high_risk:
        sys.exit(1)

if __name__ == '__main__':
    run_security_scan()
```

---

## Test Execution Commands

### Local Development
```bash
# Unit tests
npm test                          # Node.js
pytest tests/unit                 # Python

# Component tests
npm run test:component
pytest tests/component

# Contract tests
npm run test:pact
npm run pact:publish

# Watch mode
npm test -- --watch
pytest-watch
```

### CI Pipeline
```bash
# Run all tests with coverage
npm test -- --coverage --ci
pytest tests/ --cov=src --cov-report=xml

# Run integration tests
docker-compose up -d
npm run test:integration
docker-compose down

# Performance tests
k6 run tests/performance/orderServiceLoad.js

# Security scans
python tests/security/zap_scan.py
snyk test
trivy image order-service:latest
```

### E2E Tests
```bash
# Playwright
npx playwright test
npx playwright test --headed  # With browser UI
npx playwright show-report    # View HTML report

# Cypress
npx cypress run
npx cypress open  # Interactive mode
