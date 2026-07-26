# Deploying persistently on a Lightning AI Studio

This app is currently reachable only through the Studio's temporary port-forward URL
(the `<port>-<studio-id>.cloudspaces.litng.ai` link you get from "Port active"). That
link is ephemeral — it depends on the Studio's live tunnel and isn't meant for
sharing as a stable app URL.

Lightning Studios also let you **host web apps directly** with a stable public URL and
serverless auto-start/stop, instead of raw port forwarding. That's what this runbook
sets up: one persistent app for the FastAPI backend, one for the Next.js frontend.

Reference: https://lightning.ai/docs/platform/build/host-web-apps

## Why two steps, in this order

The frontend bakes `NEXT_PUBLIC_API_URL` into its production build at **build time**
(`frontend/app/paper2notebook/page.tsx`: `process.env.NEXT_PUBLIC_API_URL`). It cannot
be changed at runtime afterward. So the backend must be deployed first to get its
stable URL, and the frontend must be (re)built after that URL is known.

## 1. Deploy the backend

1. In the Studio, open the app-hosting/plugin panel and add an app for
   `paper-to-notebook/backend`, using the existing start command from
   `backend/Procfile`:
   ```
   uvicorn app:app --host 0.0.0.0 --port $PORT
   ```
2. Set the backend's environment variables (`LIGHTNING_API_KEY`, `LIGHTNING_MODEL`,
   `MAX_UPLOAD_MB`; `GITHUB_TOKEN` is optional now — see `backend/.env.example`).
3. Enable serverless/auto-start so the app sleeps when idle and wakes on the first
   request (expect a short cold-start delay after idle periods).
4. Note the **stable public URL** Lightning assigns to this app — this replaces the
   old `NNNN-<studio-id>.cloudspaces.litng.ai` port-forward link.

## 2. Point the frontend at it and rebuild

1. Update `frontend/.env.local`:
   ```
   NEXT_PUBLIC_API_URL=https://<the-stable-backend-url-from-step-1>
   ```
2. Rebuild the production bundle so the URL is baked in:
   ```bash
   cd frontend
   npm install
   npm run build
   ```

## 3. Deploy the frontend

1. Add a second app in the Studio for `paper-to-notebook/frontend`, started with:
   ```
   npm run start
   ```
   (Lightning's docs recommend the React/Vue pattern for production UIs like this
   one, treating the backend as an independent API — which is exactly this app's
   shape.)
2. Enable serverless/auto-start here too if you want it to idle when unused.
3. This app's stable URL is what you share/use going forward, instead of the old
   port-forward link.

## Cost note

Lightning's docs mention that a Studio only stays fully free while **at most one**
app is actively running at a time — running backend + frontend simultaneously as two
always-on apps in the same Studio may incur cost for the second one. If that matters,
either accept the cost, or split backend and frontend across two separate Studios.

## Re-deploying after backend URL changes

Any time the backend's stable URL changes (e.g. redeployed to a new Studio), repeat
step 2 (`.env.local` + `npm run build`) and redeploy the frontend app — the old build
will keep pointing at the stale URL until you do.
