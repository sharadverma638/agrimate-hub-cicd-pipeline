# AgriMate Hub: Frontend

The farmer-facing web app for AgriMate Hub, built with React and TypeScript. It talks to the [backend](../backend/README.md) through a REST API. Docker builds and serves it, so nothing needs to be installed on your machine.

## Tech Stack

- React 18 and TypeScript
- Vite (build tool)
- Tailwind CSS and shadcn/ui components
- React Router (routing) and TanStack Query (data fetching)
- nginx (serves the built app and proxies API requests)

## Pages and Features

| Route | Page |
|---|---|
| `/` | Landing page |
| `/dashboard/agriculture` | Agriculture dashboard |
| `/dashboard/horticulture` | Horticulture dashboard |
| `/dashboard/floriculture` | Floriculture dashboard |

Each dashboard contains the feature modules: crop recommendation, disease prediction, government schemes, mandi prices, marketplace and profile.

## Project Structure

```
frontend/
├── Dockerfile              # build the app, then serve it with nginx
├── nginx.conf              # static serving and proxy rules
├── .dockerignore
├── index.html
├── vite.config.ts
├── tailwind.config.ts
├── public/
└── src/
    ├── main.tsx            # entry point
    ├── App.tsx             # routes
    ├── pages/              # Landing, Dashboard, NotFound
    ├── components/
    │   ├── features/       # CropRecommendation, DiseasePrediction, GovernmentSchemes,
    │   │                   # MandiPrice, MarketSelling, Profile
    │   └── ui/             # shadcn/ui components
    ├── contexts/           # ProfileContext
    ├── hooks/
    ├── lib/                # utilities
    └── data/               # sample data
```

## Running

The frontend runs together with the backend from the project root:

```bash
cp .env.example .env
docker compose up --build
```

Open http://localhost:8080. See the [root README](../README.md) for configuration and troubleshooting.

To rebuild only the frontend after changing its code:

```bash
docker compose up -d --build frontend
```

## How the Docker Image Works

1. **Build stage:** installs the dependencies and builds the app into static files.
2. **Serve stage:** copies those files into an unprivileged nginx image, which keeps the final image small.

The feature modules call the API with fixed `http://127.0.0.1:8000` addresses. The Dockerfile rewrites these to relative `/api/...` paths during the build. `nginx.conf` then:

- serves the app at `/`
- forwards `/api/`, `/admin/` and `/static/` to the backend
- serves uploaded images from `/media/`

This keeps everything on one address. A cleaner long-term fix is to move the API address into one shared constant in `src/lib/`.
