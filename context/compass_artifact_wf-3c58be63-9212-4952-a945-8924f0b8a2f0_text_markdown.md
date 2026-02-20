# AI and automated trading on Polymarket

**Automated bots now dominate Polymarket, extracting an estimated $40 million in arbitrage profits between April 2024 and April 2025, yet only 0.51% of all users have earned more than $1,000.** The prediction market platform — which processed $3.7 billion in 2024 U.S. election bets alone — has become a sophisticated battleground where AI agents, arbitrage bots, and market-making algorithms compete for edge against retail traders. A growing open-source ecosystem (led by Polymarket's own `agents` framework with 1,700+ GitHub stars) has lowered the barrier to entry, but the combination of compressed spreads, oracle manipulation risk, and an evolving regulatory landscape means profitability is far from guaranteed.

---

## Seven dominant strategies define the bot landscape

Automated Polymarket trading falls into distinct strategy categories, each exploiting different market inefficiencies. Understanding these approaches is essential before building anything.

**1. Intra-market arbitrage (sum-to-one).** In binary markets, YES + NO should equal $1.00 at resolution. Due to separate order books and asynchronous updates, combined prices sometimes dip below $1.00. Bots monitor the WebSocket feed and submit simultaneous Fill-or-Kill orders on both sides, locking in guaranteed profit. The spread must exceed **~2.5–3%** after Polymarket's 2% winner fee to be profitable. These opportunities typically last only seconds.

**2. Cross-platform arbitrage.** Different prediction platforms (Polymarket, Kalshi, Robinhood) price identical events differently. Bots compare prices across platforms, buying YES on one and NO on another when combined cost falls below $1.00. Tools like EventArb and OddPool aggregate cross-venue odds in real time. The key caveat: trades aren't atomic across platforms, so one leg can fail while the other executes.

**3. Latency/temporal arbitrage.** This exploits the lag between confirmed spot prices on exchanges (Binance, Coinbase) and Polymarket's 15-minute crypto markets. One bot famously turned **$313 into $438,000 in a single month** by detecting confirmed directional momentum in BTC spot markets and buying the corresponding side on Polymarket before prices adjusted. Win rates of **98%** have been documented for sophisticated implementations.

**4. Market making.** Bots place limit orders on both sides of a market, profiting from the bid-ask spread plus Polymarket's liquidity rewards program. Developer @defiance_cr open-sourced a market-making bot that earned **$700–$800/day** at peak on $10,000 starting capital, analyzing historical volatility across multiple timeframes to find low-volatility, high-reward markets. His code is available at `github.com/warproxxx/poly-maker`.

**5. AI/ML ensemble probability models.** These systems build probabilistic models from news, social data, polling, and statistical indicators, then compare estimated "true" probabilities to market prices. One documented bot generated **$2.2 million in two months** using ensemble models that continuously retrained on fresh data. This approach requires full ML infrastructure — data pipelines, model training, real-time monitoring — and is described as "not a side hustle but a bona fide startup."

**6. LLM-powered agents.** Large language models (GPT-4, Claude, etc.) evaluate prediction market questions, search for relevant information, estimate probabilities, and execute trades autonomously. Polymarket's official `agents` framework provides a ready-made architecture combining LLMs with ChromaDB for news vectorization, the Gamma API for market discovery, and the CLOB API for execution. Academic research shows frontier LLMs now match average human forecasters but still **significantly underperform superforecasters** (Brier scores of ~0.13 vs 0.02).

**7. News-driven rapid trading and sentiment analysis.** Bots monitor Twitter/X, Reuters, Telegram, and news APIs using NLP to detect sentiment shifts, then build positions before markets fully adjust. During breaking news events, **30–60 second windows** exist where prices deviate from fair value. A top-0.01% Polymarket trader with $850K in historical profit used this discretionary, news-driven approach across politics, macro, and gaming markets.

---

## How bots interact with Polymarket's technical infrastructure

Polymarket's CLOB is a **hybrid-decentralized** system: an off-chain operator handles order matching, while settlement occurs on-chain on Polygon via EIP-712 signed messages. The exchange uses a custom CTF (Conditional Token Framework) Exchange contract that facilitates atomic swaps between binary outcome tokens (ERC-1155) and USDC collateral.

The REST API at `https://clob.polymarket.com` provides unauthenticated endpoints for market data (order books, prices, midpoints, price history) and authenticated endpoints for trading (place/cancel orders, check balances, retrieve trade history). Real-time data streams through WebSocket at `wss://ws-subscriptions-clob.polymarket.com/ws/`. Authentication uses three levels: Level 0 (public, no auth), Level 1 (wallet private key signing for API key derivation), and Level 2 (HMAC-based API key/secret/passphrase for trading).

A typical bot's data flow works as follows. Market discovery happens via the **Gamma API** (`gamma-api.polymarket.com`), which provides event metadata, market slugs, tags, and volume data. Price and order book data come from the **CLOB API** or WebSocket for real-time updates. The bot's strategy layer processes this data alongside external sources (news APIs, social media, spot exchange prices), then generates trade signals. Execution happens through the CLOB API's `POST /order` endpoint using signed EIP-712 order messages. The official Python client, `py-clob-client`, abstracts this entire workflow:

```python
from py_clob_client.client import ClobClient
from py_clob_client.clob_types import OrderArgs, OrderType
from py_clob_client.order_builder.constants import BUY

client = ClobClient(
    "https://clob.polymarket.com",
    key="<private-key>",
    chain_id=137,  # Polygon mainnet
    signature_type=0  # EOA wallet
)
client.set_api_creds(client.create_or_derive_api_creds())

# Place a GTC limit order: buy 100 shares at $0.50
order = client.create_order(OrderArgs(
    price=0.50, size=100.0, side=BUY, token_id="<token-id>"
))
resp = client.post_order(order, OrderType.GTC)
```

Three order types are supported: **GTC** (Good-Til-Cancelled, rests on book), **GTD** (Good-Til-Date, auto-expires), and **FOK** (Fill-or-Kill, used for market orders). Prices range from $0.01 to $0.99, representing 1–99% implied probability. Tick sizes are dynamic, shifting to finer granularity at extreme prices (above $0.96 or below $0.04). Rate limits are generous: **3,500 order submissions per 10 seconds** burst, sustained at 60/second. Most markets currently charge **zero maker and taker fees**, though 15-minute crypto markets charge up to 3.15% taker fees, and a **2% fee on net winnings** applies at withdrawal.

---

## Five concrete strategy examples anyone can understand

**Example 1 — "Buy both sides cheap."** A bot monitors BTC 15-minute up/down markets via WebSocket. When temporary order book imbalances push YES to $0.52 and NO to $0.45 (total $0.97), the bot buys both sides for $0.97. One side pays $1.00 at resolution, netting $0.03 per dollar risked — roughly **3% guaranteed return every 15 minutes**. The documented "Gabagool" bot earned ~$58 profit per 15-minute window doing exactly this.

**Example 2 — "Cross-platform price mismatch."** Polymarket shows "Will Bitcoin exceed $100K by March?" at YES = $0.72. Kalshi shows the same event at NO = $0.22. Buying YES on Polymarket ($0.72) and NO on Kalshi ($0.22) costs $0.94 total. One side pays $1.00 regardless of outcome, yielding a **6% risk-free spread**. The bot polls both APIs every few seconds and executes when spreads appear.

**Example 3 — "Fast news reaction."** A bot monitors a Twitter firehose and news APIs for keywords related to active Polymarket markets (e.g., candidate names, geopolitical events). When sentiment for a presidential candidate spikes positive with high volume, the bot buys YES shares before the broader market reacts. The 30–60 second reaction window before market adjustment is the edge.

**Example 4 — "Spread the book, earn the gap."** A market-making bot places a BUY order at $0.48 and a SELL order at $0.52 on a political market. When both fill, the bot earns the **$0.04 spread** regardless of outcome. It continuously adjusts quotes based on volatility calculations across 3-hour, 24-hour, and 7-day windows, widening spreads in volatile markets and tightening in stable ones.

**Example 5 — "Ask the AI."** Using Polymarket's official `agents` framework, an LLM agent fetches active markets from the Gamma API, retrieves recent news articles into a ChromaDB vector store, prompts GPT-4 to estimate the probability of each event, compares its estimate to market prices, and buys shares wherever the model detects >10% mispricing. It executes trades automatically via the CLOB API with position sizing based on the Kelly criterion.

---

## What you need to build an automated Polymarket system

The technical stack for building a Polymarket bot includes several essential components. **Python** is the dominant language (used by most open-source bots), with TypeScript and Rust as alternatives. The core libraries include `py-clob-client` (official Python CLOB client, pip-installable), `polymarket-apis` (unified Pydantic-validated client covering CLOB, Gamma, and WebSocket APIs), and for AI agents, `langchain` with `chromadb` for RAG-based reasoning.

On the blockchain side, you need a **Polygon-compatible wallet** (an EOA with a private key), USDC on Polygon for collateral, and one-time token approvals for the CTF Exchange contract. When you first interact with Polymarket, a proxy wallet (1-of-1 Gnosis Safe) is deployed to hold your positions and collateral, enabling gasless trading through Polymarket's relayer.

The required infrastructure includes:

- **VPS hosting**: $10–80/month depending on strategy complexity. Low-latency VPS near Polygon nodes is essential for arbitrage (services like QuantVPS from $30/month)
- **Data sources**: Polymarket's APIs are free. External data costs vary — news APIs (NewsAPI, etc.) range from free tiers to $50+/month, social media APIs can be expensive
- **For AI/ML strategies**: GPU compute for model training ($50–500/month on cloud), vector database storage, LLM API costs (GPT-4 at ~$10–30/1M tokens)
- **Monitoring**: Grafana/Prometheus for production bots, Telegram/Discord webhooks for alerts

Key official SDKs and their GitHub URLs:

| SDK | URL |
|-----|-----|
| Python CLOB Client | https://github.com/Polymarket/py-clob-client |
| TypeScript CLOB Client | https://github.com/Polymarket/clob-client |
| Rust CLOB Client | https://github.com/Polymarket/rs-clob-client |
| AI Agents Framework | https://github.com/Polymarket/agents |
| CTF Exchange Contracts | https://github.com/Polymarket/ctf-exchange |
| Polymarket US Python SDK | https://github.com/Polymarket/polymarket-us-python |

The official documentation lives at https://docs.polymarket.com/developers/CLOB/introduction, with market data available through the Gamma API at `gamma-api.polymarket.com`. The Builders Program (https://docs.polymarket.com/developers/builders/builder-intro) has distributed over **$2.5 million** in grants and provides revenue-sharing tiers for serious developers.

---

## The pitfalls that catch most bot builders

**Oracle manipulation is the most severe systemic risk.** In March 2025, a UMA token holder used 5 million UMA tokens (25% of total votes) across three accounts to manipulate an oracle vote, falsely resolving a $7M market on Ukraine's mineral deal. Users who bet correctly lost everything. Polymarket confirmed the oracle reached the incorrect outcome but stated the result was final. UMA has since transitioned to a restricted proposer whitelist, but this vulnerability demonstrated that even "correct" bets can lose to governance attacks.

**Liquidity and slippage destroy theoretical profits.** Liquid markets (>$100K volume) have 1–3 cent spreads, but thin markets can show 5–10 cent spreads. A $500 order on a thin market can experience 2–5% slippage. Non-atomic execution means one leg of an arbitrage can fill while the other fails, creating dangerous "legged" positions. The popular "buy both sides" strategy in 15-minute crypto markets **fails catastrophically** during trending periods — if BTC trends strongly in one direction, one side progressively cheapens but then loses, wiping out accumulated gains.

**Regulatory risk is real and evolving.** Polymarket paid a **$1.4 million CFTC penalty** in 2022 for operating unregistered event contracts. While the DOJ and CFTC ended investigations in July 2025 and Polymarket has since acquired a CFTC-licensed exchange, U.S. users on the global platform risk **frozen funds** if detected using VPNs. Multiple states (Nevada, Tennessee, Massachusetts) are actively challenging prediction markets. Internationally, France, Belgium, Poland, and Singapore have banned or restricted access.

**Bot security is a real concern.** In January 2026, the Polycule Telegram trading bot was hacked, resulting in **~$230,000** in stolen user assets. The bot stored private keys server-side with SQL injection vulnerabilities. Never use custodial bot services or expose private keys to third-party platforms. A viral "Polymarket Arbitrager" bot was also documented as fundamentally flawed — it claimed to exploit YES+NO price discrepancies but market changes between order book checks led to money-losing trades.

Additional failure modes include API rate limit violations causing missed opportunities, standard Python order signing taking ~1 second (too slow for competitive HFT), model overfitting to historical patterns that don't repeat, survivorship bias in promoted strategies (the 92.4% of unprofitable wallets are never profiled), and the compressing spreads as institutional capital enters the space.

---

## Capital requirements and realistic return expectations

The practical minimum for automated trading is **$5,000–$10,000**. While Polymarket has no minimum deposit, transaction costs, spread requirements, and position sizing make sub-$1,000 accounts unviable for most strategies. The market-making bot developer @defiance_cr started with $10,000 and earned $200–$800/day. AI/ML-based strategies require additional infrastructure investment of $50–500/month for compute and data.

Realistic return profiles vary dramatically by strategy. Market making on low-volatility markets with active liquidity rewards has delivered **$200–$800/day on $10K+ capital**, though rewards decreased significantly after the 2024 election. Sum-to-one arbitrage yields **0.5–2% per trade** but opportunities last milliseconds and require speed infrastructure. Cross-platform arbitrage offers higher spreads but carries execution risk. The documented extreme successes ($313→$438K, $2.2M in two months) represent survivorship bias — the vast majority of automated traders lose money.

Monthly infrastructure costs range from **$10–$30** for a basic VPS running a single-market bot, to **$200–$500+** for multi-market strategies with AI/ML components, data feeds, and monitoring. Transaction costs on Polygon are minimal ($0.007–$0.50 per transaction), but the 2% winner fee and potential 3.15% taker fees on crypto markets eat into margins. Spreads need to exceed **2.5–3%** after all costs to remain profitable.

---

## GitHub repositories and open-source tools worth exploring

The open-source ecosystem spans official SDKs, community trading bots, AI agents, and analytics tools. Here are the most notable repositories, organized by category.

**Official Polymarket repositories:**
- **Polymarket/agents** (~1,700 stars) — The flagship AI agent framework combining LLMs, ChromaDB, and CLOB trading. MIT license. https://github.com/Polymarket/agents
- **Polymarket/py-clob-client** (~776 stars) — Official Python SDK for the CLOB API. https://github.com/Polymarket/py-clob-client
- **Polymarket/clob-client** (~445 stars) — Official TypeScript SDK. https://github.com/Polymarket/clob-client
- **Polymarket/rs-clob-client** (~486 stars) — Official Rust SDK. https://github.com/Polymarket/rs-clob-client
- **Polymarket/ctf-exchange** (~300 stars) — Smart contracts (Solidity). https://github.com/Polymarket/ctf-exchange

**Community trading bots:**
- **warproxxx/poly-maker** — The open-sourced market-making bot from @defiance_cr, using Google Sheets for config. https://github.com/warproxxx/poly-maker
- **gabagool222/15min-btc-polymarket-trading-bot** — BTC 15-minute arbitrage bot with depth-aware sizing and simulation mode. https://github.com/gabagool222/15min-btc-polymarket-trading-bot
- **ent0n29/polybot** — Enterprise-grade Java/Python platform with microservices (strategy, executor, ingestor), ClickHouse + Kafka, backtesting, Grafana monitoring. https://github.com/ent0n29/polybot
- **Trust412/Polymarket-spike-bot-v1** — HFT spike detection with risk controls and multi-threading. https://github.com/Trust412/Polymarket-spike-bot-v1
- **Trust412/polymarket-copy-trading-bot-version-3** — Copy-trading bot mirroring target wallets with 4-second polling. https://github.com/Trust412/polymarket-copy-trading-bot-version-3
- **lorine93s/polymarket-market-maker-bot** — Production market-making bot with inventory management and auto-redeem. https://github.com/lorine93s/polymarket-market-maker-bot
- **discountry/polymarket-trading-bot** — Beginner-friendly bot with gasless transactions via Builder Program. https://github.com/discountry/polymarket-trading-bot

**AI/LLM prediction agents:**
- **gnosis/prediction-market-agent** — Autonomous agents trading Omen, Manifold, Polymarket, and Metaculus with multiple agent types (microchain with LLM function calling, prophet_gpt4o). https://github.com/gnosis/prediction-market-agent
- **gnosis/prediction-market-agent-tooling** — Clean abstractions for benchmarking and deploying agents across platforms. https://github.com/gnosis/prediction-market-agent-tooling
- **ElAdrixHD/polymarket-agent** — Multi-LLM agent (Claude, GPT-4, Gemini, Ollama) with Kelly criterion sizing and natural language risk configuration. https://github.com/ElAdrixHD/polymarket-agent
- **llSourcell/Poly-Trader** — Educational AI agent by Siraj Raval using ChatGPT for market inefficiency detection. https://github.com/llSourcell/Poly-Trader
- **caiovicentino/polymarket-mcp-server** — MCP server enabling Claude Desktop to autonomously trade Polymarket with 45 tools. https://github.com/caiovicentino/polymarket-mcp-server
- **claudefi/claudefi** — Open-source Claude agent trading across DeFi and Polymarket with memory system and skill marketplace. https://github.com/claudefi/claudefi

**Arbitrage and analytics tools:**
- **ImMike/polymarket-arbitrage** — Cross-platform arbitrage (Polymarket/Kalshi) across 10,000+ markets with web dashboard and backtesting. https://github.com/ImMike/polymarket-arbitrage
- **CarlosIbCu/polymarket-kalshi-btc-arbitrage-bot** — BTC 1-hour market arbitrage between Polymarket and Kalshi with React frontend. https://github.com/CarlosIbCu/polymarket-kalshi-btc-arbitrage-bot
- **warproxxx/poly_data** — Comprehensive data pipeline for fetching and analyzing Polymarket trade data. https://github.com/warproxxx/poly_data
- **NYTEMODEONLY/polyterm** — Terminal-based monitoring with whale tracking, insider detection, and arbitrage scanning. https://github.com/NYTEMODEONLY/polyterm
- **aarora4/Awesome-Prediction-Market-Tools** — Curated directory of 170+ prediction market tools. https://github.com/aarora4/Awesome-Prediction-Market-Tools

**Third-party integrations:** NautilusTrader provides a full institutional-grade venue integration for Polymarket (`pip install nautilus_trader[polymarket]`), documented at https://nautilustrader.io/docs/latest/integrations/polymarket/.

---

## Community resources and where to learn more

The developer ecosystem centers on several hubs. **Polymarket's official Discord** has a #devs channel for technical questions. The **Builders Program** (https://builders.polymarket.com/) has distributed over $2.5 million in grants. Community Discords include **PolyToolz** (analyst insights, whale alerts), **PolyOdds** (daily signals, probability analysis), and **Betmoar** ($188M+ cumulative volume, professional terminal). On Reddit, **r/polymarket** serves as the main community hub.

Notable blog posts and analyses for deeper learning include the QuantVPS series on automated trading (https://www.quantvps.com/blog/automated-trading-polymarket), the @defiance_cr interview in Polymarket's Oracle newsletter (https://news.polymarket.com/p/automated-market-making-on-polymarket), the CoinsBench deep-dive on Gabagool's bot mechanics (https://coinsbench.com/inside-the-mind-of-a-polymarket-bot-3184e9481f0a), and the QuantJourney Substack analysis of Polymarket's fee curve (https://quantjourney.substack.com/p/understanding-the-polymarket-fee).

The most relevant academic work is the **IMDEA Networks study** (arXiv: 2508.03474) analyzing 86 million bets and documenting $40M in arbitrage extraction, the **AIA Forecaster** paper (arXiv: 2511.07678) showing LLM ensemble forecasts add value beyond market consensus, and Metaculus's ongoing AI benchmarking series which found AI bots are "closing the gap" with human superforecasters, with convergence estimated around **mid-2026**.

---

## Conclusion

Automated Polymarket trading is a legitimate but intensely competitive domain where the gap between theoretical edge and realized profit is wide. The technical infrastructure is mature — official SDKs in three languages, generous rate limits, and a rich open-source ecosystem lower the barrier to entry. But **profitability concentrates ruthlessly**: 3.7% of users (identified as bots) generate 37.44% of trading volume, while 92.4% of all wallets remain unprofitable.

The strategies with the most documented success are temporal arbitrage on 15-minute crypto markets (exploiting spot exchange lag), market making with liquidity rewards (though rewards have declined), and AI ensemble models for longer-term political and event markets. LLM-powered agents represent the fastest-growing approach, but academic evidence confirms they still lag human superforecasters. The oracle manipulation incident, regulatory fragmentation across U.S. states, and compressing spreads from institutional entry all suggest the easy arbitrage era is ending. For anyone entering this space, the most important investments are rigorous risk management, security hygiene around private key handling, and honest assessment of whether your edge is real or illusory.