# AgriMate Hub: Backend

The Django REST API for AgriMate Hub. It stores data, runs the machine learning models and fetches live mandi prices. The web app is in [frontend](../frontend/README.md). Docker runs it, so nothing needs to be installed on your machine.

## Tech Stack

- Python 3.10 (required by the TensorFlow version used)
- Django 5 and Django REST Framework
- scikit-learn for crop recommendation
- TensorFlow / Keras for plant disease detection
- SQLite, gunicorn and WhiteNoise
- data.gov.in API for mandi prices

## Apps

| App | Purpose | Main models |
|---|---|---|
| `users` | Accounts and farmer profiles | `User`, `FarmerProfile` |
| `crops` | Crop data and ML crop recommendation | `Crop`, `CropRecommendation` |
| `diseases` | Disease data and ML disease prediction | `Disease`, `DiseasePrediction` |
| `schemes` | Government schemes | `GovernmentScheme` |
| `mandi` | Market prices and price alerts | `MandiPrice`, `PriceAlert` |
| `market` | Produce marketplace | `ProductListing`, `ProductInquiry` |

## Machine Learning Models

| Model | File | Used by |
|---|---|---|
| Crop recommendation (Random Forest) | `model.pkl` | `crops/recommendation.py` |
| Plant disease detection (CNN) | `plant_disease_model.h5` | `diseases/prediction.py` |

Models load once at startup. If a model cannot load, the API falls back to rule-based logic. The script `train_model.py` and its dataset `Crop_recommendation.csv` are in the repository for retraining the crop model, but they are not included in the Docker image.

## Project Structure

```
backend/
├── Dockerfile
├── docker-entrypoint.sh        # applies migrations, then starts gunicorn
├── requirements.docker.txt     # runtime dependencies
├── .dockerignore
├── manage.py
├── agri_sahayak/               # settings, URLs, WSGI
├── users/  crops/  diseases/  schemes/  mandi/  market/
├── model.pkl
├── plant_disease_model.h5
└── 9ef84268-....csv            # bundled mandi price data (fallback)
```

## Running

The backend runs together with the frontend from the project root:

```bash
cp .env.example .env
docker compose up --build
```

The API is available at `http://localhost:8080/api/` through nginx. The backend has no public port by default. See the [root README](../README.md) for configuration and troubleshooting.

Create an admin user, then manage data at http://localhost:8080/admin/:

```bash
docker compose exec backend python manage.py createsuperuser
```

To rebuild only the backend after changing its code:

```bash
docker compose up -d --build backend
```

## Configuration

Settings are read from environment variables, which come from the `.env` file.

| Variable | Description |
|---|---|
| `DJANGO_SECRET_KEY` | Django secret key |
| `DJANGO_DEBUG` | `0` with Docker |
| `ALLOWED_HOSTS` | Comma-separated hostnames |
| `CSRF_TRUSTED_ORIGINS` | Comma-separated trusted origins, e.g. `http://localhost:8080` |
| `DATA_GOV_API_KEY` | Key from api.data.gov.in for live mandi prices |
| `MANDI_API_KEY` | Separate mandi key (defaults to `DATA_GOV_API_KEY`) |
| `OPENAI_API_KEY` | Optional AI fallback |
| `SQLITE_PATH` | Database file path (set by Docker Compose) |
| `MEDIA_ROOT` | Upload folder (set by Docker Compose) |

## API Endpoints

Paths are relative to `/api/`.

| Area | Endpoint | Notes |
|---|---|---|
| Users | `users/` | Register (anyone) or list (logged in) |
| | `users/me/` | Current user |
| | `users/profiles/` | Farmer profiles (own profile, or all for staff) |
| Crops | `crops/list/` | Crop catalog |
| | `crops/by_season/` | Crops filtered by season |
| | `crops/recommendations/` | `POST` soil and weather values to get a recommendation |
| Diseases | `diseases/` | Disease catalog |
| | `diseases/by_crop/` | Diseases filtered by crop |
| | `diseases/predictions/` | `POST` a leaf image to get a prediction |
| Schemes | `schemes/` | Government schemes |
| | `schemes/by_category/`, `schemes/by_state/` | Filtered lists |
| Mandi | `mandi/prices/` | Stored prices |
| | `mandi/prices/latest/`, `mandi/prices/statistics/` | Latest prices and summary |
| | `mandi/alerts/` | Price alerts (logged in) |
| | `mandi/proxy/` | Live prices from data.gov.in |
| Market | `market/listings/` | Produce listings |
| | `market/inquiries/` | Buyer inquiries |
| | `market/mandi/` | Market price lookup |

Most endpoints can be read without logging in. Creating or changing data needs a logged-in user (session authentication), except account registration and disease prediction, which are open. Log in at `/admin/` and the browser session also works on the API pages.

### Example requests

```bash
# List government schemes
curl http://localhost:8080/api/schemes/

# Disease prediction (image upload)
curl -X POST http://localhost:8080/api/diseases/predictions/ \
  -F "plant_image=@leaf.jpg"
```

Field names for the crop recommendation request body are defined in `crops/serializers.py`.

## Management Commands

Run them inside the container:

```bash
docker compose exec backend python manage.py test_crop_model
docker compose exec backend python manage.py create_sample_listings
```

| Command | Purpose |
|---|---|
| `test_crop_model` | Check that the crop model loads and predicts |
| `create_sample_listings` | Add sample marketplace listings |

## Docker Image

- Multi-stage build on `python:3.10-slim`, with dependencies installed into a virtual environment.
- Runs as a non-root user with gunicorn (1 worker, 4 threads, to keep model memory low).
- Static files are collected at build time and served by WhiteNoise.
- The database and uploads live in the `/data` volumes, so they survive restarts.
