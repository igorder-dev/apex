# /new-bot — Scaffold a New Trading Strategy Bot

When the user runs `/new-bot $ARGUMENTS`, scaffold a complete bot module.

Extract `{strategy_name}` from the arguments (e.g., "market_maker", "copy_trader"). Convert to snake_case for directories/files, PascalCase for class names.

## Steps

1. **Read the BaseBot interface**: Read `docs/04-technical-specification.md` → find the BaseBot interface section. Identify all 10 required lifecycle methods and their signatures.

2. **Read the reference implementation**: Read `src/bots/binary_arbitrage/` directory to understand the expected structure:
   - `__init__.py`
   - `bot.py` (lifecycle glue — delegates to strategy)
   - `strategy.py` (core algorithm)
   - `models.py` (strategy-specific Pydantic models)

3. **Create the bot directory**: `src/bots/{strategy_name}/`

4. **Create files**:
   - `src/bots/{strategy_name}/__init__.py` — empty, just makes it a package
   - `src/bots/{strategy_name}/bot.py` — class `{StrategyName}Bot(BaseBot)` implementing all 10 lifecycle methods. Each method should delegate to the strategy where appropriate. Include proper type hints and docstrings.
   - `src/bots/{strategy_name}/strategy.py` — placeholder strategy class with `analyze()` and `generate_signal()` methods. Include TODO comments marking where the user needs to implement logic.
   - `src/bots/{strategy_name}/models.py` — strategy-specific Pydantic V2 models (signal model, config model). Import from `src.core.models` for shared types.

5. **Create config**: `config/bots/{strategy_name}-default.yaml` with:
   ```yaml
   bot_id: "{strategy_name}_01"
   strategy: "{strategy_name}"
   enabled: true
   paper_trading: true
   graduated: false
   markets: []  # Add token_ids here
   risk:
     max_position_usdc: 100
     max_daily_loss_usdc: 50
     max_drawdown_pct: 15.0
     max_loss_per_trade_usdc: 10
   ```

6. **Create unit test**: `tests/unit/test_{strategy_name}_strategy.py` with:
   - Test class structure matching the testing strategy doc
   - Placeholder test methods for signal generation, edge cases
   - Fixture for realistic order book data (reference binary_arbitrage tests)

7. **Validate**: Run `ruff check src/bots/{strategy_name}/` and `mypy src/bots/{strategy_name}/` to ensure generated code passes linting.

8. **Report**: Summarize what was created and what the user needs to implement next.

## Important Rules
- The bot MUST implement ALL BaseBot methods — the orchestrator validates this at load time
- Strategy logic goes in `strategy.py`, NOT in `bot.py` (bot.py is lifecycle glue only)
- Never import from `src/services/` directly — use the injected BotContext
- Config MUST have `paper_trading: true` and `graduated: false` — bots always start in paper mode
