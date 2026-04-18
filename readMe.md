# Alpaca Watchlist Bot

`alpaca-watchlist-bot` is a small Python trading helper built around the Alpaca brokerage API. It runs as a daily price-check job over a JSON watchlist, tracks multi-day price direction streaks, and either sends email alerts or places simple market orders depending on how each symbol is configured.

This repository is best understood as a lightweight personal automation project rather than a production trading system. The code is straightforward and usable, but it still has a few hard-coded assumptions and operational rough edges documented below.

## What the project does

On each run, the bot:

1. Connects to Alpaca using credentials from `secrets/secrets.ini`.
2. Exits immediately if the market is closed.
3. Downloads the watchlist JSON from Google Cloud Storage.
4. Fetches the latest daily bar for each symbol.
5. Compares the current price with the stored price history.
6. Updates an `UP` or `DOWN` move streak for each symbol.
7. Takes one of two actions based on the watchlist entry:
   - `ALERT`: send an email after the configured streak threshold is reached.
   - `TRADE`: place simple buy or sell orders through Alpaca.
8. Uploads the updated watchlist back to Google Cloud Storage.

## Trading logic

Each watchlist item is configured as either `ALERT` or `TRADE`.

### `ALERT` entries

- The bot tracks whether the symbol has been moving up or down.
- Once the streak reaches the threshold, it appends an alert to an in-memory alert list.
- At the end of the run, alerts are sent through Gmail.

### `TRADE` entries

- On an upward move:
  - If the configured threshold is met and buying power is sufficient, the bot submits a market buy order.
  - After a buy, it trails the stop-loss upward using `stop_loss_percent`.
- On a downward move:
  - If the threshold is met, the bot checks whether the account already holds the symbol.
  - If the current price is below the configured stop-loss, it submits a market sell order.

### Price comparison detail

The code uses:

- `LAST_PRICE` from the stored watchlist as the previous observed price.
- A historical bar from roughly 14 days earlier as an additional reference when evaluating upward movement.

That makes the strategy more of a streak tracker with some momentum checks than a complete portfolio-management system.

## Repository layout

```text
.
├── main.py                  # Main run loop and trade/alert orchestration
├── app/
│   ├── google_service.py    # Gmail OAuth + service creation
│   ├── utils.py             # Watchlist, streak, stop-loss, and order helpers
│   ├── unitTests.py         # Unit tests for helper functions
│   └── test.py              # Older manual test harness
├── assets/watchlist.json    # Example watchlist structure
├── test_assets/             # Small JSON fixtures used for testing
├── requirements.txt         # Python dependencies
├── CONTRIBUTING.md
├── CODE_OF_CONDUCT.md
└── LICENSE
```

## Requirements

- Python 3
- An Alpaca account with API keys
- A Google Cloud Storage bucket
- Google credentials with access to that bucket
- Gmail API OAuth credentials for sending alerts

## Setup

### 1. Install dependencies

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### 2. Configure secrets

The code expects a `secrets/secrets.ini` file with at least these values under the `DEFAULT` section:

```ini
[DEFAULT]
APCA_API_KEY_ID=your_alpaca_key
APCA_API_SECRET_KEY=your_alpaca_secret
APCA_API_BASE_URL=https://paper-api.alpaca.markets
TO_MAIL=you@example.com
BUCKET_NAME=your-gcs-bucket
```

The repository also expects these files to exist:

- `secrets/google_client_secrets.json`
- `static/token_gmail_v1.pickle`

On the first Gmail OAuth flow, the token can be created interactively and then stored in the configured Google Cloud Storage bucket.

### 3. Configure Google Cloud access

`google.cloud.storage.Client()` is created directly in the code, so your environment needs working Google Cloud credentials. In practice that usually means one of the following:

- Application Default Credentials on your machine
- A service account credential exported through `GOOGLE_APPLICATION_CREDENTIALS`

### 4. Upload the watchlist JSON

The bot reads its watchlist from Google Cloud Storage. By default it looks for:

```text
assets/watchlist.json
```

Use the local [`assets/watchlist.json`](assets/watchlist.json) file as a template for the object you store in the bucket.

## Watchlist format

Each top-level key is a ticker symbol. Each symbol entry looks like this:

```json
{
  "MSFT": {
    "type": "TRADE",
    "LAST_PRICE": 288.16,
    "MOVE_TYPE": "UP",
    "streak": 1,
    "stop_loss": 275.14550399999996,
    "stop_loss_percent": 4
  }
}
```

Field meanings:

- `type`: either `ALERT` or `TRADE`
- `LAST_PRICE`: last observed daily close used for the next comparison
- `MOVE_TYPE`: current direction state, typically `UP`, `DOWN`, or `NONE`
- `streak`: count of consecutive detections in the current move direction
- `stop_loss`: active stop-loss price or `"NONE"`
- `stop_loss_percent`: percentage used when trailing the stop-loss after a buy

For new symbols, the examples in `test_assets/` show that the code can start from entries such as:

- `LAST_PRICE: "NONE"`
- `MOVE_TYPE: "NONE"`
- `streak: "NONE"`
- `stop_loss: "NONE"`

## Running the bot

```bash
python3 main.py
```

What happens during a normal run:

1. Alpaca authentication is initialized.
2. The script checks whether the market is open.
3. The watchlist is downloaded from Google Cloud Storage.
4. Prices are evaluated and streaks are updated.
5. Orders and alerts are generated when thresholds are met.
6. The watchlist is uploaded back to storage.
7. If any alerts were collected, a Gmail message is sent.

## Tests

The maintained automated tests are the helper unit tests:

```bash
python3 -m unittest app.unitTests
```

These tests cover the core utility functions for:

- move detection
- streak threshold checks
- stop-loss checks
- buying power checks
- watchlist mutation helpers

There is also an older script at [`app/test.py`](app/test.py), but it is closer to an ad hoc harness than a current automated test entrypoint.

## Current implementation notes

These are worth knowing before using the project heavily:

- `main(**kwargs)` currently ignores incoming keyword arguments and always calls `startEngine()` with defaults.
- The watchlist is downloaded from the path given by `inputPath`, but updates are uploaded to `watchlist.json` in the bucket rather than back to the same path.
- The code uses many broad `try/except` blocks and suppresses most exceptions, which makes failures quiet.
- Order placement is intentionally simple: market orders only, fixed quantities, and no portfolio sizing beyond a buying-power percentage check.
- The project is designed around daily bars and a single-pass execution model, not continuous intraday monitoring.

## Development notes

If you want to extend the project, the most natural places to start are:

- [`main.py`](main.py) for the orchestration flow
- [`app/utils.py`](app/utils.py) for strategy helper logic
- [`app/google_service.py`](app/google_service.py) for Gmail authentication and token handling

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md) for contribution guidance and [`CODE_OF_CONDUCT.md`](CODE_OF_CONDUCT.md) for community expectations.

## License

This project is licensed under the [MIT License](LICENSE).
