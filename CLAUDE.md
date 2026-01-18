# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the official **Binance Public Data** repository - a toolkit for downloading historical market data from Binance's public data archive at `https://data.binance.vision/`. It provides both Python scripts and shell scripts to programmatically access:
- **Spot trading data**: klines (candlesticks), trades, aggregated trades
- **USD-M Futures** (USDT-margined): klines, trades, aggregated trades, index price, mark price, premium index
- **COIN-M Futures** (coin-margined): klines, trades, aggregated trades, index price, mark price, premium index

## Setup and Dependencies

```bash
# Install Python dependencies (minimal - just pandas)
pip install -r python/requirements.txt

# Set output directory environment variable (optional, can override with -folder)
export STORE_DIRECTORY=/path/to/output
```

## Running Download Scripts

All Python scripts are in the `python/` directory. Basic pattern:

```bash
# Download klines (candlestick data)
python3 python/download-kline.py -t spot -s ETHUSDT BTCUSDT -i 1h 1d -y 2023 2024

# Download trades
python3 python/download-trade.py -t spot -s ETHUSDT -startDate 2023-01-01 -endDate 2023-12-31

# Download aggregated trades
python3 python/download-aggTrade.py -t um -s BTCUSDT -y 2023

# Futures-only data (index/mark/premium price klines)
python3 python/download-futures-indexPriceKlines.py -t um -s BTCUSDT
python3 python/download-futures-markPriceKlines.py -t cm -s BTCUSD_PERP
python3 python/download-futures-premiumPriceKlines.py -t um -i 1m
```

### Common Arguments

All download scripts support these arguments:

| Argument | Purpose |
|----------|---------|
| `-t` | **Market type**: `spot`, `um` (USD-M Futures), or `cm` (COIN-M Futures) - **Required** |
| `-s` | Symbols to download (space-separated, e.g., `BTCUSDT ETHUSDT`) - Default: all symbols |
| `-i` | Kline intervals (e.g., `1m`, `1h`, `1d`, `1w`) - Default: all intervals |
| `-y` | Years (e.g., `2023 2024`) - Default: 2020 to current year |
| `-m` | Months (1-12, space-separated) - Default: all months |
| `-d` | Specific dates in `YYYY-MM-DD` format - Default: from 2020-01-01 |
| `-startDate` | Start date range in `YYYY-MM-DD` format - Default: 2020-01-01 |
| `-endDate` | End date range in `YYYY-MM-DD` format - Default: current date |
| `-skip-monthly` | Set to `1` to skip monthly files |
| `-skip-daily` | Set to `1` to skip daily files |
| `-folder` | Output directory - Default: `STORE_DIRECTORY` env var or current directory |
| `-c` | Set to `1` to download `.CHECKSUM` files for data verification |
| `-max_workers` | Number of threads for parallel downloads (multi-threaded scripts only) |

### Date Filtering Behavior

Scripts download data from **2020-01-01** by default. Use date filters to limit range:
- `-startDate` and `-endDate`: Download all data within date range
- `-y` and `-m`: Download specific year/month combinations
- `-d`: Download specific dates only

These options are mutually exclusive - using date range vs specific years/months/dates.

## Architecture

### Core Components

1. **`python/enums.py`** - Central configuration
   - `YEARS`: Available data years (2017-2024)
   - `INTERVALS`: All kline intervals (`1s`, `1m`, `3m`, `5m`, `15m`, `30m`, `1h`, `2h`, `4h`, `6h`, `8h`, `12h`, `1d`, `3d`, `1w`, `1mo`)
   - `TRADING_TYPE`: Market types (`spot`, `um`, `cm`)
   - `BASE_URL`: Binance data endpoint (`https://data.binance.vision/`)

2. **`python/utility.py`** - Shared utilities used by all scripts
   - `get_all_symbols(type)`: Fetches all trading symbols from Binance API for given market type
   - `get_download_url(file_url)`: Constructs full download URL
   - `get_destination_dir(file_url, folder)`: Determines local storage path
   - `get_path(trading_type, market_data_type, time_period, symbol, interval)`: Constructs path for data files
   - `download_file(base_path, file_name, date_range, folder)`: Downloads single file with progress bar
   - `get_parser(parser_type)`: Creates ArgumentParser with common arguments

3. **Download Scripts** - Each script is standalone but shares utilities
   - `download-kline.py`: Multi-threaded kline/candlestick downloader
   - `download-trade.py`: Trade data downloader
   - `download-aggTrade.py`: Aggregated trades downloader
   - `download-futures-*.py`: Futures-specific data (index/mark/premium prices)

### Data URL Structure

```
https://data.binance.vision/
├── data/
│   ├── spot/
│   │   ├── monthly/klines/{SYMBOL}/{INTERVAL}/{SYMBOL}-{INTERVAL}-{YEAR}-{MONTH}.zip
│   │   └── daily/klines/{SYMBOL}/{INTERVAL}/{SYMBOL}-{INTERVAL}-{YEAR}-{MONTH}-{DAY}.zip
│   └── futures/
│       ├── um/ (USD-M Futures)
│       └── cm/ (COIN-M Futures)
```

### Multi-Threading

The `download-kline.py` script uses multi-threading for parallel downloads. The `-max_workers` argument controls thread count. Other download scripts may be single-threaded.

## Cross-Platform Compatibility

The scripts are designed to work on:
- **Linux**: Uses `sha256sum` for checksum verification
- **macOS**: Uses `shasum -a 256` for checksum verification
- **Windows**: Compatible (recent commits fixed Windows 11 issues)

## Shell Scripts

The `shell/` directory contains simpler wrapper scripts:
- `download-klines.sh`: Basic kline downloader
- `download-futures-klines-simultaneously.sh`: Batch futures downloads
- `fetch-all-trading-pairs.sh`: Retrieves all current trading pairs from Binance

These are less feature-rich than the Python scripts but useful for quick tasks.

## Data Integrity

Each downloaded `.zip` file has a corresponding `.CHECKSUM` file. Verify downloads with:

```bash
# Linux
sha256sum -c FILENAME.zip.CHECKSUM

# macOS
shasum -a 256 -c FILENAME.zip.CHECKSUM
```

Use the `-c 1` argument when running scripts to automatically download checksum files.

## Key Implementation Details

- **Symbol fetching**: Scripts call Binance API (`/api/v3/exchangeInfo` for spot, `/fapi/v1/exchangeInfo` for USD-M futures, `/dapi/v1/exchangeInfo` for COIN-M futures) to get current trading pairs
- **Skip existing files**: `download_file()` checks if file exists locally and skips re-download
- **Progress indicators**: Downloads show 50-character progress bar (`[#######.....]`)
- **SSL/TLS**: Uses `certifi` for proper certificate verification
- **Error handling**: HTTP 404 errors print "File not found" and continue (not all date/symbol combinations exist)
