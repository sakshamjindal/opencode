# Flintstone: Jetson for Finance

## Vision

**Flintstone** is Jetson specialized for financial software development, analysis, and automation.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│                          ARCHITECTURE OPTIONS                                │
│                                                                              │
│   Option A: Plugin Layer          Option B: Separate Package                │
│   ─────────────────────           ──────────────────────────                │
│                                                                              │
│   ┌─────────────────────┐         ┌─────────────────────┐                   │
│   │     Flintstone      │         │     Flintstone      │                   │
│   │   (finance tools)   │         │   (full package)    │                   │
│   ├─────────────────────┤         ├─────────────────────┤                   │
│   │       Jetson        │         │   Jetson (import)   │                   │
│   │    (core engine)    │         │   + Finance Layer   │                   │
│   └─────────────────────┘         └─────────────────────┘                   │
│                                                                              │
│   • Jetson stays generic          • Flintstone is standalone                │
│   • Flintstone is a plugin        • Can diverge from Jetson                 │
│   • Easy to update both           • More flexibility                        │
│   • Single codebase               • Separate releases                       │
│                                                                              │
│   RECOMMENDED: Option A (Plugin Layer)                                      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Recommended Architecture: Plugin Layer

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│                             FLINTSTONE                                       │
│                         (Jetson + Finance)                                   │
│                                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   FINANCE LAYER (Flintstone-specific)                                        │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                                                                      │   │
│   │   Finance Tools                 Finance MCP Servers                  │   │
│   │   ├── market_data              ├── bloomberg-mcp                    │   │
│   │   ├── portfolio_analysis       ├── refinitiv-mcp                    │   │
│   │   ├── risk_calc                ├── yahoo-finance-mcp                │   │
│   │   ├── backtest                 └── internal-trading-mcp             │   │
│   │   ├── trade_execute                                                  │   │
│   │   └── compliance_check         Finance Skills                        │   │
│   │                                 ├── quant-analysis                   │   │
│   │   Finance Prompts               ├── risk-modeling                    │   │
│   │   ├── system prompt            ├── regulatory-compliance            │   │
│   │   ├── trading rules            └── portfolio-optimization           │   │
│   │   └── compliance context                                             │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                         │
│                                    │ extends                                 │
│                                    ▼                                         │
│   CORE LAYER (Jetson - unchanged)                                            │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                                                                      │   │
│   │   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐              │   │
│   │   │  Agent  │  │Provider │  │  Tools  │  │ Plugin  │              │   │
│   │   │ Engine  │  │ Factory │  │Registry │  │ System  │◄── Flintstone│   │
│   │   └─────────┘  └─────────┘  └─────────┘  └─────────┘   registers  │   │
│   │                                                          here      │   │
│   │   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐              │   │
│   │   │   MCP   │  │   LSP   │  │ Storage │  │  Event  │              │   │
│   │   │ Client  │  │ Client  │  │ (SQLite)│  │   Bus   │              │   │
│   │   └─────────┘  └─────────┘  └─────────┘  └─────────┘              │   │
│   │                                                                      │   │
│   │   Core Tools: read, write, edit, bash, glob, grep, task, etc.       │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Implementation Approach

### 1. Project Structure

```
flintstone/
├── pyproject.toml                # Depends on jetson
├── README.md
├── src/
│   └── flintstone/
│       │
│       ├── __init__.py           # from jetson import Jetson
│       ├── __main__.py           # Entry: flintstone CLI
│       ├── cli.py                # Wraps jetson CLI + finance commands
│       │
│       │   ╔═════════════════════════════════════════════════════════╗
│       │   ║              FINANCE-SPECIFIC EXTENSIONS                 ║
│       │   ╚═════════════════════════════════════════════════════════╝
│       │
│       ├── tools/                # Finance tools
│       │   ├── __init__.py
│       │   ├── market_data.py    # Fetch prices, quotes, etc.
│       │   ├── portfolio.py      # Portfolio analysis
│       │   ├── risk.py           # VaR, Greeks, etc.
│       │   ├── backtest.py       # Strategy backtesting
│       │   ├── trade.py          # Order execution
│       │   └── compliance.py     # Regulatory checks
│       │
│       ├── mcp/                  # Finance MCP server configs
│       │   ├── __init__.py
│       │   ├── bloomberg.py      # Bloomberg Terminal MCP
│       │   ├── refinitiv.py      # Refinitiv Eikon MCP
│       │   └── yahoo.py          # Yahoo Finance MCP
│       │
│       ├── skills/               # Finance skill prompts
│       │   ├── quant.md          # Quantitative analysis
│       │   ├── risk.md           # Risk modeling
│       │   └── compliance.md     # Regulatory compliance
│       │
│       ├── prompts/              # System prompts
│       │   └── system.py         # Finance-aware system prompt
│       │
│       ├── config/               # Default configs
│       │   ├── permissions.py    # Finance-specific permissions
│       │   └── defaults.py       # Default providers, settings
│       │
│       └── plugin.py             # Jetson plugin registration
│
└── tests/
    ├── test_market_data.py
    ├── test_portfolio.py
    └── test_risk.py
```

