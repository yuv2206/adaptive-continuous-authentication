# Adaptive Continuous Authentication (ACA)

Adaptive Continuous Authentication (ACA) is a zero-trust, post-login security platform that continuously verifies user identity based on keystroke behavioral dynamics. 

Traditional authentication only verifies identity at the login boundary (point-in-time), leaving active workstations vulnerable to session hijacking, token theft, and physical takeover. ACA addresses this by evaluating *how* a user types rather than *what* they type. Keystrokes are captured in 10-second sliding windows, analyzed via an unsupervised One-Class Isolation Forest model, and mapped to a continuous trust score (0–100%) that enforces dynamic access policies (`ALLOW`, `PROMPT_REAUTH`, or `LOCK_SESSION`).

---

## Key Features

- **Privacy-by-Design Key Tokenization:** Keystrokes are pseudonymized in real time using client-side HMAC-SHA256 with an ephemeral 32-byte secret. Plaintext keys are never recorded or transmitted across the network.
- **Microsecond Monotonic Timing:** Uses monotonic hardware clocks (`time.perf_counter()` / `performance.now()`) to eliminate clock drift and timing anomalies.
- **Biometric Feature Engineering:**
  - **Dwell / Hold Time:** Key press duration with active-key dictionary tracking for n-key rollover.
  - **Flight Time:** Inter-character transition latencies.
  - **Cadence & Motor Jitter:** Characters per second, hold time standard deviation, and hesitation/pause frequency (>500 ms).
- **Unsupervised Anomaly Model:** Built on Scikit-Learn `IsolationForest` (100 estimators) packaged with median imputation and standard scaling in a unified `Pipeline`. Executes inference in under 1 ms without needing prior adversary training data.
- **Inactive Window Policy:** Natural reading or thinking pauses (<5 characters per window) bypass the model, preserving session trust without triggering false lockouts.
- **Real-Time SOC Dashboard:** Zero-dependency web dashboard (Flask + native SVG) with dynamic radial trust gauges, rolling 40-window score history charts, and an accelerated replay evaluation mode.
- **Automated Test Suite:** Comprehensive unit and integration test coverage across security headers, CSRF protection, session isolation, and ML determinism.

---

## Project Structure

```text
adaptive-continuous-authentication/
├── collectors/
│   └── keyboard_logger.py       # OS-level event listener with HMAC hashing
├── data/
│   ├── processed/               # Extracted window features & benchmark data
│   └── raw/                     # Anonymized calibration recordings
├── models/
│   └── anomaly_model.pkl        # Serialized Scikit-Learn pipeline artifact
├── src/
│   ├── authentication/
│   │   ├── decision.py          # 3-tier security policy engine
│   │   └── engine.py            # Unified pipeline & session state manager
│   ├── features/
│   │   └── keystroke_features.py # Dwell/flight extraction & sliding windowing
│   ├── models/
│   │   ├── config.py            # Feature contract & model hyperparameters
│   │   ├── predict.py           # Real-time inference API wrapper
│   │   └── train_model.py       # Pipeline construction & training script
│   └── scoring/
│       └── trust_score.py       # Continuous 0-100% linear score transformation
├── static/
│   ├── dashboard.js             # Client capture engine & SVG chart renderer
│   ├── favicon.svg
│   └── style.css                # Dark-mode SOC dashboard styling
├── templates/
│   ├── dashboard.html           # Main live monitoring view
│   ├── icons.html               # SVG icon definitions
│   └── login.html               # Mock application login view
├── tests/
│   ├── test_collector.py        # Privacy & collector path traversal tests
│   └── test_dashboard.py        # API gateway, CSRF & engine integration tests
├── auth.py                      # Session authentication & cookie handling
├── engine_adapter.py            # REST API to ML pipeline adapter with caching
├── main.py                      # Flask application entry point
├── requirements.txt             # Core runtime dependencies
└── .gitignore
```

---

## Installation & Setup

### 1. Clone the repository
```bash
git clone https://github.com/yuv2206/adaptive-continuous-authentication.git
cd adaptive-continuous-authentication
```

### 2. Create and activate a virtual environment
```bash
# macOS/Linux
python3 -m venv venv
source venv/bin/activate

# Windows
python -m venv venv
venv\Scripts\activate
```

### 3. Install dependencies
```bash
pip install -r requirements.txt
```

---

## Usage

### Running the Web Dashboard
Start the local Flask application:
```bash
python main.py
```
Open your browser and navigate to `http://127.0.0.1:5000`:
- **Default Login:** Username: `anjali` | Password: `password123`
- Choose between **Live Typing Monitoring** (real-time in-browser capture) or **Replay Mode** (streams pre-recorded baseline and anomalous sessions).

### Running the Keystroke Collector
To record a new baseline typing session locally:
```bash
python collectors/keyboard_logger.py --user U01 --session S01
```
Press `ESC` at any time to end the recording session safely.

### Extracting Features & Retraining Model
```bash
# Extract 10-second behavioral window features
python -m src.features.keystroke_features

# Train and serialize the Isolation Forest model
python -m src.models.train_model
```

### Running the Test Suite
Execute the automated test suite:
```bash
python -m unittest discover tests
```

---

## Security Policy Tiers

| State | Score Range | Action | Description |
| :--- | :--- | :--- | :--- |
| **`TRUSTED`** | **$\ge 70.0\%$** | `ALLOW` | Typing behavior matches genuine user profile; unimpeded access. |
| **`VERIFY`** | **$40.0\% - 69.9\%$** | `PROMPT_REAUTH` | Slight behavioral divergence (fatigue/stress); prompts step-up MFA challenge without immediate lockout. |
| **`REJECT`** | **$< 40.0\%$** | `LOCK_SESSION` | Severe anomaly detected (unauthorized user); immediate session termination. |
| **`INACTIVE`** | **$< 5$ chars** | `HOLD` | Natural pause/idle window; holds previous trust score without penalty. |

---

## License
MIT License
