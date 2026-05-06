<div align="center">

# AgriFuture Platform

### AI-Powered Agricultural Intelligence Platform

[![Live Demo](https://img.shields.io/badge/Live_Demo-Render-46E3B7?style=for-the-badge&logo=render&logoColor=white)](https://final-year-pjf9.onrender.com/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.8-3178C6?style=for-the-badge&logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![React](https://img.shields.io/badge/React-19-61DAFB?style=for-the-badge&logo=react&logoColor=black)](https://react.dev/)
[![Node.js](https://img.shields.io/badge/Node.js-Express_5-339933?style=for-the-badge&logo=node.js&logoColor=white)](https://nodejs.org/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-18-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Gemini](https://img.shields.io/badge/Google_Gemini-AI-4285F4?style=for-the-badge&logo=google&logoColor=white)](https://ai.google.dev/)

</div>

---

## What This Project Demonstrates

- **Multi-module AI platform architecture** — 6 AI modules (crop recommendation, disease detection, drone terrain analysis, market forecasting, digital twin simulation, conversational AI) behind a single Express 5 server, each backed by Google Gemini with different prompting strategies.
- - **Production security implementation** — JWT httpOnly cookies, bcrypt password hashing (10 salt rounds), OTP auth via both email (Nodemailer/Gmail) and SMS (MSG91), rate limiting (5 OTP/15min, 10 login/15min per IP), Helmet security headers, Zod schema validation on all inputs.
  - - **Real-time data integration** — 27+ crop market prices via live mandi API, weather data via Open-Meteo (no API key), monsoon tracking, and photosynthesis index — all hydrated on the client without polling.
    - - **Full-stack TypeScript discipline** — 44+ TypeScript interfaces, 56 `.tsx`/`.ts` files, Zod validation front-to-back (same schemas on client and server), strict type safety throughout 12,500+ lines.
      - - **PWA with offline capability** — cache-first service worker, Web App Manifest, install prompts. Works offline for core features.
        - - **CI/CD and deployment** — GitHub Actions pipeline, `render.yaml` for one-click Render deploy, `railway.json` for Railway, PostgreSQL managed by Render.
         
          - ---

          ## Architecture

          ```
          Browser (React 19 + TypeScript + Vite + TailwindCSS)
          │
          │ HTTPS / REST API
          ▼
          Express 5 Server (Node.js + TypeScript, ESM)
          ├── routes/auth.ts        JWT auth, OTP (Email + SMS), rate limiting
          ├── routes/predictions.ts AI crop & disease endpoints (Gemini)
          ├── routes/payments.ts    Razorpay order creation & HMAC verification
          ├── routes/market.ts      Mandi price data (27+ crops)
          ├── routes/reports.ts     Scan history CRUD
          └── routes/expenses.ts    Farm expense CRUD
          │
          ├── PostgreSQL (Render)        Users, reports, expenses, orders, activity log
          ├── Google Gemini 2.5 Flash    Crop, disease, drone, market, digital twin
          ├── Google Gemini 3 Flash      AgriBot conversational AI
          ├── Razorpay API               Payment processing + HMAC signature verification
          ├── MSG91 API                  SMS OTP delivery
          ├── Nodemailer                 Email OTP (Gmail SMTP)
          └── Open-Meteo API             Live weather (no API key needed)
          ```

          ---

          ## Platform Modules (6 AI Systems)

          | Module | Technology | What It Does |
          |--------|-----------|-------------|
          | **Crop Recommendation** | Gemini 2.5 Flash | Soil parameters → AI recommendations with confidence scores + fertilizer plans |
          | **Disease Detector** | Gemini 2.5 Flash | Image upload → diagnosis (severity Low/Moderate/High), pathogen type, treatment |
          | **AgriDrone V2** | Gemini 2.5 Flash | Satellite terrain → NDVI, irrigation mapping, soil health gauge (0–100) |
          | **Market Forecasting** | Gemini 2.5 Flash | 30-day price predictions, sentiment gauge, profit calculator, live mandi ticker |
          | **Digital Twin** | Gemini 3 Flash | Real-time field parameter modeling via visual node network |
          | **AgriBot** | Gemini 3 Flash | Streaming conversational AI with farming FAQ and context-aware guidance |

          ---

          ## Key Features

          | Domain | Features |
          |--------|----------|
          | **Live Data** | Mandi prices (27+ crops), weather dashboard, monsoon sync, Open-Meteo integration |
          | **Security** | JWT httpOnly, bcrypt, OTP (email + SMS), rate limiting, Helmet, Zod validation |
          | **Commerce** | AgriStore (58+ products), Razorpay integration with HMAC verification |
          | **Analytics** | Expense tracker, report history, bar charts, sparklines, admin dashboard |
          | **UX** | 7 Indian languages, dark/light mode, PWA, Cmd+K command palette, Framer Motion |
          | **Infrastructure** | Render/Railway deploy configs, PostgreSQL, GitHub Actions CI/CD |

          ---

          ## Tech Stack

          **Frontend:** `React 19` `TypeScript 5.8` `Vite 6` `TailwindCSS 4` `Framer Motion 12` `Zod 4`

          **Backend:** `Node.js` `Express 5` `PostgreSQL` `JWT` `bcryptjs` `Nodemailer` `MSG91` `Razorpay` `Multer`

          **AI:** `Google Gemini 2.5 Flash` `Google Gemini 3 Flash`

          **DevOps:** `GitHub Actions` `Render` `Railway` `Vitest` `Supertest`

          ---

          ## Engineering Highlights

          **Security is not bolted on.** Rate limiting, Zod schema validation, Helmet security headers, and httpOnly JWT cookies are enforced at the middleware layer — not as afterthoughts inside route handlers. OTP generation uses cryptographically random codes with 15-minute TTL stored in PostgreSQL (not in-memory, so server restarts don't invalidate active OTPs).

          **Gemini prompt engineering.** Each AI module uses a different prompting strategy — structured JSON extraction for crop recommendations, image + text multi-modal for disease detection, coordinate-to-satellite-description for drone analysis. The same Gemini API key routes to 6 distinct intelligence layers via `geminiService.ts`.

          **HMAC payment verification.** Razorpay webhook signatures are verified server-side before orders are committed to the database — the pattern used in production payment systems to prevent replay attacks and forged completion events.

          **TypeScript end-to-end.** 44+ interfaces in `types.ts` are shared between client and server. Zod schemas validate the same shape at the API boundary and in the frontend form layer, eliminating the class of bugs where the client sends a shape the server doesn't expect.

          ---

          ## Quick Start

          ```bash
          git clone https://github.com/UTKARSH698/agrifuture-platform
          cd agrifuture-platform
          npm install

          # Configure environment
          cp .env.local.example .env
          # Required: DATABASE_URL, VITE_GEMINI_API_KEY, JWT_SECRET
          # Optional: EMAIL_USER/PASS (OTP), MSG91 keys (SMS), RAZORPAY keys (payments)

          # Development
          npm run dev        # → http://localhost:3000

          # Production build
          npm run build && npm start
          ```

          **Deploy to Render (one-click):** Fork → connect to Render → add PostgreSQL plugin → set env vars → deploy. The `render.yaml` handles all service config automatically.

          ---

          ## Live Demo

          [**https://final-year-pjf9.onrender.com**](https://final-year-pjf9.onrender.com/) — deployed on Render free tier (cold start ~30s on first load).

          ---

          ## Known Limitations & Honest Tradeoffs

          - Gemini API latency: AI modules take 1–3s per request; no caching layer for AI responses
          - - Single-region PostgreSQL: no read replicas or connection pooling configured
            - - Demo mode: some AI features fall back to mock data when Gemini quota is reached
              - - SMS OTP requires MSG91 account (email OTP works without extra config)
               
                - For scaling analysis, architectural decisions, and production upgrade path, see **[TECHNICAL.md](TECHNICAL.md)**.
               
                - ---

                *MIT License · Utkarsh Batham*
