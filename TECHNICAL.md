# AgriFuture Platform — Technical Architecture

Deep engineering documentation for the AgriFuture agricultural intelligence platform. The README covers what the platform does and how to run it. This document covers architectural decisions, security implementation, scaling constraints, and production upgrade path.

---

## Multi-Module AI Architecture

### Why Google Gemini for All 6 Modules?

AgriFuture uses a single AI provider (Google Gemini) across all 6 intelligence modules with different model versions and prompting strategies per module:

- **Gemini 2.5 Flash** — used for structured extraction tasks (crop recommendation, disease detection, drone analysis, market forecasting) where output must follow a schema
- **Gemini 3 Flash** — used for conversational tasks (AgriBot, digital twin) where response diversity is more important than strict structure

This single-provider approach has a known tradeoff: all AI functionality fails simultaneously if Gemini is unavailable. A multi-provider architecture (Gemini + Claude + OpenAI with automatic failover) would improve availability but significantly increase operational complexity.

### Prompting Strategies by Module

Each module uses a different prompting approach. This is not just parameter tuning — the prompt structure determines whether the output is useful:

**Crop Recommendation:** Structured extraction prompt. Gemini is asked to return JSON with `recommendation`, `confidence_score`, `fertilizer_plan`, and `alternative_crops`. A free-form response is harder to parse and display in the UI.

**Disease Detection:** Multi-modal prompt (image + text). The image bytes are sent directly to Gemini with a text description of what to detect. The prompt includes severity classification criteria (Low/Moderate/High) to ensure consistent output.

**AgriDrone V2:** Coordinate-to-satellite-description prompt. Latitude/longitude coordinates are sent with a request to describe the terrain, vegetation, and irrigation sources. Gemini generates terrain descriptions from its training data about geographic features at those coordinates.

**Market Forecasting:** Historical + current price data is serialised and sent with a request for 30-day forecast, sentiment analysis, and profit calculations. The model performs numerical reasoning over the provided data.

**AgriBot:** Conversational prompt with farming domain system message. Prior turns are included for multi-turn coherence.

**Digital Twin:** Real-time sensor parameter prompt. Field parameters (soil moisture, temperature, humidity) are sent and Gemini generates a simulation response describing projected crop behaviour.

---

## Security Implementation

### Authentication Stack

