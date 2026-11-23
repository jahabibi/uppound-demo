# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

UpPound is a demo pet-adoption app used to help new users through their Upbound journey. It has three parts: a React frontend, a Node.js/Express backend, and a PostgreSQL database. There is no infrastructure-as-code in this repo; it holds only the application and its container definitions.

## Commands

Run the full stack (frontend on http://localhost:3000, backend on host port 3002, Postgres on 5432):

```sh
docker compose up --build
```

`init-scripts/init.sql` (schema and seed pets) runs only when the `postgres_data` volume is first created. After changing it, reset with `docker compose down -v`.

Frontend (Create React App, run from `frontend/`):

```sh
npm start                                  # dev server
npm run build
npm test                                   # Jest in watch mode
CI=true npm test -- --watchAll=false       # single run
npm test -- -t "test name"                 # single test by name
```

The only test, `src/App.test.js`, is the unmodified CRA boilerplate ("learn react") and fails against the current UI. Linting is CRA's built-in ESLint (`react-app` config), which runs during `start` and `build`.

Backend (run from `backend/`): `npm start`. There are no tests or lint config. Outside Docker it needs `DATABASE_URL` set, and it needs Node 15 or newer because `pg` 8.14 requires `util/types`.

## Architecture

- **Request path:** the browser calls relative `/api/...` URLs (hardcoded as `'/api'` in `PetList.js`, `PetCard.js` and `PetDetailPage.js`; `REACT_APP_API_URL` is set in compose and the Dockerfile but never read). In the container, nginx (`frontend/nginx.conf`) serves the built SPA, proxies `/api/` to `http://backend:3001/api/`, and falls back to `index.html` for client routes. The CRA dev server has no proxy configured, so `/api` calls fail under `npm start` unless you add one.
- **Backend:** everything is in `backend/server.js`. The `pg` pool is created by `connectWithRetry` (10 attempts, 5s apart) before `listen`. Routes:
  - `GET /api/pets` and its alias `/pets`
  - `GET /api/pets/:id`
  - `POST /api/pets` (multer upload into `uploads/`)
  - `GET` and `POST /api/adoptions`
  - `GET /api/pets/search?query=&species=`, which must stay registered before `/api/pets/:id`
  - `/health` and `/debug-pets`

  A Socket.IO server relays `new_adoption` events as `adoption_update`; the frontend doesn't use it. The backend Dockerfile copies only `server.js`.
- **Database:** two tables, `pets` and `adoptions` (FK `pet_id`). Species values in the seed are `Dog`, `Cat` and `Small`. The frontend filters by species on the client after fetching all pets, and does not call the search endpoint.
- **Frontend:** React 19 with react-router-dom v7. There are two routes: `/` (`HomePage` → `PetList` → `PetCard`) and `/pets/:id` (`PetDetailPage`). The detail page's "adopt" CTA links to upbound.io by design; adoption forms are not wired up.

## Pet images

Pet photos are static frontend assets in `frontend/public/images/`. Seed rows store `image_url` as `/images/<name>.jpg`, and the frontend uses that absolute path directly as the `<img src>`, so nginx (or the CRA dev server) serves it. To add a pet, put the photo in that directory and reference it as `/images/<file>` in `init.sql`. Images uploaded through `POST /api/pets` get an `image_url` of `/uploads/...`. The backend serves those at `/uploads`, but nginx doesn't proxy that path, so uploaded images won't display through the frontend.