### 2. Dependencies

```toml
# flintstone/pyproject.toml

[project]
name = "flintstone"
version = "0.1.0"
description = "Jetson for Finance - AI coding agent for financial software"
requires-python = ">=3.11"
dependencies = [
    # Core: Jetson engine
    "jetson>=0.1.0",

    # Finance data
    "yfinance>=0.2.0",           # Yahoo Finance
    "pandas>=2.0.0",             # Data manipulation
    "numpy>=1.24.0",             # Numerical computing

    # Finance analytics
    "quantlib>=1.30",            # Derivatives pricing
    "pyfolio>=0.9.0",            # Portfolio analysis
    "empyrical>=0.5.0",          # Risk metrics

    # Optional: Premium data
    # "bloomberg-sdk",           # Bloomberg (licensed)
    # "refinitiv-data",          # Refinitiv (licensed)
]

[project.scripts]
flintstone = "flintstone.cli:app"
fstone = "flintstone.cli:app"    # Short alias
```

### 3. Plugin Registration

```python
# flintstone/src/flintstone/plugin.py

"""
Flintstone plugin for Jetson - registers finance tools and configurations.
"""

from jetson.plugins import Plugin
from .tools import (
    MarketDataTool,
    PortfolioTool,
    RiskTool,
    BacktestTool,
    TradeTool,
    ComplianceTool,
)
from .prompts.system import FINANCE_SYSTEM_PROMPT
from .config.permissions import FINANCE_PERMISSIONS


def create_plugin() -> Plugin:
    """Create the Flintstone plugin for Jetson."""

    return Plugin(
        name="flintstone",
        version="0.1.0",

        # Register finance tools
        tools=[
            MarketDataTool(),
            PortfolioTool(),
            RiskTool(),
            BacktestTool(),
            TradeTool(),
            ComplianceTool(),
        ],

        # Finance system prompt (prepended to Jetson's)
        system_prompt=FINANCE_SYSTEM_PROMPT,

        # Finance-specific permissions
        permissions=FINANCE_PERMISSIONS,

        # MCP servers to auto-configure
        mcp_servers={
            "yahoo-finance": {
                "command": ["python", "-m", "flintstone.mcp.yahoo"],
                "env": {}
            }
        },

        # Event hooks
        hooks={
            "tool.execute.before": validate_trade_compliance,
            "tool.execute.after": log_financial_action,
        }
    )


async def validate_trade_compliance(event):
    """Check compliance before trade execution."""
    if event.tool_name == "trade_execute":
        # Run compliance checks
        pass


async def log_financial_action(event):
    """Audit log for financial operations."""
    if event.tool_name in ["trade_execute", "portfolio_rebalance"]:
        # Log to audit trail
        pass
```

---

## Finance Tools

### market_data - Fetch Market Data

```python
# flintstone/src/flintstone/tools/market_data.py

from pydantic import BaseModel, Field
from jetson.tools import Tool, ToolContext
import yfinance as yf
import pandas as pd


class MarketDataParams(BaseModel):
    symbols: list[str] = Field(description="Stock symbols (e.g., ['AAPL', 'GOOGL'])")
    data_type: str = Field(
        default="price",
        description="Type: price, quote, history, fundamentals, options"
    )
    period: str = Field(default="1d", description="Period: 1d, 5d, 1mo, 3mo, 1y, 5y")
    interval: str = Field(default="1d", description="Interval: 1m, 5m, 1h, 1d, 1wk")


class MarketDataTool(Tool):
    id = "market_data"
    description = """
    Fetch market data for stocks, ETFs, and other securities.

    Examples:
    - Get current price: market_data(symbols=["AAPL"], data_type="price")
    - Get historical data: market_data(symbols=["SPY"], data_type="history", period="1mo")
    - Get options chain: market_data(symbols=["TSLA"], data_type="options")
    """
    parameters = MarketDataParams

    async def execute(self, args: MarketDataParams, ctx: ToolContext) -> str:
        results = {}

        for symbol in args.symbols:
            ticker = yf.Ticker(symbol)

            if args.data_type == "price":
                info = ticker.info
                results[symbol] = {
                    "price": info.get("currentPrice") or info.get("regularMarketPrice"),
                    "change": info.get("regularMarketChange"),
                    "change_pct": info.get("regularMarketChangePercent"),
                    "volume": info.get("regularMarketVolume"),
                }

            elif args.data_type == "history":
                hist = ticker.history(period=args.period, interval=args.interval)
                results[symbol] = hist.tail(10).to_string()

            elif args.data_type == "fundamentals":
                info = ticker.info
                results[symbol] = {
                    "pe_ratio": info.get("trailingPE"),
                    "market_cap": info.get("marketCap"),
                    "revenue": info.get("totalRevenue"),
                    "profit_margin": info.get("profitMargins"),
                }

            elif args.data_type == "options":
                options = ticker.options  # Expiration dates
                if options:
                    chain = ticker.option_chain(options[0])
                    results[symbol] = {
                        "expirations": list(options[:5]),
                        "calls": chain.calls.head().to_string(),
                        "puts": chain.puts.head().to_string(),
                    }

        return format_market_data(results)
```

