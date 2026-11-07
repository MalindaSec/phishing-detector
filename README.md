# Phishing Detector

A Flask web application for AI-assisted phishing URL detection. The app combines machine-learning predictions with deterministic URL and homoglyph heuristics to classify suspicious links through a browser UI and JSON API.

## Main Capabilities

- Single URL analysis (`/analyze`)
- Batch URL analysis (`/batch-analyze`)
- Model loading from tracked artifact `phishing_model.pkl` (~23.8 MB in repository) with rule-based fallback if loading fails
- URL feature extraction (structure, protocol, subdomain depth, character patterns, etc.)
- Homoglyph and IDN-style spoofing checks (including `xn--` handling)
- Suspicious TLD and embedded-brand detection
- Obfuscation checks (for example `@` and excessive `%` encoding)
- Final risk verdict labeling: `LOW_RISK`, `MEDIUM_RISK`, `HIGH_RISK`

## Features

- Flask frontend + API in one app (`app.py`)
- Interactive UI for single and batch scans (`templates/index.html`, `static/js/script.js`)
- Risk details including:
  - ML output (`is_phishing`, `probability`, `confidence`, `verdict`)
  - Homoglyph analysis (count, normalized URL, suspicious character indicators)
  - Critical extracted feature flags
- Deployment-ready configuration for Railway (`railway.toml`)

## Repository Structure

```text
phishing-detector/
├── app.py
├── phishing_model.pkl
├── requirements.txt
├── runtime.txt
├── railway.toml
├── templates/
│   └── index.html
└── static/
    ├── css/
    │   └── style.css
    └── js/
        └── script.js
```

## Prerequisites

- Python 3.11 (repository runtime is `python-3.11.0`)
- `pip`

## Local Installation

```bash
git clone https://github.com/MalindaSec/phishing-detector.git
cd phishing-detector
python -m venv .venv
source .venv/bin/activate  # Windows (PowerShell): .venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

## Running the App

### Flask development run

```bash
python app.py
```

`app.py` reads `PORT` from the environment and defaults to `5000` when `PORT` is not set.

### Gunicorn run

```bash
gunicorn app:app
```

For production platforms (such as Railway), ensure the runtime port is provided via environment/platform settings.

## Web UI

Open `http://localhost:5000` (or your configured port) to use:

- **Single URL** tab: analyze one URL
- **Batch Scan** tab: analyze multiple URLs at once

## HTTP API

### `POST /analyze`

Analyze one URL.

Request body:

```json
{"url":"https://example.com"}
```

Behavior notes:
- Returns `400` when `url` is missing/empty.
- Adds `https://` automatically if protocol is omitted.

Typical success response fields:

- `url`
- `final_verdict` (`LOW_RISK`, `MEDIUM_RISK`, `HIGH_RISK`)
- `ml_result`:
  - `is_phishing` (boolean)
  - `probability` (float)
  - `confidence` (`LOW`/`MEDIUM`/`HIGH`)
  - `verdict` (`PHISHING`/`LEGITIMATE`)
- `homoglyph_analysis`:
  - `homoglyph_count`
  - `suspicious_chars`
  - `multi_char_homoglyphs`
  - `normalized_url`
  - `homoglyph_risk_score`
  - `is_suspicious_homoglyph`
- `critical_features`:
  - `IsLegitimateDomain`
  - `EmbeddedBrandName`
  - `HasSuspiciousTLD`
  - `HomoglyphScore`
  - `HasAtObfuscation`
  - `HasExcessiveEncoding`

### `POST /batch-analyze`

Analyze multiple URLs.

Request body:

```json
{"urls":["https://example.com","https://example.org"]}
```

Behavior notes:
- Returns `400` when `urls` is missing or empty.
- Each non-empty item is normalized to include protocol if omitted.

Success response shape:

```json
{
  "results": [
    {
      "url": "https://example.com",
      "final_verdict": "LOW_RISK",
      "ml_result": {"is_phishing": false, "probability": 0.2, "confidence": "LOW", "verdict": "LEGITIMATE"},
      "homoglyph_analysis": {"homoglyph_count": 0, "suspicious_chars": [], "multi_char_homoglyphs": [], "normalized_url": "https://example.com", "homoglyph_risk_score": 0.0, "is_suspicious_homoglyph": false},
      "critical_features": {"IsLegitimateDomain": 0, "EmbeddedBrandName": 0, "HasSuspiciousTLD": 0, "HomoglyphScore": 0, "HasAtObfuscation": 0, "HasExcessiveEncoding": 0}
    }
  ]
}
```

## Analysis Pipeline (High Level)

1. Parse and normalize URL input.
2. Run homoglyph/IDN-style analysis (`EnhancedHomoglyphDetector`).
3. Extract URL features (`ImprovedURLFeatureExtractor`) including brand embedding, suspicious TLDs, obfuscation, and structural indicators.
4. Run ML inference from loaded model; if unavailable, use rule-based fallback.
5. Combine ML output and heuristic overrides to assign final risk label:
   - `LOW_RISK`
   - `MEDIUM_RISK`
   - `HIGH_RISK`

## Deployment Notes (Railway)

- `railway.toml` uses `nixpacks` and starts with:
  - `gunicorn app:app`
- `runtime.txt` pins Python runtime to `python-3.11.0`.
- Railway should provide the runtime port/environment expected by the deployment.

## Limitations & Security Disclaimer

- This project is a heuristic/ML decision aid, **not** a guarantee of safety.
- Do not open suspicious links just because they score low risk.
- The checked-in pickle artifact (`phishing_model.pkl`) should only be loaded from a trusted source.
- Analysis is based on URL string inspection; it does **not** fetch, execute, or browse target websites.

## Development & Contribution

1. Create a feature branch.
2. Make focused changes.
3. Validate behavior locally.
4. Open a pull request for review.

## License

No license file or explicit license declaration is currently present in this repository.
