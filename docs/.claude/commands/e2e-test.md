# /e2e-test — Dashboard E2E Tests

When the user runs `/e2e-test $ARGUMENTS`, execute Playwright-based E2E tests against the running dashboard.

Extract optional `{suite_name}` from arguments. Valid suites: `dashboard`, `emergency-stop`, `config-change`, `all` (default).

## Prerequisites

- Dashboard must be running and accessible
- Playwright MCP must be available (connected as Claude Code plugin)

## Steps

1. **Read test specifications**: Read `docs/11-testing-strategy.md` → E2E Test Scenarios section for expected behavior.

2. **Verify dashboard is accessible**: Use Playwright MCP to navigate to the dashboard URL (default: `https://localhost/dashboard` or `http://localhost:8000` in dev). If the page doesn't load, STOP and report the issue.

3. **Run the specified test suite**:

### Suite: `dashboard`
- Navigate to main dashboard page
- Verify bot status cards render (check for bot names, status badges)
- Verify P&L charts render (Recharts components should be visible)
- Wait 5 seconds and verify SSE events update the UI (bot metrics should change)
- Navigate to each main page (Dashboard, Bots, Wallets, Settings)
- Take screenshot of each page

### Suite: `emergency-stop`
- Navigate to dashboard
- Locate the emergency stop button
- Click emergency stop (handle confirmation dialog if present)
- Wait for state update (up to 5 seconds)
- Verify all bot cards show STOPPED status
- Verify emergency banner appears
- Take screenshot of the stopped state

### Suite: `config-change`
- Navigate to bot configuration page
- Note current value of a bot parameter
- Change a parameter via the UI (e.g., toggle paper_trading)
- Wait for update confirmation
- Refresh the page and verify the change persists
- Take screenshot

### Suite: `all`
- Run all suites above in sequence

4. **Take screenshots** at each checkpoint using Playwright MCP screenshot tool.

5. **Report results**:
   ```
   E2E TEST RESULTS: {suite_name}
   ═══════════════════════════════
   [PASS] Dashboard loads successfully
   [PASS] Bot status cards render (3 bots visible)
   [PASS] P&L chart renders with data
   [FAIL] SSE events not updating — no metric changes after 5 seconds

   Screenshots saved: page-dashboard.png, page-bots.png, page-wallets.png

   Failures: 1
   Suggested investigation: Check if SSE endpoint is working (GET /api/v1/events/stream)
   ```

## Notes
- This command is READ-ONLY against the system except for the emergency-stop suite
- For emergency-stop suite, warn the user that it will actually trigger an emergency stop on the running system
- If dashboard auth is required, prompt the user for the API key