### portfolio - Portfolio Analysis

```python
# flintstone/src/flintstone/tools/portfolio.py

from pydantic import BaseModel, Field
from jetson.tools import Tool, ToolContext
import pandas as pd
import numpy as np


class PortfolioParams(BaseModel):
    action: str = Field(
        description="Action: analyze, optimize, rebalance, backtest"
    )
    holdings: dict[str, float] = Field(
        default=None,
        description="Holdings as {symbol: shares} or {symbol: weight}"
    )
    benchmark: str = Field(default="SPY", description="Benchmark symbol")
    risk_free_rate: float = Field(default=0.05, description="Risk-free rate")


class PortfolioTool(Tool):
    id = "portfolio_analysis"
    description = """
    Analyze portfolio performance, risk metrics, and optimization.

    Examples:
    - Analyze holdings: portfolio_analysis(action="analyze", holdings={"AAPL": 100, "GOOGL": 50})
    - Optimize allocation: portfolio_analysis(action="optimize", holdings={"AAPL": 0.3, "GOOGL": 0.7})
    """
    parameters = PortfolioParams

    async def execute(self, args: PortfolioParams, ctx: ToolContext) -> str:
        if args.action == "analyze":
            return await self._analyze(args.holdings, args.benchmark)
        elif args.action == "optimize":
            return await self._optimize(args.holdings, args.risk_free_rate)
        elif args.action == "rebalance":
            return await self._rebalance(args.holdings)

    async def _analyze(self, holdings: dict, benchmark: str) -> str:
        """Calculate portfolio metrics."""
        # Fetch historical returns
        # Calculate: Sharpe, Sortino, Max Drawdown, Beta, Alpha
        return """
Portfolio Analysis:
──────────────────
Total Value:     $125,430.00
Daily Return:    +1.23%
YTD Return:      +15.7%

Risk Metrics:
  Sharpe Ratio:  1.45
  Sortino Ratio: 1.89
  Max Drawdown:  -12.3%
  Beta:          1.12
  Alpha:         +3.2%

Holdings:
  AAPL:  $65,000 (51.8%)
  GOOGL: $60,430 (48.2%)
"""
```

### risk - Risk Calculations

```python
# flintstone/src/flintstone/tools/risk.py

from pydantic import BaseModel, Field
from jetson.tools import Tool, ToolContext


class RiskParams(BaseModel):
    calculation: str = Field(
        description="Calculation: var, cvar, greeks, stress_test, scenario"
    )
    portfolio: dict = Field(default=None, description="Portfolio holdings")
    confidence: float = Field(default=0.95, description="Confidence level for VaR")
    horizon: int = Field(default=1, description="Time horizon in days")


class RiskTool(Tool):
    id = "risk_calc"
    description = """
    Calculate risk metrics for portfolios and positions.

    Examples:
    - Calculate VaR: risk_calc(calculation="var", portfolio={"AAPL": 100}, confidence=0.99)
    - Calculate Greeks: risk_calc(calculation="greeks", portfolio={"AAPL_CALL_150": 10})
    - Stress test: risk_calc(calculation="stress_test", portfolio={"SPY": 1000})
    """
    parameters = RiskParams

    async def execute(self, args: RiskParams, ctx: ToolContext) -> str:
        if args.calculation == "var":
            return self._calculate_var(args.portfolio, args.confidence, args.horizon)
        elif args.calculation == "greeks":
            return self._calculate_greeks(args.portfolio)
        elif args.calculation == "stress_test":
            return self._stress_test(args.portfolio)

    def _calculate_var(self, portfolio, confidence, horizon):
        return f"""
Value at Risk (VaR) Analysis
────────────────────────────
Confidence Level: {confidence * 100}%
Time Horizon:     {horizon} day(s)

Results:
  Parametric VaR:   $12,450
  Historical VaR:   $13,200
  Monte Carlo VaR:  $12,890

Interpretation:
  With {confidence * 100}% confidence, the portfolio will not lose
  more than $12,890 over the next {horizon} day(s).
"""
```

