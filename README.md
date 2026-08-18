# Trading-AI
XAUUSD AI Market Intelligence Terminal
A modular XAUUSD-only market analysis and signal platform: FastAPI backend (indicators, market structure, support/resistance, weighted signal engine, backtester, ML skeleton), PostgreSQL schema, and a Next.js/Tailwind dark trading dashboard.
⚠️ Honest scope note
This is a real, runnable foundation — not a finished, deployed, licensed trading product. Specifically:
No live market-data provider is wired in. app/services/market_data.py is provider-agnostic and will correctly return "Live market data unavailable" until you implement _fetch_from_provider-style calls for a real source (broker REST API, Polygon.io, Twelve Data, OANDA, etc.) via the MARKET_DATA_* env vars. This is intentional — the spec explicitly forbids hard-coded/fake prices and fake signals from fake prices.
No news calendar provider is wired in (same pattern — plug one into NEWS_PROVIDER / NEWS_API_KEY).
The ML classifier is a real, walk-forward-validated skeleton (app/services/ml_model.py) but has not been trained on your actual historical data or calibrated — do that before trusting its output as anything beyond "signal confidence."
Not deployed. Deployment steps below are accurate but you'll need to actually provision Vercel/Render/Railway/AWS accounts and a Postgres instance.
Broker contract size for XAUUSD is never assumed — the risk calculator requires contract_size_oz as an input you get from your broker.
Everything else — indicator math, EMA/RSI/MACD/ATR/Bollinger/ADX/StochRSI, swing/BOS/CHoCH market structure detection, support/resistance clustering, the weighted (100-point) signal scoring engine, the spread/slippage-aware backtester, the risk calculator, and the dashboard UI — is implemented and unit-tested (see backend/tests/).
Architecture
xauusd-terminal/
├── backend/            FastAPI app
│   ├── app/
│   │   ├── main.py             app entrypoint, routers, CORS, rate limiting
│   │   ├── config.py           env-driven settings (no hard-coded secrets)
│   │   ├── database.py         SQLAlchemy engine/session
│   │   ├── schemas.py          Pydantic request/response models
│   │   ├── models/tables.py    ORM models matching db/schema.sql
│   │   ├── routers/            market.py, signal.py, risk.py
│   │   └── services/
│   │       ├── indicators.py        EMA/SMA/RSI/MACD/StochRSI/ATR/BB/ADX/VWAP
│   │       ├── market_structure.py  swings, HH/HL/LH/LL, BOS/CHoCH
│   │       ├── support_resistance.py zone clustering, psychological levels
│   │       ├── signal_engine.py     weighted scoring -> BUY/SELL/WAIT
│   │       ├── market_data.py       provider-agnostic live data layer
│   │       ├── risk_calculator.py   position sizing
│   │       ├── backtest_engine.py   bar-by-bar backtest w/ spread & slippage
│   │       └── ml_model.py          walk-forward validated classifier
│   ├── tests/           pytest unit tests (indicators + signal engine)
│   └── requirements.txt
├── frontend/            Next.js 14 + TypeScript + Tailwind
│   └── app/
│       ├── page.tsx               dashboard
│       └── components/            PriceHeader, SignalCard
└── db/schema.sql        Postgres DDL for all 9 required tables
Local setup
Backend
cd backend
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env        # fill in DATABASE_URL and (optionally) a data provider
createdb xauusd              # or use your Postgres GUI of choice
psql xauusd -f ../db/schema.sql
uvicorn app.main:app --reload --port 8000
Visit http://localhost:8000/docs for interactive API docs.
Frontend
cd frontend
npm install
cp .env.local.example .env.local
npm run dev
Visit http://localhost:3000.
Tests
cd backend
pytest tests/ -v
Connecting a real market-data provider
Sign up with a provider that carries XAUUSD spot/CFD data (e.g. a broker's REST API, Polygon.io, Twelve Data, OANDA v20 API).
Set MARKET_DATA_PROVIDER, MARKET_DATA_API_KEY, MARKET_DATA_BASE_URL in .env.
Implement the request/response mapping in app/services/market_data.py::get_current_price and get_candles to match that provider's actual API shape (the file has a working example structure to adapt).
Do the same for a news/economic-calendar provider to power the News and News Risk modules.
Deployment (outline)
Frontend → Vercel: vercel --prod from frontend/, set NEXT_PUBLIC_API_BASE_URL to your deployed backend URL in Vercel's env vars.
Backend → Render/Railway/AWS: deploy backend/ as a Docker/Python service, set all .env variables in the platform's secret manager, run db/schema.sql against your managed Postgres instance, provision Redis.
Restrict CORS in main.py to your real frontend origin before going live.
Put the backend behind HTTPS (the platforms above do this by default).
Compliance / disclaimer
The platform must always display:
This platform provides market analysis and probability-based signals for educational and informational purposes only. It does not guarantee profits or predict future prices with certainty. Trading XAUUSD involves substantial risk, including the possibility of losing your entire trading capital. Always verify market conditions and use appropriate risk management.
Never use "guaranteed profit," "100% accurate," or similar language anywhere in the UI or API responses.
