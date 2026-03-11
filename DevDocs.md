# Developer Documentation

## Project Overview

This repository contains a car-pooling platform split into three applications:

- `BACKEND/Car-Pooling-System-Backend`: Node.js + Express + MongoDB API, ride lifecycle logic, OTP verification, payments, and chat.
- `FRONTEND/Car-Pooling-System-Web-Frontend`: React web client for drivers and riders.
- `FRONTEND_MOBILE/Car-Pooling-System-Mobile-Frontend`: Expo/React Native client with role-specific navigation.

The system solves the core ride-sharing flow for daily commuters:

- Drivers publish rides with route, timing, seat setup, and preferences.
- Riders discover rides by location/date, request or book seats, and track ride status.
- Both roles use chat and live ride updates.
- The platform tracks trust/verification/payment/emission metrics around those rides.

## Architecture Overview

### High-level system design

1. Clients authenticate with Clerk.
2. Clients call backend REST APIs for domain actions (driver profile, ride lifecycle, rider booking, verification, payment).
3. Backend persists data in MongoDB via Mongoose models.
4. Realtime behavior (chat + live ride notifications) runs over Socket.IO.
5. Optional integrations:
   - Twilio Verify (phone OTP)
   - Firebase Storage (document/image uploads from clients)
   - SMTP/Nodemailer (booking/ride-start/SOS emails)
   - External ML service proxy (`/api/ml/*`)

### Major modules and interaction

- Backend route modules are grouped by concern:
  - `routes/driver/*`
  - `routes/rider/*`
  - `routes/rides/*`
  - `routes/chat.router.js`
  - `routes/payment/payment.router.js`
  - support routers: health, ML proxy, phone verification, carbon emission.
- Ride matching relies on encoded polyline + grid indexing (`route.gridsCovered`) and utility math functions.
- Web frontend uses React Router and a centralized API wrapper in `src/lib/api.js`.
- Mobile frontend uses Expo Router route groups: `(auth)`, `(app)` for drivers, `(rider)` for riders.

### Important design decisions

- **Embedded ride snapshots**: ride documents embed driver/vehicle/passenger snapshots to reduce joins during search/details.
- **Grid-based route indexing**: ride search first filters by discrete geo-grid intersection, then computes segment distance/fare.
- **Request/confirm booking workflow**: booking initially creates `requested` passengers; seat decrement happens on driver confirmation.
- **Realtime room model**:
  - conversation rooms for chat
  - `ride:{rideId}` rooms for live ride events.
- **Role-specific UX**: Clerk `unsafeMetadata.role` controls routing in web/mobile clients.

## Project Structure

### Repository root

- `BACKEND/`: API service
- `FRONTEND/`: web app
- `FRONTEND_MOBILE/`: mobile app

### Backend (`BACKEND/Car-Pooling-System-Backend`)

- `server.js`: app bootstrap, middleware, router mounting, Socket.IO startup.
- `config/db.js`: MongoDB connection.
- `models/`: Mongoose schemas (`Ride`, `Driver`, `Rider`, `Payment`, `Conversation`, `Message`, etc.).
- `routes/`: REST handlers by domain.
- `socket/chat.socket.js`: Socket.IO event handlers.
- `utils/`: geo/polyline/fare/recurrence/mailer helpers.
- `tests/`: Jest route/integration/util tests.
- `Dockerfile`, `docker-compose.yml`, `railway.json`: deployment/runtime config.

### Web frontend (`FRONTEND/Car-Pooling-System-Web-Frontend`)

- `src/main.jsx`: React root + ClerkProvider + BrowserRouter.
- `src/App.jsx`: route table and role-based route selection.
- `src/Pages/`: feature pages by role.
- `src/components/`: UI building blocks.
- `src/lib/api.js`: fetch-based API layer with base-url fallback logic.
- `src/services/`: additional service wrappers (partially legacy/inconsistent; see notes).
- `utils/`: Firebase integration and upload helpers.
- `vite.config.js`: alias `@ -> ./src`, env prefix config.
- `vercel.json`: SPA rewrite config.

### Mobile frontend (`FRONTEND_MOBILE/Car-Pooling-System-Mobile-Frontend`)