### compliance - Regulatory Compliance

```python
# flintstone/src/flintstone/tools/compliance.py

from pydantic import BaseModel, Field
from jetson.tools import Tool, ToolContext


class ComplianceParams(BaseModel):
    check_type: str = Field(
        description="Check: pre_trade, post_trade, position_limits, restricted_list"
    )
    trade: dict = Field(default=None, description="Trade details")
    portfolio: dict = Field(default=None, description="Current portfolio")


class ComplianceTool(Tool):
    id = "compliance_check"
    description = """
    Check trades and positions against compliance rules.

    Examples:
    - Pre-trade check: compliance_check(check_type="pre_trade", trade={"symbol": "AAPL", "qty": 1000})
    - Position limits: compliance_check(check_type="position_limits", portfolio={"AAPL": 50000})
    """
    parameters = ComplianceParams

    async def execute(self, args: ComplianceParams, ctx: ToolContext) -> str:
        if args.check_type == "pre_trade":
            return self._pre_trade_check(args.trade)
        elif args.check_type == "position_limits":
            return self._position_limits_check(args.portfolio)

    def _pre_trade_check(self, trade):
        return """
Pre-Trade Compliance Check
──────────────────────────
Trade: BUY 1000 AAPL @ MARKET

✓ Not on restricted list
✓ Within position limits (current: 5000, limit: 10000)
✓ Within sector concentration (Tech: 35%, limit: 40%)
✓ Sufficient buying power
✗ WARNING: Large order (>5% ADV) - consider TWAP execution

Recommendation: APPROVED with execution guidance
"""
```

---

## Finance System Prompt

```python
# flintstone/src/flintstone/prompts/system.py

FINANCE_SYSTEM_PROMPT = """
You are Flintstone, an AI assistant specialized in financial software development,
quantitative analysis, and trading systems.

## Financial Expertise

You have deep knowledge of:
- Market microstructure and trading systems
- Quantitative finance and derivatives pricing
- Risk management (VaR, Greeks, stress testing)
- Portfolio optimization and asset allocation
- Regulatory compliance (SEC, FINRA, MiFID II)
- Financial data formats (FIX, FPML, ISO 20022)

## Available Finance Tools

In addition to standard coding tools, you have:
- `market_data`: Fetch real-time and historical market data
- `portfolio_analysis`: Analyze portfolio performance and risk
- `risk_calc`: Calculate VaR, Greeks, stress tests
- `backtest`: Backtest trading strategies
- `trade_execute`: Execute trades (with compliance checks)
- `compliance_check`: Verify regulatory compliance

## Finance Coding Guidelines

When writing financial code:
1. **Precision**: Use Decimal for monetary calculations, not float
2. **Timestamps**: Always use timezone-aware datetimes (UTC preferred)
3. **Validation**: Validate all market data before calculations
4. **Audit Trail**: Log all trading decisions and executions
5. **Error Handling**: Financial operations must never fail silently
6. **Testing**: Include edge cases (market holidays, splits, dividends)

## Risk Awareness

Always consider:
- Market risk, credit risk, operational risk
- Liquidity constraints and market impact
- Regulatory requirements and reporting
- Data quality and staleness
- System failures and failover procedures

## Compliance First

Before any trade-related action:
1. Check restricted lists
2. Verify position limits
3. Confirm compliance with investment guidelines
4. Document decision rationale
"""
```

---

## Finance Permissions

```python
# flintstone/src/flintstone/config/permissions.py

FINANCE_PERMISSIONS = [
    # Default: ask for most things
    {"permission": "*", "pattern": "*", "action": "ask"},

    # Auto-allow read operations
    {"permission": "read", "pattern": "*", "action": "allow"},
    {"permission": "glob", "pattern": "*", "action": "allow"},
    {"permission": "grep", "pattern": "*", "action": "allow"},

    # Auto-allow market data (read-only)
    {"permission": "market_data", "pattern": "*", "action": "allow"},

    # Always ask for trades
    {"permission": "trade_execute", "pattern": "*", "action": "ask"},

    # Always ask for portfolio changes
    {"permission": "portfolio_rebalance", "pattern": "*", "action": "ask"},

    # Block dangerous operations
    {"permission": "bash", "pattern": "rm -rf *", "action": "deny"},
    {"permission": "bash", "pattern": "* --force *", "action": "deny"},
]
```

