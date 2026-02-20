# /deploy — Build and Deploy to VPS

When the user runs `/deploy`, execute the full deployment workflow with safety gates.

## Steps

### Phase 1: Pre-flight Checks

1. **Read deployment spec**: Read `docs/09-infrastructure-spec.md` → Deployment Workflow section.

2. **Run all tests**:
   ```bash
   make test
   ```
   If tests fail, STOP and report failures. Do not proceed.

3. **Build frontend**:
   ```bash
   cd frontend && npm run build
   ```
   If build fails, STOP and report.

4. **Lint and typecheck**:
   ```bash
   ruff check src/
   mypy src/
   ```
   If either fails, STOP and report.

5. **Report pre-flight status**:
   ```
   PRE-FLIGHT CHECKS
   ══════════════════
   [PASS] All tests pass (42 unit, 8 integration)
   [PASS] Frontend builds successfully
   [PASS] Linting clean
   [PASS] Type checking clean
   ```

### Phase 2: Build

6. **Build Docker images**:
   ```bash
   docker compose build
   ```

7. **Show config diff** since last deploy:
   ```bash
   git diff HEAD~1 config/
   ```
   If there are config changes, highlight them clearly.

### Phase 3: Confirmation

8. **MANDATORY CHECKPOINT**: Present a deployment summary and WAIT for user confirmation:
   ```
   DEPLOYMENT SUMMARY
   ══════════════════
   Services to update: [list changed services]
   Config changes: [yes/no — detail if yes]
   Docker images rebuilt: [list]

   ⚠️  This will restart services on the VPS.
   ⚠️  Active bots will be stopped and restarted.

   Type "confirm" to proceed or "abort" to cancel.
   ```

   **NEVER proceed without explicit user confirmation.**

### Phase 4: Deploy

9. **Deploy**:
   ```bash
   docker compose up -d
   ```

10. **Post-deploy verification** (30-second window):
    - Tail logs: `docker compose logs -f --tail=50` (for ~30 seconds)
    - Check for error-level log entries
    - If Docker MCP available: verify all containers are healthy
    - If Playwright MCP available: navigate to dashboard URL and verify it loads

11. **Verify monitoring**: Check that Prometheus is scraping all targets (curl the Prometheus targets endpoint).

### Phase 5: Report

12. **Report deployment status**:
    ```
    DEPLOYMENT COMPLETE
    ═══════════════════
    [OK] All 11 containers running
    [OK] Health checks passing
    [OK] Dashboard accessible
    [OK] Prometheus scraping all targets
    [WARN] Bot "arb_01" restarting — check logs

    Rollback: docker compose down && git checkout HEAD~1 -- docker-compose.yml && docker compose up -d
    ```

## Safety Rules
- NEVER deploy without running tests first
- NEVER skip the user confirmation checkpoint
- If any post-deploy check fails, immediately present rollback instructions
- Keep the previous Docker images available for rollback