- `app/_layout.jsx`: root providers (Clerk + Socket context).
- `app/(auth)/*`: sign-in and role selection.
- `app/(app)/*`: driver-oriented tabs and flows.
- `app/(rider)/*`: rider-oriented tabs and flows.
- `context/SocketContext.js`: Socket.IO client lifecycle.
- `utils/`: Firebase upload, polyline, tests.
- `app.json`: Expo config (scheme/plugins/etc.).

## Key Components

### Backend core modules

- `Ride` model (`models/ride.model.js`)
  - central domain entity with route, schedule, pricing, seats, passengers, live-state fields.
- `Driver` model (`models/driver.model.js`)
  - driver profile, verification, ratings, stats, and both legacy single `vehicle` and new `vehicles[]`.
- `Rider` model (`models/user.model.js`)
  - rider profile, bookings, verification, emergency contacts, trust-related fields.
- `chat.socket.js`
  - handles messaging events, typing/read receipts, and live ride event broadcasts.
- `utils/mailer.utils.js`
  - sends booking confirmations, ride-start notifications, and SOS alerts.

### Web frontend core modules

- `src/App.jsx`: role-aware route switch for rider/driver views.
- `src/hooks/useProfile.js`: orchestrates role-specific profile/stat/ride fetches.
- `src/lib/api.js`: exported endpoint functions used by pages.
- `src/Pages/driver/CreateRidePage.jsx`: map-driven ride creation/edit payload builder.

### Mobile frontend core modules

- `app/(auth)/sign-in.jsx`: Clerk OAuth entry.
- `app/(auth)/role-select.jsx`: sets role and triggers backend driver registration.
- `app/(rider)/search/index.jsx`: nearby/search flow with maps, filters, and booking navigation.
- `context/SocketContext.js`: auto-connects socket per authenticated user.

## API or Service Layer

### Backend route map (mounted prefixes)

- `GET /health`
- `GET|POST /api/ml/*`
- `POST /api/phone-verification/send-otp`
- `POST /api/phone-verification/verify-otp`
- `GET /get-emission`
- ` /api/chat/*`
- ` /api/payment/*`
- ` /api/rides/*`
- ` /api/rider/*`
- ` /api/driver-*` (through driver router mounted at `/api`)

### Ride lifecycle endpoints (selected)

- `POST /api/rides`
  - Creates ride after route validation and schedule collision check.
- `GET /api/rides/search`
  - Query params: `pickupLat,pickupLng,dropLat,dropLng,date?,minSeats?`
  - Returns candidate rides with estimated segment fare.
- `GET /api/rides/nearby`
  - Query params: `lat,lng,radiusKm?,limit?`
  - Returns upcoming rides near user location.
- `POST /api/rides/:rideId/book`
  - Body includes rider, pickup/drop, seat preference, optional guest passengers.
  - Creates `requested` passengers and rider booking record.
- `POST /api/rides/:rideId/confirm-request`
  - Driver confirms pending request(s), including grouped guest bookings.
- `POST /api/rides/:rideId/cancel`
  - Driver can cancel whole ride; riders can cancel their own/group booking.
- `POST /api/rides/:rideId/start|complete|verify-otp|update-location|drop-passenger`
  - Live ride state transitions and participant tracking.

### Driver and rider endpoints (selected)

- Driver:
  - `POST /api/driver-register/:userId`
  - `GET|PUT|DELETE /api/driver-profile/:userId`
  - `GET|POST|DELETE /api/driver-rating/:userId`
  - `GET|PUT /api/driver-docs/:userId`
  - `GET|POST|PUT|DELETE /api/driver-vehicles/:userId...`
  - `GET|POST /api/driver-stats/:userId...`
  - `GET|PUT|DELETE /api/driver-verification/:userId`
- Rider:
  - `GET /api/rider/rider-rides/:userId`
  - `GET|PUT /api/rider/rider-verification/:userId`
  - `GET|PUT|POST /api/rider/emergency/:userId...`

### Chat and payments

- Chat REST:
  - `GET /api/chat/conversations?userId=...`
  - `GET /api/chat/messages/:conversationId?page=&limit=`
  - `POST /api/chat/conversations/direct`
  - `POST /api/chat/conversations/group`
