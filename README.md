# HTTP Tarpit & Bot Analyzer

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.19631120.svg)](https://doi.org/10.5281/zenodo.19631120)
[![Python Version](https://img.shields.io/badge/python-3.11%2B-blue.svg)](https://www.python.org/downloads/)
[![Poetry](https://img.shields.io/endpoint?url=https://python-poetry.org/badge/v0.json)](https://python-poetry.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A high-performance asynchronous HTTP tarpit for monitoring automated scanning campaigns, collecting behavioral data, and implementing active defense mechanisms.

This tool complements the [Multi-threaded SSH Honeypot](https://github.com/boykoatwork/honeypot) by providing a lightweight active-defense layer. While the SSH honeypot captures post-authentication payload delivery, the tarpit focuses on large-scale automated scanning and reconnaissance, offering a broader view of the botnet lifecycle.

---

## Features

- **Asynchronous Engine** — built with `aiohttp` to handle high-concurrency connections with minimal resource overhead.
- **Active Defense** — implements resource exhaustion (tarpit) by draining attacker connections with slow, drip-feed responses.
- **Forensic Logging** — captures detailed HTTP metadata including full request headers, User-Agent strings, and session duration in structured JSON format.
- **GeoIP Enrichment** — integrated MaxMind GeoLite2 support for real-time ASN and geographic mapping of source IPs.
- **Structured Storage** — SQLite-backed persistence for reliable event storage and post-hoc query analysis.
- **AbuseIPDB Integration** — optional automated reporting of detected malicious IPs with configurable rate-limiting.

---

## Architecture Overview

```
Internet ──► [Reverse Proxy / iptables REDIRECT]
                        │
                        ▼
              HTTP Tarpit (aiohttp)
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
    SQLite DB      GeoIP Lookup   AbuseIPDB API
  (events log)    (MaxMind DB)    (optional)
```

The tarpit listens on a configurable address and port. All incoming requests are accepted and held open while the server drip-feeds a slow response, exhausting scanner thread pools. Every connection is fully logged before and after the tarpit cycle.

---

## Requirements

- Python 3.11 or newer
- [Poetry](https://python-poetry.org/docs/#installation) dependency manager
- (Optional) MaxMind GeoLite2 databases (`GeoLite2-City.mmdb`, `GeoLite2-ASN.mmdb`)
- (Optional) AbuseIPDB API key

---

## Installation

### 1. Clone the repository

```bash
git clone https://github.com/t1a0/http-tarpit.git
cd http-tarpit
```

### 2. Install dependencies

```bash
poetry install --without analysis
```

To include the data analysis toolset (pandas, matplotlib, seaborn, folium):

```bash
poetry install
```

### 3. Configure environment variables

Create a `.env` file in the project root:

```env
# Optional: enable AbuseIPDB reporting (leave unset to disable)
ABUSEIPDB_API_KEY=your_api_key_here
```

All other parameters are configured directly in `src/http_tarpit/config.py`.

### 4. Configure tarpit parameters (optional)

Edit `src/http_tarpit/config.py` to adjust the core settings:

| Parameter | Default | Description |
|---|---|---|
| `HOST` | `127.0.0.1` | Bind address |
| `PORT` | `8080` | Bind port |
| `RESPONSE_DELAY_SECONDS` | `1.5` | Delay between drip-feed chunks |
| `RESPONSE_CHUNK` | `b'.'` | Payload sent per chunk |
| `MAX_RESPONSE_BYTES` | `1200` | Maximum bytes per connection before close |
| `ABUSEIPDB_REPORT_INTERVAL_MINUTES` | `40` | Minimum interval between reports for the same IP |

### 5. Set up GeoIP databases (optional)

1. Register for a free MaxMind account at [maxmind.com](https://www.maxmind.com/en/geolite2/signup).
2. Download `GeoLite2-City.mmdb` and `GeoLite2-ASN.mmdb`.
3. Place both files in the `data/` directory (created automatically on first run).

GeoIP enrichment is automatically disabled if the database files are not present.

---

## Running the Tarpit

### Development / foreground

```bash
poetry run python main.py
```

### Production — systemd service

Create `/etc/systemd/system/http-tarpit.service`:

```ini
[Unit]
Description=HTTP Tarpit & Bot Analyzer
After=network.target

[Service]
Type=simple
User=tarpit
WorkingDirectory=/opt/http-tarpit
ExecStart=/opt/http-tarpit/.venv/bin/python main.py
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now http-tarpit
sudo systemctl status http-tarpit
```

### Production — redirect traffic with iptables

To redirect traffic from common scan targets (e.g., port 80, 8888) to the tarpit without running as root:

```bash
# Redirect port 80 to tarpit on port 8080
sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 8080

# Redirect additional ports
sudo iptables -t nat -A PREROUTING -p tcp --dport 8888 -j REDIRECT --to-port 8080
```

The tarpit reads the `X-Tarpit-Target-Port` header to record the originally targeted port in the database. Set this header in your reverse proxy configuration if using nginx or HAProxy as a front-end.

### Production — nginx reverse proxy (optional)

```nginx
server {
    listen 80 default_server;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Tarpit-Target-Port $server_port;
        proxy_read_timeout 3600s;
    }
}
```

---

## Output & Logs

| Path | Contents |
|---|---|
| `logs/tarpit.log` | Structured JSON log of all events |
| `data/tarpit_events.db` | SQLite database with full event records |

The SQLite database is created automatically on first run. The schema includes GeoIP fields, AbuseIPDB reporting status, full request metadata, and session timing.

---

## Data Usage & Research

This project generates structured datasets suited for threat intelligence research, botnet activity analysis, and behavioral classification studies. The `tarpit_events.db` database supports complex SQL queries for identifying scanning trends, injection attempts, and geographic distribution of malicious actors.

The dataset generated by this tarpit is available on Zenodo:

**[DOI: 10.5281/zenodo.19631120](https://doi.org/10.5281/zenodo.19631120)**

All published datasets are pre-processed for forensic fidelity, including IP address sanitization and exclusion of operational reporting metadata.

### Reporting Policy & Data Integrity

The integrated AbuseIPDB module operates without ASN-based whitelisting. This provides utility for internal security monitoring but can produce false positives for high-frequency legitimate crawlers (e.g., search engines, security research scanners).

**ASN-based filtering or User-Agent whitelisting is strongly recommended before enabling `ABUSEIPDB_API_KEY` in production.**

If generating datasets for publication, reporting metadata should be excluded. Automated reports may propagate inaccurate reputation signals to the community.

---

## Tech Stack

| Component | Technology |
|---|---|
| Language | Python 3.11+ |
| Async runtime | `asyncio`, `aiohttp` |
| Dependency management | Poetry |
| Configuration | `python-dotenv` |
| Geolocation | `geoip2` (MaxMind GeoLite2) |
| Database | SQLite (`sqlite3`) |

---

## Citation

If you use this framework or the associated dataset in your research, please cite:

```bibtex
@software{boiko_2026_tarpit_git,
  author       = {Boiko, Viktor and Spesivtsev, Mykola},
  title        = {HTTP Tarpit & Bot Analyzer},
  month        = jun,
  year         = 2026,
  publisher    = {GitHub},
  version      = {v1.0.0},
  url          = {https://github.com/boykoatwork/http-tarpit}
}
```

---

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

---

## Authors

[Viktor Boiko](https://orcid.org/0000-0001-5929-657X) — [@boykoatwork](https://github.com/boykoatwork)

[Mykola Spesivtsev](https://orcid.org/0009-0007-5640-5241) — [@t1a0](https://github.com/t1a0)