```
Registration flow:
  email/phone → bcrypt hash → PostgreSQL users table
    OTP generation → cryptographically random 6-digit code
      OTP storage → PostgreSQL with 15-minute TTL (not in-memory)
        OTP delivery → Nodemailer (Gmail SMTP) or MSG91 (SMS)

        Login flow:
          credentials → bcrypt.compare → JWT creation (7-day expiry)
            JWT → httpOnly cookie (not localStorage)
              Session → cookie validated per request via authenticate.ts middleware
              ```

              **Why httpOnly cookies over localStorage?** localStorage is accessible to JavaScript, making JWT tokens vulnerable to XSS attacks. httpOnly cookies are not accessible to JavaScript — the browser attaches them automatically to requests, but client-side scripts cannot read them.

              **Why store OTPs in PostgreSQL, not in-memory?** In-memory OTP storage (e.g., a `Map<string, OTP>`) is lost on server restart. Render's free tier restarts the server periodically. PostgreSQL-backed OTPs survive restarts and allow expiry via `created_at` comparison rather than `setTimeout`.

              ### Rate Limiting Configuration

              ```
              OTP endpoints:   5 requests per IP per 15 minutes
              Login endpoints: 10 requests per IP per 15 minutes
              ```

              These limits are enforced by `express-rate-limit` at the route middleware layer. The limits are intentionally conservative — OTP brute force is the most common auth attack vector against mobile-first applications.

              ### Input Validation (Zod)

              All API inputs are validated with Zod schemas at the route layer before reaching handlers. The same Zod schemas are imported in the frontend for client-side form validation. Benefits:

              - Single source of truth for validation logic
              - TypeScript types are derived from Zod schemas (`z.infer<typeof schema>`), eliminating hand-written interfaces for API request bodies
              - Schema errors are caught before they reach the database layer

              ### Payment Security (Razorpay HMAC)

              Razorpay sends a webhook with `razorpay_payment_id` and `razorpay_signature`. The server verifies the signature before marking an order as paid:

              ```typescript
              const expected = crypto
                .createHmac('sha256', RAZORPAY_KEY_SECRET)
                  .update(`${razorpay_order_id}|${razorpay_payment_id}`)
                    .digest('hex');

                    if (expected !== razorpay_signature) {
                      return res.status(400).json({ error: 'Invalid payment signature' });
                      }
                      ```

                      This prevents replay attacks (where an attacker reuses a valid payment ID from a different order) and forged completion events (where an attacker POSTs to `/verify-payment` with a fabricated payload).

                      ---

                      ## Database Schema Design

                      ### Why PostgreSQL Over MongoDB?

                      The data model is relational: users have reports, reports belong to users, expenses reference users. A relational schema with foreign keys enforces referential integrity — a report without a user cannot exist. MongoDB would require application-level enforcement of these constraints.

                      The Render-managed PostgreSQL instance provides automatic backups, connection pooling, and a stable connection string — appropriate for a demo/portfolio deployment.

                      ### Activity Log

                      The `activity_log` table records user actions with IP address, action type, and detail. This is the foundation of an audit trail. In production, this would feed a SIEM system or anomaly detection pipeline. Currently it's a read-only admin dashboard view.

                      ### Known Scaling Constraints

                      **No connection pooling:** The current `db.ts` creates a direct PostgreSQL connection pool with `pg.Pool`. Render's free tier PostgreSQL has a connection limit of 97. Under concurrent load, the pool can exhaust connections. Production fix: add PgBouncer or use a managed connection pooler.

                      **Single-region PostgreSQL:** All users route to the same Render PostgreSQL instance. For an Indian-farmer-focused application, a deployment in Mumbai (AWS ap-south-1) would reduce database latency significantly.

                      ---

                      ## Production Upgrade Path

                      ### From Render Free Tier to AWS Production

                      | Component | Current (Render Free) | Production Target (AWS) |
                      |-----------|----------------------|------------------------|
                      | Compute | Single Render Web Service | ECS Fargate (auto-scaling) |
                      | Database | Render PostgreSQL | RDS PostgreSQL Multi-AZ |
                      | Sessions | JWT cookies | ElastiCache Redis (session store) |
                      | AI | Gemini API (single provider) | Gemini + Claude fallback |
                      | CDN | None | CloudFront |
                      | Monitoring | None | CloudWatch + X-Ray |
                      | Cost | $0 (free tier) | ~$150–200/month |

                      ### Gemini API Rate Limits

                      The free Gemini tier has rate limits that can cause AI module failures under concurrent load. The current implementation does not implement retry logic or exponential backoff for Gemini calls. Production fix:

                      ```typescript
                      const withRetry = async (fn: () => Promise<T>, maxRetries = 3): Promise<T> => {
                        for (let i = 0; i < maxRetries; i++) {
                            try { return await fn(); }
                                catch (e) {
                                      if (i === maxRetries - 1) throw e;
                                            await sleep(2 ** i * 1000); // exponential backoff
                                                }
                                                  }
                                                  };
                                                  ```

                                                  ### Caching Layer

                                                  Currently, identical AI requests (same query, same parameters) make fresh Gemini API calls every time. A Redis TTL cache would reduce Gemini API usage and response latency for common queries:

                                                  - Crop recommendations: cache by `(soil_params_hash, location)`, TTL 24h
                                                  - Market forecasting: cache by `(crop, date)`, TTL 1h
                                                  - Disease detection: no cache (image inputs are unique)
                                                  - Weather data: cache by `(lat, lon)`, TTL 15m (Open-Meteo returns fresh data every 15 minutes)

                                                  ---

                                                  ## Known Limitations & Honest Tradeoffs

                                                  ### Gemini as Single Point of Failure

                                                  All 6 AI modules fail simultaneously if the Gemini API is unavailable or quota-exhausted. The application has demo fallbacks for some modules but not all. A multi-provider strategy (Gemini → Claude → OpenAI in order of preference) would require abstracting the AI client behind a provider interface.

                                                  ### No Read Replicas

                                                  All database reads route to the primary PostgreSQL instance. For a read-heavy analytics dashboard, read replicas would offload query load. Not implemented.

                                                  ### Drone Analysis Accuracy

                                                  The AgriDrone V2 module uses Gemini's knowledge of geographic features at given coordinates — it does not access live satellite imagery. The terrain analysis is Gemini's best description of what it knows about that location from training data, not a real-time satellite analysis. This is a significant caveat for production agricultural use.

                                                  ### Language Support Limitations

                                                  The 7-language support is implemented as static translation strings in `constants.ts`. AI responses from Gemini are in English regardless of the selected language. Full multilingual AI would require prompt translation or a multilingual-native model.

                                                  ---

                                                  *For quick start and deployment instructions, see the README.*
