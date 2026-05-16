```bash
npm install -D vitest @testing-library/react @testing-library/jest-dom jsdom @nx/vite
```


```bash
npx nx generate @nx/vite:vitest --project=finance-ui --uiFramework=react
```


Test a single module:
npx nx test finance-ui

Test everything in the BES:
npx nx run-many -t test

The Magic Command (Test only what changed):
npx nx affected -t test



You usually apply E2E tests to the Shell (the host app), because that is where all the modules come together.

Step 1: Install the Playwright Plugin

Bash
npm install -D @nx/playwright
Step 2: Inject Playwright into the Shell

Bash
npx nx generate @nx/playwright:configuration --project=shell
What this does: Nx will generate a shell-e2e folder at the root of your workspace. It sets up the configuration so that when you run the tests, it automatically boots up your Vite dev server, runs the browser tests, and then shuts the server down.

Step 3: Run the E2E Tests

Bash
npx nx e2e shell-e2e
By keeping the --unitTestRunner=none flag when we generated the folders earlier, you kept the workspace clean. When you are ready to enforce test coverage, you simply run these generator commands on a per-library basis, and Nx seamlessly weaves the testing framework into the monolith.