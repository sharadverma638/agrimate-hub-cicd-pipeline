# AgriMate Hub

AgriMate Hub is an AI-powered web platform for Indian farmers. It brings crop recommendation, plant disease detection, live mandi prices, government schemes and a produce marketplace into one application.

## Features

- **Crop recommendation:** suggests the most suitable crop from soil nutrients (N, P, K), temperature, humidity, pH and rainfall using a Random Forest model.
- **Disease detection:** predicts the plant disease from an uploaded leaf image using a CNN model, with a confidence score.
- **Mandi prices:** current market prices from the data.gov.in open API, with a bundled CSV fallback and price alerts.
- **Government schemes:** subsidies and programs, filterable by category and state.
- **Marketplace:** farmers list produce with photos and receive buyer inquiries.
- **Dashboards:** separate Agriculture, Horticulture and Floriculture sections.

If a model or external API is unavailable, the backend falls back to rule-based logic instead of failing.

## Tech Stack

- **Frontend:** React, TypeScript, Vite, Tailwind CSS, shadcn/ui
- **Backend:** Python 3.10, Django 5, Django REST Framework, gunicorn
- **Machine learning:** scikit-learn (crop model), TensorFlow / Keras (disease model)
- **Database:** SQLite, stored in a Docker volume
- **Deployment:** Docker, Docker Compose, nginx

## How It Works

The app is served from one address. nginx routes each request, and the backend is not exposed publicly.

```
Browser ──► http://localhost:8080  (nginx)
              ├─ /          → React app
              ├─ /api/      → Django REST API
              ├─ /admin/    → Django admin
              ├─ /static/   → Django static files
              └─ /media/    → uploaded images
```

## Project Structure

```
agrimate-hub/
├── docker-compose.yml           # runs frontend and backend together
├── .env.example                 # environment template (copy to .env)
├── .gitignore
├── backend/                     # Django API and ML models
│   ├── Dockerfile
│   ├── docker-entrypoint.sh     # applies migrations, then starts the server
│   ├── requirements.docker.txt  # runtime dependencies
│   ├── agri_sahayak/            # settings and URL routing
│   ├── users/  crops/  diseases/  schemes/  mandi/  market/
│   ├── model.pkl                # crop recommendation model
│   └── plant_disease_model.h5   # disease detection model
└── frontend/                    # React app
    ├── Dockerfile
    └── nginx.conf               # static serving and reverse proxy rules
```

More detail: [backend/README.md](backend/README.md) and [frontend/README.md](frontend/README.md).

## Getting Started

You need [Docker](https://docs.docker.com/get-docker/) with Docker Compose.

1. **Create the environment file** and set `DJANGO_SECRET_KEY` in it:

   ```bash
   cp .env.example .env
   ```

   To generate a key:

   ```bash
   python3 -c "import secrets; print(secrets.token_urlsafe(50))"
   ```

2. **Build and start** (the first build takes a few minutes):

   ```bash
   docker compose up --build
   ```

3. **Open the app** at http://localhost:8080. The backend needs about a minute to load the ML models.

4. **Create an admin user** (one time):

   ```bash
   docker compose exec backend python manage.py createsuperuser
   ```

## Configuration

Set these in `.env`, then run `docker compose up -d` to apply changes.

| Variable | Description |
|---|---|
| `DJANGO_SECRET_KEY` | Django secret key (required) |
| `DJANGO_DEBUG` | `0` with Docker, `1` only for debugging |
| `ALLOWED_HOSTS` | Allowed hostnames, e.g. `localhost,127.0.0.1,yourdomain.com` |
| `CSRF_TRUSTED_ORIGINS` | Address of the app, e.g. `http://localhost:8080` |
| `DATA_GOV_API_KEY` | Optional key from [api.data.gov.in](https://api.data.gov.in/) for live mandi prices |
| `MANDI_API_KEY` | Optional separate mandi key (defaults to `DATA_GOV_API_KEY`) |
| `OPENAI_API_KEY` | Optional AI fallback |

Never commit `.env` or real API keys.

## Pages and URLs

| Page | URL |
|---|---|
| Landing page | http://localhost:8080/ |
| Dashboards | `/dashboard/agriculture`, `/dashboard/horticulture`, `/dashboard/floriculture` |
| Admin panel | http://localhost:8080/admin/ |
| API | http://localhost:8080/api/ |

Use the admin panel to add, edit and delete data. After you log in there, the browser session also works on the API pages.

## API Overview

| Area | Endpoints |
|---|---|
| Users | `/api/users/`, `/api/users/me/`, `/api/users/profiles/` |
| Crops | `/api/crops/list/`, `/api/crops/recommendations/` |
| Diseases | `/api/diseases/`, `/api/diseases/predictions/` |
| Schemes | `/api/schemes/` |
| Mandi | `/api/mandi/prices/`, `/api/mandi/alerts/`, `/api/mandi/proxy/` |
| Market | `/api/market/listings/`, `/api/market/inquiries/` |

Most endpoints can be read without logging in. Creating or changing data needs a logged-in user. The full list is in [backend/README.md](backend/README.md).

## Useful Commands

| Task | Command |
|---|---|
| Start in the background | `docker compose up -d --build` |
| Stop | `docker compose down` |
| View logs | `docker compose logs -f` |
| Back up the database | `docker compose cp backend:/data/db/db.sqlite3 ./db.sqlite3` |
| Stop and delete the database and uploads | `docker compose down -v` |

## Troubleshooting

| Problem | Solution |
|---|---|
| `.env` not found | Run `cp .env.example .env`. |
| Port 8080 in use | Change `"8080:8080"` to `"8081:8080"` in `docker-compose.yml`. |
| `/admin/` shows a "not found" page | Run `docker compose build --no-cache frontend && docker compose up -d`. |
| Admin or API pages have no styling | Run `docker compose up -d --build backend`. |
| `DisallowedHost` error | Add your hostname to `ALLOWED_HOSTS` in `.env`. |
