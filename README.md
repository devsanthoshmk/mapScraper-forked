![mapScraper](github-header-image.png)

# mapScraper

## Purpose

mapScraper is a Python maps scraper and local business data extraction tool. It scrapes Google Maps search results to collect business names, categories, addresses, phone numbers, websites, coordinates, ratings, and review counts without requiring Google Maps API keys.

The project provides two command-line interfaces:
1. `mapScraperX.py`: A Google Places search crawler with an asynchronous CID fallback endpoint for retrieving missing phone numbers and addresses.
2. `main.py`: A unified data pipeline supporting raw Google Maps scraping, website enrichment, and rule-based lead scoring (0 to 100).

## Architecture

The project separates raw data scraping from data enrichment.

```
mapScraper-forked/
├── main.py                  Pipeline CLI for scrape, enrich, and full modes
├── mapScraperX.py           Search crawler CLI with fallback modes
├── requirements.txt         Python package dependencies
├── mapScraper/
│   ├── __init__.py          Package exports for places_crawler
│   ├── places_crawler.py    Places tab scraper with async CID fallback
│   └── placesCrawlerV2.py   Google Maps SPA scraper with tbm=map JSON parser
├── enrichment/
│   ├── __init__.py          Enrichment package marker
│   ├── features.py          Feature engineering from scraped CSV data
│   ├── web_scraper.py       Async website scraper for tech stack and contacts
│   └── scoring.py           Rule-based lead scoring (0 to 100) and segmentation
├── pipeline/
│   ├── __init__.py          Pipeline package marker
│   └── orchestrator.py      Workflow coordinator connecting scraper and enrichment
├── data/
│   └── output.csv           Default output CSV path
└── query_example.txt        Example search queries
```

### Scraping engines

The codebase includes two independent scraping implementations:

1. `mapScraper/places_crawler.py` (called by `mapScraperX.py`):
   - Sends requests to the Google Search Places tab (`https://www.google.com/search?q={query}&udm=1`).
   - Parses business cards from HTML using `lxml`.
   - Handles missing phone numbers through an asynchronous fallback to Google's CID detail endpoint (`https://www.google.com/async/lcl_akp?q={query}&async=ludocids:{cid},_fmt:prog`).
   - Offers three operational modes: `fast`, `standard`, and `complete`.

2. `mapScraper/placesCrawlerV2.py` (called by `main.py`):
   - Resolves the search session from `https://www.google.com/maps/search/{query}`.
   - Extracts the internal `pb=` search path and paginates through `https://www.google.com/search?tbm=map...` in batches of 20 results.
   - Parses the internal JSON array at `data[64]` after stripping Google's `)]}'` cross-site script inclusion prefix.
   - Deduplicates results by unique place ID before saving.

### Enrichment pipeline

The enrichment pipeline in `enrichment/` processes any compatible CSV file:
- `features.py`: Computes validation flags (`has_phone`, `has_website`, `domain_valid`), weighted ratings (`rating_score`), and relative review volume (`review_density`).
- `web_scraper.py`: Fetches business homepages asynchronously using `aiohttp` to detect contact sections, services pages, modern JavaScript frameworks (React, Vue, Next.js, Nuxt, Angular, Gatsby, Svelte), and dominant keywords. Unreachable domains fail silently.
- `scoring.py`: Computes a deterministic 0 to 100 lead score and assigns size segments (`micro`, `small`, `medium`, `large`).

## Setup

### Prerequisites

- Python 3.10 or newer
- pip package manager

### Installation

Clone the repository and install dependencies:

```bash
git clone https://github.com/devsanthoshmk/mapScraper-forked.git
cd mapScraper-forked
pip install -r requirements.txt
```

### Dependencies

The project uses the following key packages:
- `aiohttp`: Asynchronous HTTP requests for crawling and website enrichment
- `beautifulsoup4` and `lxml`: HTML parsing
- `lxml_html_clean`: HTML sanitization
- `pandas`: Data manipulation and CSV export
- `phonenumbers`: Phone number parsing and validation
- `tqdm`: Terminal progress bars

## Usage

### Original search CLI (mapScraperX.py)

`mapScraperX.py` runs the Google Places scraper directly.

```bash
# Scrape a single query
python mapScraperX.py "restaurants in Miami" --limit 50

