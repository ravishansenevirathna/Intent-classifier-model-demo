# Intent Classifier Model

A lightweight ML-powered REST API for text intent classification. This project demonstrates an end-to-end machine learning workflow from model training to serving predictions via a Flask API.

## Overview

The Intent Classifier takes user text input and classifies it into predefined intent categories (greeting, question, complaint, praise). It's commonly used in chatbot systems, customer service automation, and NLP applications.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Client Request                       │
│                    POST /predict {"text": "..."}            │
└─────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                     Flask API (app.py)                      │
│                   Handles HTTP requests                     │
└─────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                IntentModel (intent_model.py)                │
│              Loads and runs the ML pipeline                 │
└─────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│               Scikit-learn Pipeline (pkl file)              │
│  ┌─────────────────────┐    ┌─────────────────────────────┐ │
│  │   CountVectorizer   │───▶│   MultinomialNB Classifier  │ │
│  │ (Text → Features)   │    │   (Features → Intent)       │ │
│  └─────────────────────┘    └─────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                     JSON Response                           │
│            {"intent": "complaint", "probabilities": {...}}  │
└─────────────────────────────────────────────────────────────┘
```

## Project Structure

```
Intent-classifier-model/
├── app.py                 # Flask API entry point
├── wsgi.py                # WSGI wrapper for production deployment
├── requirements.txt       # Python dependencies
├── README.md              # Project documentation
├── .gitignore             # Git ignore rules
└── model/
    ├── train.py           # Model training script
    ├── intent_model.py    # IntentModel inference class
    └── artifacts/         # Generated model files (git-ignored)
        └── intent_model.pkl
```

## Technologies

| Component | Technology |
|-----------|------------|
| ML Library | scikit-learn |
| Text Vectorization | CountVectorizer (Bag-of-Words) |
| Classifier | Multinomial Naive Bayes |
| Web Framework | Flask |
| Model Serialization | joblib |
| Production Server | Gunicorn |
| Testing | pytest |

## Quick Start

### 1. Setup Environment

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### 2. Train the Model

```bash
python model/train.py
```

This creates `model/artifacts/intent_model.pkl`.

### 3. Run the API

**Development:**
```bash
python app.py
```

**Production:**
```bash
gunicorn -w 4 -b 0.0.0.0:6000 wsgi:application
```

The API will be available at `http://127.0.0.1:6000`

## API Endpoints

### Health Check

```bash
GET /health
```

Response:
```json
{"status": "ok"}
```

### Predict Intent

```bash
POST /predict
Content-Type: application/json

{"text": "your input text here"}
```

Response:
```json
{
    "intent": "complaint",
    "probabilities": {"complaint": 0.85, "question": 0.05, ...}
}
```

## Example Usage

```bash
# Health check
curl http://127.0.0.1:6000/health

# Predict intent
curl -X POST http://127.0.0.1:6000/predict \
  -H "Content-Type: application/json" \
  -d '{"text":"I want to cancel my subscription"}'
```

## Intent Categories

The model classifies text into the following categories:

| Intent | Example |
|--------|---------|
| greeting | "hi", "hello" |
| question | "how to reset password" |
| complaint | "cancel my subscription" |
| praise | "great service" |

## Model Details

- **Algorithm**: Multinomial Naive Bayes
- **Feature Extraction**: Bag-of-Words using CountVectorizer
- **Pipeline**: Vectorizer → Classifier (scikit-learn Pipeline)
- **Serialization**: joblib pickle format

## Dependencies

See `requirements.txt`:
- Flask
- scikit-learn
- joblib
- pytest
- gunicorn
