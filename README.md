# SafeGuard — Online Grooming Detection System

A real-time chat monitoring system that detects online grooming behaviour using a fine-tuned NLP model, emoji analysis, and image scanning. When a conversation crosses a risk threshold, SafeGuard silently swaps the child out of the chat and replaces them with an AI persona — buying time while alerting a parent or guardian by email.

---

## How It Works

Every message is scored across three signals:

| Signal | Method |
|---|---|
| **Text** | Fine-tuned DistilBERT classifier (`distilbert-base-uncased`) trained on `Combined_data.csv` |
| **Emoji** | Rule-based scorer that weights sexual, secrecy, and romantic emoji categories |
| **Image** | NudeNet detector that checks for explicit or medium-risk content |

The three scores are combined into a single **room risk score** (0.0 – 1.0) that rises cumulatively as the conversation progresses. Each risk level maps to a grooming stage:

| Risk | Stage |
|---|---|
| < 0.20 | Friendly Interaction |
| 0.20 – 0.39 | Personal Info Requests |
| 0.40 – 0.59 | Isolation & Trust Building |
| 0.60 – 0.79 | Emotional Manipulation |
| ≥ 0.80 | Exploitation Attempt |

**Sandbox mode** activates automatically when risk is high enough. Once active, an AI (powered by Llama 3.3 70B via Groq) responds to the predator on the child's behalf using cautious, deflecting language, while the real child sees a warning. The conversation winds down gracefully after 10 sandbox exchanges, then terminates.

---

## Project Structure

```
Online-Grooming-main/
├── detector.py              # ML pipeline: text cleaning, emoji scoring, image scoring, DistilBERT inference
├── app.py                   # Flask + SocketIO server: risk tracking, sandbox logic, email alerts
├── templates/
│   ├── chat.html            # Real-time chat UI (child and predator views)
│   └── dashboard.html       # Evidence and risk dashboard
├── Combined_data.csv        # Training dataset
├── safeguard_model/         # Saved DistilBERT model weights
├── safeguard_tokenizer/     # Saved tokenizer
├── safeguard_meta.pkl       # Label metadata saved at training time
├── safeguard-checkpoints/   # Training checkpoint (step 287)
├── evidence_*.json          # Auto-saved conversation evidence files
├── requirements.txt
└── testemail.py             # Utility to test SMTP alert config
```

---

## Setup

### 1. Install dependencies

```bash
pip install -r requirements.txt
pip install python-dotenv
```

### 2. Configure environment variables

Create a `.env` file in the project root:

```env
GROQ_API_KEY=your_groq_key       # Required — get a free key at https://console.groq.com
SMTP_EMAIL=you@example.com       # Optional — sender address for email alerts
SMTP_PASSWORD=your_app_password  # Optional — SMTP password / app password
PARENT_EMAIL=parent@example.com  # Optional — alert recipient
SECRET_KEY=your_flask_secret     # Optional — defaults to a dev placeholder
```

### 3. Train the model (first run only)

```bash
python detector.py --train --csv Combined_data.csv
```

### 4. Verify the model

```bash
python detector.py --test
```

### 5. Run the server

```bash
python app.py
```

Open `http://localhost:5000` for the chat UI and `http://localhost:5000/dashboard` for the evidence dashboard.

---

## Key Thresholds

| Constant | Value | Effect |
|---|---|---|
| `SCORE_THRESHOLD` | 0.65 | Minimum per-message score that contributes to room risk |
| `FLAG_THRESHOLD` | 0.72 | Per-message score that adds a visible warning badge |
| `EMAIL_THRESHOLD` | 1.0 | Room risk at which the parent email alert fires |
| `WIND_DOWN_AFTER` | 10 | Number of sandbox exchanges before chat terminates |

---

## Evidence Files

Every time sandbox mode activates, the full conversation is saved as a timestamped JSON file (`evidence_YYYYMMDD_HHMMSS.json`). Each file contains the complete message history with per-message risk scores, score breakdowns (text / emoji / image), flags, and grooming stage at the time of each message. These files are suitable for reporting to authorities or a safeguarding team.

---

## Requirements

- Python 3.8+
- PyTorch (CPU is sufficient; GPU speeds up training)
- A Groq API key (free tier works)
- SMTP credentials for email alerts (optional but recommended)