- Payment REST:
  - `POST /api/payment`
  - `PUT /api/payment/:paymentId/status`
  - `GET /api/payment/:paymentId`
  - `GET /api/payment/passenger/:passengerId`
  - `GET /api/payment/driver/:driverId`

### Realtime socket events

- Chat: `join-conversations`, `join-room`, `send-message`, `typing`, `stop-typing`, `mark-read`.
- Live ride: `join-ride`, `location-update`, `ride-started`, `rider-ready`, `otp-verified`, `passenger-dropped`, `sos-alert`, etc.

### Example usage

```bash
# Search rides
GET /api/rides/search?pickupLat=12.97&pickupLng=77.59&dropLat=12.92&dropLng=77.62&minSeats=1

# Book ride
POST /api/rides/{rideId}/book
{
  "user": { "userId": "user_123", "name": "Rider A" },
  "pickup": { "lat": 12.97, "lng": 77.59 },
  "drop": { "lat": 12.92, "lng": 77.62 },
  "seatPreference": "any"
}
```

## Data Models

### `Ride`

Important fields:

- `driver`: embedded user snapshot (`userId`, `name`, `rating`, live location, verification flags in details endpoint).
- `vehicle`: embedded vehicle snapshot.
- `route`: `start`, `end`, `stops[]`, `encodedPolyline`, `gridsCovered[]`.
- `schedule`: `departureTime`, optional recurrence metadata.
- `seats`: `total`, `available`, `seatTypes[]`.
- `pricing`: `baseFare`, `currency`, `pricePerKm`.
- `passengers[]`: booking state, seat selection, guest metadata, OTP/live/dropoff fields.
- `status`: `scheduled|ongoing|completed|cancelled`.

### `Driver`

- `userId` (unique external identity from Clerk)
- profile fields (`profileImage`, `phoneNumber`)
- `vehicles[]` (new) and `vehicle` (legacy single-vehicle structure)
- `documents`
- `rating`, `rides` counters, distance/hours, earnings
- `verification` subdocument

### `Rider`

- `userId` (unique)
- `bookings[]` with `rideId`, grid info, fare/status
- `verification`
- `emergencyContacts[]`, `sosSecretCode`
- usage/trust counters

### Other models

- `Payment`: ride/passenger/driver amount + status lifecycle.
- `Conversation` / `Message`: direct/group chat state and messages.
- `Emission`: emission factor by vehicle type.
- `RideInstance`: recurrence support placeholder (not widely used in current routes).

## Setup and Development

## Prerequisites

- Node.js 18+ recommended.
- MongoDB connection string (Atlas/local).
- Clerk project (web + mobile).
- Google Maps API key (web/mobile map flows).
- Optional: Twilio Verify, SMTP, Firebase.

### Backend local setup

```bash
cd BACKEND/Car-Pooling-System-Backend
npm install
npm start
```

Docker option:

```bash
docker compose up --build
```

### Web frontend setup

```bash
cd FRONTEND/Car-Pooling-System-Web-Frontend
npm install
npm run dev
```

### Mobile frontend setup

```bash
cd FRONTEND_MOBILE/Car-Pooling-System-Mobile-Frontend
npm install
npm run start
```

### Environment variables

Observed from code references:

- Backend:
  - `MONGO_URI`, `PORT`
  - `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_VERIFY_SID`
  - `ML_SERVICE_URL`
  - `SMTP_HOST`, `SMTP_PORT`, `SMTP_SECURE`, `SMTP_USER`, `SMTP_PASS`, `EMAIL_FROM`
  - `WEB_URL`, `APP_SCHEME`
  - `MONGODB_URI` (used by migration script, different name from runtime `MONGO_URI`)
- Web:
  - `VITE_CLERK_PUBLISHABLE_KEY`
  - `VITE_BACKEND_URL` or `VITE_API_BASE_URL`
  - `VITE_GOOGLE_MAPS_API_KEY`
  - `VITE_FIREBASE_API_KEY`, `VITE_FIREBASE_AUTH_DOMAIN`, `VITE_FIREBASE_PROJECT_ID`, `VITE_FIREBASE_STORAGE_BUCKET`, `VITE_FIREBASE_MESSAGING_SENDER_ID`, `VITE_FIREBASE_APP_ID`
  - some files also accept `EXPO_PUBLIC_*` variants.