# Scrape with complete phone and address fallback
python mapScraperX.py "dentists in Chicago" --mode complete

# Scrape multiple queries from a text file
python mapScraperX.py --queries-file query_example.txt

# Specify language, country, concurrency, and output path
python mapScraperX.py --queries-file query_example.txt --lang en --country us --limit 25 --concurrent 5 --output-file data/results.csv
```

### Pipeline CLI (main.py)

`main.py` provides three workflow modes: `scrape`, `enrich`, and `full`.

```bash
# Mode 1: Scrape only
python main.py --mode scrape "marketing agencies in New York" --limit 50
python main.py --mode scrape --queries-file query_example.txt

# Mode 2: Enrich an existing CSV file
python main.py --mode enrich --input data/output.csv

# Mode 2: Enrich without website visits (offline and faster)
python main.py --mode enrich --input data/output.csv --no-web-scraping

# Mode 3: Full pipeline (scrape Google Maps and enrich results in one command)
python main.py --mode full --queries-file query_example.txt --output-file data/leads.csv
```

## Scraping modes and fallback logic

`mapScraperX.py` provides three modes to balance execution speed and data completeness:

- `fast`: Collects data from the primary Google Places tab only. It skips the CID details endpoint. This mode is the fastest.
- `standard`: Checks whether phone numbers exist in the primary results. If any result contains a phone number, it skips the fallback step. If no phone numbers exist, it queries the CID endpoint (`/async/lcl_akp`) to retrieve missing numbers. This is the default mode.
- `complete`: Queries the CID detail endpoint for every business record that lacks a phone number, address, or website link. This mode retrieves the highest percentage of complete contact information.

### The /places fallback mechanism

Google Maps search listings frequently truncate addresses or omit contact numbers on the initial search page. To recover this data without browser rendering:
1. The scraper identifies the Google customer ID (`cid`) from the place listing.
2. An asynchronous HTTP request is sent to `https://www.google.com/async/lcl_akp?q={query}&async=ludocids:{cid},_fmt:prog`.
3. The parser extracts the specific `Phone`, `Address`, and `Website` values from the returned partial DOM.

## Configuration

### Command-line arguments for main.py

| Argument | Default | Description |
|---|---|---|
| `--mode` | `scrape` | Pipeline mode: `scrape`, `enrich`, or `full` |
| `query` | None | Search query string (positional) |
| `--queries-file FILE` | None | Path to plain text file with one query per line |
| `--lang CODE` | `en` | Google Maps interface language code (`hl` parameter) |
| `--country CODE` | `us` | Google Maps region code (`gl` parameter) |
| `--limit N` | None | Maximum results per query |
| `--output-file PATH` | `data/output.csv` | Destination path for raw CSV results |
| `--concurrent N` | `3` | Maximum concurrent scraping workers |
| `--input PATH` | None | Input CSV path for `enrich` mode |
| `--no-web-scraping` | False | Skip external website requests during enrichment |
| `--web-concurrent N` | `10` | Maximum concurrent website scraper workers |
| `--web-batch-size N` | `100` | Number of URLs processed per batch |
| `--web-timeout SEC` | `10` | Request timeout in seconds for website fetches |
| `--log-level LEVEL` | `INFO` | Logging verbosity: `DEBUG`, `INFO`, `WARNING`, `ERROR` |

### Command-line arguments for mapScraperX.py

| Argument | Default | Description |
|---|---|---|
| `query` | None | Search query string (positional) |
| `--queries-file FILE` | None | Path to plain text file with one query per line |
| `--lang CODE` | `en` | Language code |
| `--country CODE` | `us` | Country code |
| `--limit N` | `0` | Maximum results to collect (0 means no limit) |
| `--output-file PATH` | `data/output.csv` | Destination path for CSV output |
| `--concurrent N` | `3` | Maximum concurrent queries |
| `--mode MODE` | `standard` | Scraping mode: `fast`, `standard`, or `complete` |

### Common language and region codes

The `--lang` argument accepts standard two-letter ISO 639-1 language codes (for example, `en`, `es`, `fr`, `de`, `it`, `pt`, `ja`, `zh`). The `--country` argument accepts ISO 3166-1 alpha-2 region codes (for example, `us`, `gb`, `ca`, `au`, `de`, `fr`, `es`, `in`, `br`).

## Output formats

### Raw scrape CSV (mapScraperX.py)

