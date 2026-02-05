---
name: playwright-patterns
description: |
  Write reliable, maintainable E2E tests with Playwright best practices.
  Use this skill when writing Playwright tests, debugging flaky tests, or setting up E2E automation.
  Activate when: playwright, e2e test, end-to-end, browser testing, UI automation, web testing.
---

# Playwright Patterns

**Write reliable, maintainable E2E tests with Playwright.**

## When to Use

- Setting up Playwright test framework
- Writing new E2E tests
- Debugging flaky tests
- Implementing Page Object Model
- Setting up CI/CD integration

## Project Setup

### Installation

```bash
# Create new project
npm init playwright@latest

# Or add to existing project
npm install -D @playwright/test
npx playwright install
```

### Configuration

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [
    ['html'],
    ['json', { outputFile: 'test-results/results.json' }],
    process.env.CI ? ['github'] : ['list']
  ],

  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },

  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
    {
      name: 'mobile-chrome',
      use: { ...devices['Pixel 5'] },
    },
  ],

  webServer: {
    command: 'npm run start',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

## Page Object Model

### Base Page

```typescript
// pages/BasePage.ts
import { Page, Locator } from '@playwright/test';

export class BasePage {
  readonly page: Page;

  constructor(page: Page) {
    this.page = page;
  }

  async navigate(path: string = '/') {
    await this.page.goto(path);
  }

  async waitForPageLoad() {
    await this.page.waitForLoadState('networkidle');
  }

  async getTitle(): Promise<string> {
    return this.page.title();
  }
}
```

### Example Page Object

```typescript
// pages/LoginPage.ts
import { Page, Locator, expect } from '@playwright/test';
import { BasePage } from './BasePage';

export class LoginPage extends BasePage {
  // Locators
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly loginButton: Locator;
  readonly errorMessage: Locator;
  readonly forgotPasswordLink: Locator;

  constructor(page: Page) {
    super(page);
    this.emailInput = page.getByLabel('Email');
    this.passwordInput = page.getByLabel('Password');
    this.loginButton = page.getByRole('button', { name: 'Sign in' });
    this.errorMessage = page.getByRole('alert');
    this.forgotPasswordLink = page.getByRole('link', { name: 'Forgot password?' });
  }

  async goto() {
    await this.navigate('/login');
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.loginButton.click();
  }

  async expectError(message: string) {
    await expect(this.errorMessage).toContainText(message);
  }

  async expectLoggedIn() {
    await expect(this.page).toHaveURL(/.*dashboard/);
  }
}
```

### Using Page Objects in Tests

```typescript
// tests/login.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from '../pages/LoginPage';

test.describe('Login', () => {
  let loginPage: LoginPage;

  test.beforeEach(async ({ page }) => {
    loginPage = new LoginPage(page);
    await loginPage.goto();
  });

  test('successful login', async () => {
    await loginPage.login('user@example.com', 'password123');
    await loginPage.expectLoggedIn();
  });

  test('shows error for invalid credentials', async () => {
    await loginPage.login('user@example.com', 'wrongpassword');
    await loginPage.expectError('Invalid credentials');
  });

  test('navigates to forgot password', async () => {
    await loginPage.forgotPasswordLink.click();
    await expect(loginPage.page).toHaveURL(/.*forgot-password/);
  });
});
```

## Locator Best Practices

### Priority Order (Best to Worst)

```typescript
// 1. User-facing attributes (BEST)
page.getByRole('button', { name: 'Submit' });
page.getByLabel('Email');
page.getByPlaceholder('Enter email');
page.getByText('Welcome');

// 2. Test IDs (Good for complex elements)
page.getByTestId('checkout-button');

// 3. CSS selectors (Avoid if possible)
page.locator('.btn-primary');
page.locator('#submit-form');

// 4. XPath (WORST - avoid)
page.locator('//button[@class="submit"]');
```

### Robust Locators

```typescript
// BAD - Fragile
page.locator('.sc-bdVaJa.bFDOgs'); // Generated class names
page.locator('div > div > button'); // Position-dependent
page.locator('[class*="Button"]'); // Partial class match

// GOOD - Resilient
page.getByRole('button', { name: /submit/i }); // Role + accessible name
page.getByLabel('Email address'); // Label association
page.getByTestId('submit-order'); // Explicit test ID
```

## Handling Async Operations

### Waiting Strategies

```typescript
// Wait for element
await page.waitForSelector('[data-testid="loaded"]');

// Wait for network
await page.waitForResponse(resp =>
  resp.url().includes('/api/data') && resp.status() === 200
);

// Wait for URL
await page.waitForURL('**/dashboard');

// Auto-waiting (built into actions)
await page.click('button'); // Waits automatically

// Custom wait
await expect(async () => {
  const count = await page.locator('.item').count();
  expect(count).toBeGreaterThan(5);
}).toPass({ timeout: 10000 });
```

### Handling Network

```typescript
// Wait for API response
const responsePromise = page.waitForResponse('/api/users');
await page.click('button#load-users');
const response = await responsePromise;
const data = await response.json();

// Mock API response
await page.route('/api/users', route => {
  route.fulfill({
    status: 200,
    body: JSON.stringify([{ id: 1, name: 'Test User' }]),
  });
});

// Block resources
await page.route('**/*.{png,jpg,jpeg}', route => route.abort());
```

## Test Fixtures

### Custom Fixtures

```typescript
// fixtures.ts
import { test as base } from '@playwright/test';
import { LoginPage } from './pages/LoginPage';
import { DashboardPage } from './pages/DashboardPage';

type MyFixtures = {
  loginPage: LoginPage;
  dashboardPage: DashboardPage;
  authenticatedPage: DashboardPage;
};

export const test = base.extend<MyFixtures>({
  loginPage: async ({ page }, use) => {
    await use(new LoginPage(page));
  },

  dashboardPage: async ({ page }, use) => {
    await use(new DashboardPage(page));
  },

  authenticatedPage: async ({ page }, use) => {
    // Login before test
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login('test@example.com', 'password');

    const dashboard = new DashboardPage(page);
    await use(dashboard);
  },
});

export { expect } from '@playwright/test';
```

### Using Fixtures

```typescript
// tests/dashboard.spec.ts
import { test, expect } from '../fixtures';

test('dashboard shows user data', async ({ authenticatedPage }) => {
  await expect(authenticatedPage.welcomeMessage).toBeVisible();
  await expect(authenticatedPage.userName).toHaveText('Test User');
});
```

## Visual Testing

```typescript
// Screenshot comparison
await expect(page).toHaveScreenshot('homepage.png');

// Element screenshot
await expect(page.locator('.chart')).toHaveScreenshot('chart.png');

// With options
await expect(page).toHaveScreenshot('full-page.png', {
  fullPage: true,
  maxDiffPixels: 100,
});

// Update snapshots
// npx playwright test --update-snapshots
```

## CI/CD Integration

### GitHub Actions

```yaml
# .github/workflows/playwright.yml
name: Playwright Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps

      - name: Run tests
        run: npx playwright test

      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30
```

## Debugging

```bash
# Debug mode (opens inspector)
npx playwright test --debug

# UI mode (interactive)
npx playwright test --ui

# Headed mode
npx playwright test --headed

# Specific browser
npx playwright test --project=chromium

# Generate code
npx playwright codegen localhost:3000
```

## Best Practices

1. **Use web-first assertions** - `expect(locator).toBeVisible()` not `isVisible()`
2. **Avoid hard waits** - Use auto-waiting and explicit conditions
3. **Isolate tests** - Each test should be independent
4. **Use Page Objects** - Centralize selectors and actions
5. **Test user flows** - Not implementation details
6. **Keep tests fast** - Parallelize, mock slow APIs
7. **Meaningful names** - Test names should describe behavior