- Mobile:
  - `EXPO_PUBLIC_BACKEND_URL`
  - `EXPO_PUBLIC_GOOGLE_MAPS_API_KEY`
  - `EXPO_PUBLIC_FIREBASE_*`
  - one file uses `EXPO_BACKEND_URL` (without `PUBLIC`), which is inconsistent.

## Build and Deployment

### Backend

- Build/runtime:
  - Dockerfile uses `node:18-alpine`, installs production dependencies, runs `node server.js`.
- Deployment config:
  - `railway.json` uses Nixpacks and `npm start`.

### Web frontend

- Build:
  - `npm run build` (Vite output in `dist/`)
- Deployment:
  - `vercel.json` rewrites all routes to `index.html` for SPA routing.

### Mobile frontend

- Expo-managed app:
  - `npm run android`, `npm run ios`, `npm run web` for dev targets.
- No CI/CD deployment pipeline is defined in this repository for app store delivery.

## Code Conventions

### Naming and organization

- Backend uses domain-based router segmentation and `*.router.js` naming.
- Mongoose models are co-located in `models/` with `*.model.js`.
- Frontend uses page-first organization (`Pages/*`) plus shared `components/*`.
- Mobile uses Expo Router filesystem routes with role-based route groups.

### Architectural patterns

- Backend follows a thin controller style inside route files (no separate service/repository layers).
- Shared calculations (fare/polyline/grid) live in utility modules and are imported by route handlers.
- Clients mostly use centralized API wrappers rather than direct fetches, though mobile screens frequently call `fetch` directly.

### Practices visible in code

- Progressive enhancement/fallback is common (base URL fallback, nearby search fallback, Twilio mock mode).
- Defensive checks are present in many handlers, but validation is not fully standardized.
- Tests are present mainly for backend routes/utils; frontend tests are minimal and mobile tests are placeholders.

## Future Improvements or Notes

### Technical debt and inconsistencies

- Driver schema currently keeps both `vehicle` and `vehicles[]`, and routes support both patterns.
- Some web service modules are inconsistent with actual API exports:
  - `src/services/driverService.js` and `src/services/rideService.js` import a default `api` from `src/lib/api.js`, but that file exports named functions instead.
  - `src/Pages/CreateRidePage.jsx` imports named functions not exported by those services.
- Booking status enum mismatch:
  - `Rider.bookings.status` enum does not include `"completed"`, but live completion route sets booking status to `"completed"`.
- Duplicate/overlapping passenger-removal endpoints exist (`ride.remove-passenger.router.js` and `ride.driver.router.js`).
- `routes/carbon.router.js` uses `GET` with `req.body`; this is unusual and brittle for clients/proxies.
- Environment naming is inconsistent:
  - `MONGO_URI` vs `MONGODB_URI`
  - `EXPO_PUBLIC_BACKEND_URL` vs `EXPO_BACKEND_URL`.
- `app.json` currently contains a Clerk publishable key value directly; typically this should be environment-driven.

### Missing or unclear context

- No root-level `.env.example` covering all three apps.
- The ML service itself is not in this repository; only proxy routes exist.
- Web frontend includes an older `src/Pages/CreateRidePage.jsx` alongside active `src/Pages/driver/CreateRidePage.jsx`; intended ownership of legacy pages is not documented.
- Mobile and web have substantial direct API usage in screens but limited shared API contract typing; no OpenAPI/Swagger contract exists.

### Recommended next refactors

1. Consolidate vehicle data model and remove legacy `vehicle` paths.
2. Standardize booking status enums and ride lifecycle transitions.
3. Introduce request validation middleware (e.g., zod/joi) and shared error handling.
4. Unify env var names across backend/web/mobile and publish complete `.env.example` files.
5. Remove or fix stale service wrappers and unused legacy pages in web frontend.
6. Add API contract documentation (OpenAPI) and typed client generation for web/mobile.
7. Expand automated tests for web/mobile and end-to-end ride flows.