| Column | Description |
|---|---|
| `id` | Google CID identifier |
| `gmap_url` | Direct Google Maps CID URL |
| `title` | Business name |
| `category` | Business classification |
| `address` | Street address |
| `phoneNumber` | Primary phone number |
| `additionalPhoneNumbers` | Pipe-delimited secondary phone numbers |
| `website` | Business website URL |
| `stars` | Average customer rating |
| `reviews` | Total review count |

### Raw scrape CSV (main.py)

| Column | Description |
|---|---|
| `id` | Google Place ID (`ChIJ...`) |
| `url_place` | Direct Google Maps place link |
| `title` | Business name |
| `category` | Business category |
| `address` | Full address |
| `phoneNumber` | Local telephone format |
| `completePhoneNumber` | International telephone format |
| `domain` | Cleaned website domain |
| `url` | Full website URL |
| `coor` | Latitude and longitude coordinate pair |
| `stars` | Average star rating |
| `reviews` | Total review count |
| `source_query` | Query string that produced the record |

### Enriched CSV schema

When running `main.py` in `enrich` or `full` mode, the output includes all raw scrape columns plus the following data points:

| Column | Type | Description |
|---|---|---|
| `has_phone` | bool | True if a local or international phone number is present |
| `has_website` | bool | True if a domain or website URL is present |
| `domain_valid` | bool | True if the domain passes regular expression syntax checks |
| `rating_score` | float | Rating multiplied by the natural logarithm of reviews plus one |
| `review_density` | float | Review count normalized against the maximum review count in the dataset |
| `web_has_contact` | bool | True if contact keywords appear in homepage text |
| `web_has_services` | bool | True if service or product keywords appear in homepage text |
| `web_keywords` | str | Comma-separated list of top content keywords |
| `web_is_modern` | bool | True if modern JavaScript frameworks (React, Vue, Next.js) are detected |
| `web_scraped` | bool | True if the website HTTP request completed successfully |
| `score` | float | Composite lead score from 0 to 100 |
| `segment` | str | Classification bucket: `micro`, `small`, `medium`, or `large` |

### Lead score calculation

The lead score ranges from 0 to 100 based on four transparent factors:
- Review volume (0 to 30 points): 30 points for 500 or more reviews, 24 for 200 to 499, 18 for 100 to 199, 12 for 50 to 99, 7 for 10 to 49, 3 for 1 to 9, 0 for 0 reviews.
- Star rating (0 to 25 points): 25 points for rating 4.5 or higher, 20 for 4.0 to 4.4, 15 for 3.5 to 3.9, 10 for 3.0 to 3.4, 5 for ratings above 0.
- Digital presence (0 to 30 points): 10 points for having a website, 5 for a valid domain syntax, 5 for contact signals, 5 for service signals, 5 for a modern frontend stack.
- Reachability (0 to 15 points): 15 points if a phone number exists.

Scores group into four tiers:
- `micro`: 0 to 24 points
- `small`: 25 to 49 points
- `medium`: 50 to 74 points
- `large`: 75 to 100 points

## Differences from upstream

This repository differs from the original `christivn/mapScraper` upstream project in several technical areas:

1. Active Google endpoints: Upstream used `/localservices/prolist`, which Google permanently discontinued with HTTP 410. This repository implements two working pathways: Google Places search tab scraping (`udm=1`) and Google Maps SPA JSON extraction (`tbm=map`).
2. Async CID contact fallback: Introduces `places_crawler.py` with asynchronous querying of Google's `/async/lcl_akp` endpoint to recover missing phone numbers, addresses, and website links for places records.
3. Modular package structure: Isolates crawling logic inside the `mapScraper` package with clean imports and independent CLI tools (`mapScraperX.py` and `main.py`).
4. Lead enrichment and scoring: Adds the `enrichment` and `pipeline` packages to engineer features, scrape website homepages for tech stack signals, and compute explainable 0 to 100 lead scores.
5. Modern dependency requirements: Corrects `requirements.txt` to support Python 3.10 and newer, adding `lxml_html_clean` and updating version constraints for `aiohttp`, `pandas`, `phonenumbers`, and `beautifulsoup4`.

## License

This project is licensed under the MIT License. See the LICENSE file for details. This software is provided for research and educational purposes. Ensure compliance with Google Terms of Service when crawling public web pages.
