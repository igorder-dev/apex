# /test-strategy — Run Strategy Unit Tests

When the user runs `/test-strategy $ARGUMENTS`, run the test suite for the specified bot strategy.

Extract `{bot_name}` from the arguments (e.g., "binary_arbitrage", "market_maker").

## Steps

1. **Read testing expectations**: Read `docs/11-testing-strategy.md` → Critical Unit Test Suites section to understand coverage requirements and test patterns.

2. **Verify test file exists**: Check that `tests/unit/test_{bot_name}_strategy.py` exists. If not, inform the user and suggest running `/new-bot` first.

3. **Run unit tests**:
   ```bash
   pytest tests/unit/test_{bot_name}_strategy.py -v --tb=long
   ```
   Report results clearly — show pass/fail counts and any failure details.

4. **If unit tests pass, run integration tests**:
   ```bash
   pytest tests/integration/ -k {bot_name} -v --tb=long
   ```
   If no integration tests match, note this as informational (not a failure).

5. **Run coverage report**:
   ```bash
   pytest tests/unit/test_{bot_name}_strategy.py --cov=src/bots/{bot_name} --cov-report=term-missing
   ```

6. **Compare against targets**: Coverage target for bot strategies is 70% minimum (from doc 11). Report whether the target is met.

7. **Summarize results**:
   ```
   TEST RESULTS: {bot_name}
   ════════════════════════
   Unit Tests:        12 passed, 0 failed
   Integration Tests: 3 passed, 0 failed
   Coverage:          78% (target: 70%) ✓

   Missing coverage:
     strategy.py: lines 45-52 (edge case: empty orderbook)
     strategy.py: lines 88-91 (partial fill handling)
   ```

## Notes
- If tests fail, show the full traceback and suggest fixes based on the error
- Don't modify test files — report issues and let the user decide on fixes
- If the bot directory doesn't exist at all, inform the user clearly
