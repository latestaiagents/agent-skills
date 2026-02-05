---
name: api-test-patterns
description: |
  Write comprehensive API tests for REST and GraphQL endpoints.
  Use this skill when testing APIs, writing contract tests, or validating integrations.
  Activate when: api testing, REST test, GraphQL test, endpoint testing, integration test, postman, contract testing.
---

# API Test Patterns

**Write comprehensive API tests that ensure reliability and contract compliance.**

## When to Use

- Testing REST or GraphQL APIs
- Validating API contracts
- Integration testing between services
- Testing error handling and edge cases
- Performance testing API endpoints

## REST API Testing

### Using Playwright API Testing

```typescript
// tests/api/users.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Users API', () => {
  const baseURL = process.env.API_URL || 'http://localhost:3000';

  test('GET /users returns list of users', async ({ request }) => {
    const response = await request.get(`${baseURL}/api/users`);

    expect(response.ok()).toBeTruthy();
    expect(response.status()).toBe(200);

    const users = await response.json();
    expect(Array.isArray(users)).toBeTruthy();
    expect(users.length).toBeGreaterThan(0);

    // Validate schema
    expect(users[0]).toMatchObject({
      id: expect.any(Number),
      email: expect.any(String),
      name: expect.any(String),
    });
  });

  test('POST /users creates new user', async ({ request }) => {
    const newUser = {
      email: 'test@example.com',
      name: 'Test User',
      password: 'securepassword123',
    };

    const response = await request.post(`${baseURL}/api/users`, {
      data: newUser,
    });

    expect(response.status()).toBe(201);

    const created = await response.json();
    expect(created.email).toBe(newUser.email);
    expect(created.name).toBe(newUser.name);
    expect(created).not.toHaveProperty('password');
  });

  test('GET /users/:id returns single user', async ({ request }) => {
    const response = await request.get(`${baseURL}/api/users/1`);

    expect(response.ok()).toBeTruthy();

    const user = await response.json();
    expect(user.id).toBe(1);
  });

  test('PUT /users/:id updates user', async ({ request }) => {
    const updates = { name: 'Updated Name' };

    const response = await request.put(`${baseURL}/api/users/1`, {
      data: updates,
    });

    expect(response.ok()).toBeTruthy();

    const user = await response.json();
    expect(user.name).toBe('Updated Name');
  });

  test('DELETE /users/:id removes user', async ({ request }) => {
    const response = await request.delete(`${baseURL}/api/users/1`);

    expect(response.status()).toBe(204);

    // Verify deleted
    const getResponse = await request.get(`${baseURL}/api/users/1`);
    expect(getResponse.status()).toBe(404);
  });
});
```

### Authentication Testing

```typescript
test.describe('Authentication', () => {
  test('requires auth for protected endpoints', async ({ request }) => {
    const response = await request.get(`${baseURL}/api/profile`);
    expect(response.status()).toBe(401);
  });

  test('accepts valid bearer token', async ({ request }) => {
    const loginResponse = await request.post(`${baseURL}/api/auth/login`, {
      data: { email: 'user@test.com', password: 'password' },
    });
    const { token } = await loginResponse.json();

    const response = await request.get(`${baseURL}/api/profile`, {
      headers: {
        Authorization: `Bearer ${token}`,
      },
    });

    expect(response.ok()).toBeTruthy();
  });

  test('rejects expired token', async ({ request }) => {
    const expiredToken = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...'; // expired

    const response = await request.get(`${baseURL}/api/profile`, {
      headers: {
        Authorization: `Bearer ${expiredToken}`,
      },
    });

    expect(response.status()).toBe(401);
    const body = await response.json();
    expect(body.error).toContain('expired');
  });
});
```

### Error Handling Tests

```typescript
test.describe('Error Handling', () => {
  test('returns 400 for invalid input', async ({ request }) => {
    const response = await request.post(`${baseURL}/api/users`, {
      data: { email: 'invalid-email' }, // missing required fields
    });

    expect(response.status()).toBe(400);

    const error = await response.json();
    expect(error).toMatchObject({
      error: expect.any(String),
      details: expect.any(Array),
    });
  });

  test('returns 404 for non-existent resource', async ({ request }) => {
    const response = await request.get(`${baseURL}/api/users/99999`);

    expect(response.status()).toBe(404);

    const error = await response.json();
    expect(error.error).toContain('not found');
  });

  test('returns 409 for duplicate resource', async ({ request }) => {
    // Create first user
    await request.post(`${baseURL}/api/users`, {
      data: { email: 'duplicate@test.com', name: 'User 1', password: 'pass' },
    });

    // Try to create duplicate
    const response = await request.post(`${baseURL}/api/users`, {
      data: { email: 'duplicate@test.com', name: 'User 2', password: 'pass' },
    });

    expect(response.status()).toBe(409);
  });

  test('returns 429 when rate limited', async ({ request }) => {
    const requests = Array(100).fill(null).map(() =>
      request.get(`${baseURL}/api/health`)
    );

    const responses = await Promise.all(requests);
    const rateLimited = responses.some(r => r.status() === 429);

    expect(rateLimited).toBeTruthy();
  });
});
```

