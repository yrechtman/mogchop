# MOG/CHOP

Cranial Analysis Engine. Computational craniofacial morphometry, issued as a binary classification.

Single-page PWA. Camera capture → 468-point face mesh in-browser (MediaPipe FaceLandmarker) → 12 morphometric indices → Claude Sonnet 4.6 vision-grade qualitative layer → MOG / CHOP verdict with full lab-report dossier.

## Stack
- Static HTML + React via CDN (no build step)
- MediaPipe FaceLandmarker (`@mediapipe/tasks-vision`) — runs on-device
- Vercel serverless function for the Claude API call
- localStorage for the dossier history
- PWA manifest + service worker for installability

## Deploy
1. `npm install`
2. Set `ANTHROPIC_API_KEY` in Vercel env vars
3. `vercel --prod`

## Local
Open `index.html` via any static server. The vision layer requires `ANTHROPIC_API_KEY` exposed to `api/analyze.js` (use `vercel dev` or equivalent).