---

## CLI Wrapper

```python
# flintstone/src/flintstone/cli.py

import typer
from jetson.cli import app as jetson_app
from .plugin import create_plugin

app = typer.Typer(
    name="flintstone",
    help="Flintstone: Jetson for Finance - AI coding agent for financial software"
)

# Import all Jetson commands
app.add_typer(jetson_app, name="core")


@app.command()
def run(
    prompt: str = typer.Argument(..., help="Task to execute"),
    portfolio: str = typer.Option(None, help="Portfolio file to load"),
):
    """Run a one-shot financial task."""
    from jetson import Jetson

    # Create Jetson with Flintstone plugin
    jetson = Jetson(plugins=[create_plugin()])

    # Load portfolio context if provided
    if portfolio:
        prompt = f"Portfolio context: {load_portfolio(portfolio)}\n\n{prompt}"

    jetson.run(prompt)


@app.command()
def analyze(
    symbols: list[str] = typer.Argument(..., help="Symbols to analyze"),
):
    """Quick market analysis for symbols."""
    from jetson import Jetson

    jetson = Jetson(plugins=[create_plugin()])
    jetson.run(f"Analyze these securities: {', '.join(symbols)}")


@app.command()
def backtest(
    strategy: str = typer.Argument(..., help="Strategy file or description"),
    start: str = typer.Option("2023-01-01", help="Start date"),
    end: str = typer.Option("2024-01-01", help="End date"),
):
    """Backtest a trading strategy."""
    from jetson import Jetson

    jetson = Jetson(plugins=[create_plugin()])
    jetson.run(f"Backtest this strategy from {start} to {end}: {strategy}")


if __name__ == "__main__":
    app()
```

---

## Usage Examples

```bash
# Install
pip install flintstone

# Or from source
git clone ... && cd flintstone && pip install -e .

# Basic usage (inherits all Jetson commands)
flintstone run "Analyze AAPL stock and create a DCF model"

flintstone chat  # Interactive mode with finance tools

# Finance-specific commands
flintstone analyze AAPL GOOGL MSFT

flintstone backtest "momentum strategy" --start 2022-01-01 --end 2024-01-01

# Use Jetson core commands
flintstone core serve  # Start server
flintstone core tui    # Launch TUI
```

---

## Extension Pattern for Other Domains

The same pattern works for any domain specialization:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│   JETSON SPECIALIZATION PATTERN                                              │
│                                                                              │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│   │ Flintstone  │  │  Rosie      │  │  Astro      │  │  George     │       │
│   │  (Finance)  │  │  (DevOps)   │  │  (ML/AI)    │  │  (Legal)    │       │
│   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘       │
│          │                │                │                │               │
│          └────────────────┴────────────────┴────────────────┘               │
│                                    │                                        │
│                                    ▼                                        │
│                          ┌─────────────────┐                                │
│                          │     JETSON      │                                │
│                          │  (Core Engine)  │                                │
│                          │                 │                                │
│                          │  • Agent Loop   │                                │
│                          │  • Core Tools   │                                │
│                          │  • Plugin Sys   │                                │
│                          │  • MCP/LSP      │                                │
│                          └─────────────────┘                                │
│                                                                              │
│   Each specialization adds:                                                  │
│   • Domain-specific tools                                                    │
│   • Domain-specific MCP servers                                              │
│   • Domain-specific skills/prompts                                           │
│   • Domain-specific permissions                                              │
│   • Domain-specific CLI commands                                             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Summary

| Aspect | Jetson (Core) | Flintstone (Finance) |
|--------|---------------|----------------------|
| Package | `jetson` | `flintstone` |
| Entry | `jetson` | `flintstone` / `fstone` |
| Tools | 19 generic | +6 finance-specific |
| MCP | Generic | + Bloomberg, Refinitiv, Yahoo |
| Skills | Generic coding | + Quant, Risk, Compliance |
| Prompts | Generic AI assistant | Finance-aware assistant |
| Permissions | Standard | + Trade safeguards |

**Key Benefits:**
1. Jetson stays generic and reusable
2. Flintstone adds finance layer without forking
3. Updates to Jetson automatically benefit Flintstone
4. Same pattern works for any domain specialization