## GraphQL Testing

```typescript
// tests/api/graphql.spec.ts
import { test, expect } from '@playwright/test';

test.describe('GraphQL API', () => {
  const graphqlURL = `${process.env.API_URL}/graphql`;

  async function graphqlRequest(request: any, query: string, variables = {}) {
    return request.post(graphqlURL, {
      data: { query, variables },
      headers: { 'Content-Type': 'application/json' },
    });
  }

  test('query users', async ({ request }) => {
    const query = `
      query GetUsers {
        users {
          id
          name
          email
        }
      }
    `;

    const response = await graphqlRequest(request, query);
    const { data, errors } = await response.json();

    expect(errors).toBeUndefined();
    expect(data.users).toBeInstanceOf(Array);
    expect(data.users[0]).toHaveProperty('id');
  });

  test('query with variables', async ({ request }) => {
    const query = `
      query GetUser($id: ID!) {
        user(id: $id) {
          id
          name
          email
        }
      }
    `;

    const response = await graphqlRequest(request, query, { id: '1' });
    const { data, errors } = await response.json();

    expect(errors).toBeUndefined();
    expect(data.user.id).toBe('1');
  });

  test('mutation creates resource', async ({ request }) => {
    const mutation = `
      mutation CreateUser($input: CreateUserInput!) {
        createUser(input: $input) {
          id
          name
          email
        }
      }
    `;

    const response = await graphqlRequest(request, mutation, {
      input: {
        name: 'New User',
        email: 'new@test.com',
        password: 'password123',
      },
    });

    const { data, errors } = await response.json();

    expect(errors).toBeUndefined();
    expect(data.createUser.email).toBe('new@test.com');
  });

  test('handles GraphQL errors', async ({ request }) => {
    const query = `
      query GetUser($id: ID!) {
        user(id: $id) {
          id
          name
        }
      }
    `;

    const response = await graphqlRequest(request, query, { id: '99999' });
    const { data, errors } = await response.json();

    expect(data.user).toBeNull();
    expect(errors).toBeDefined();
    expect(errors[0].message).toContain('not found');
  });
});
```

## Contract Testing with Pact

```typescript
// tests/contract/user-service.pact.ts
import { PactV3, MatchersV3 } from '@pact-foundation/pact';
import { resolve } from 'path';

const { like, eachLike, string, integer } = MatchersV3;

const provider = new PactV3({
  consumer: 'Frontend',
  provider: 'UserService',
  dir: resolve(process.cwd(), 'pacts'),
});

describe('User Service Contract', () => {
  it('returns users list', async () => {
    // Define expected interaction
    provider
      .given('users exist')
      .uponReceiving('a request for all users')
      .withRequest({
        method: 'GET',
        path: '/api/users',
      })
      .willRespondWith({
        status: 200,
        headers: { 'Content-Type': 'application/json' },
        body: eachLike({
          id: integer(1),
          name: string('John Doe'),
          email: string('john@example.com'),
        }),
      });

    // Execute test
    await provider.executeTest(async (mockserver) => {
      const response = await fetch(`${mockserver.url}/api/users`);
      const users = await response.json();

      expect(users).toHaveLength(1);
      expect(users[0]).toHaveProperty('id');
    });
  });
});
```

## API Test Checklist

```markdown
## Endpoint: [METHOD] /path

### Happy Path
- [ ] Returns correct status code
- [ ] Response body matches schema
- [ ] Required fields present
- [ ] Correct content-type header

### Authentication/Authorization
- [ ] Rejects unauthenticated requests
- [ ] Rejects unauthorized users
- [ ] Accepts valid credentials
- [ ] Handles token expiration

### Input Validation
- [ ] Rejects missing required fields
- [ ] Rejects invalid data types
- [ ] Rejects values outside constraints
- [ ] Handles empty strings/arrays

### Error Cases
- [ ] 400 for bad request
- [ ] 401 for unauthorized
- [ ] 403 for forbidden
- [ ] 404 for not found
- [ ] 409 for conflict
- [ ] 422 for validation errors
- [ ] 500 for server errors

### Edge Cases
- [ ] Empty results
- [ ] Pagination boundaries
- [ ] Special characters in input
- [ ] Maximum payload size
- [ ] Concurrent requests

### Performance
- [ ] Response time < threshold
- [ ] Handles expected load
- [ ] Rate limiting works
```

## Best Practices

1. **Test in isolation** - Use mocks for external dependencies
2. **Clean up test data** - Don't leave test data in shared environments
3. **Use fixtures** - Centralize test data creation
4. **Validate schemas** - Don't just check status codes
5. **Test error paths** - Errors are part of the contract
6. **Version your APIs** - Test each version separately
7. **Automate in CI** - Run API tests on every PR
