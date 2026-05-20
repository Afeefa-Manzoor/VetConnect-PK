# VetConnect PK — ویٹ کنیکٹ پاکستان

AI-powered veterinary service app for Pakistan — farmers describe their animal's problem in Urdu, Roman Urdu, or English and the app runs a 5-agent AI pipeline to find and book the nearest available vet.

## Run & Operate

- `pnpm --filter @workspace/api-server run dev` — run the API server (port 5000)
- `pnpm run typecheck` — full typecheck across all packages
- `pnpm run build` — typecheck + build all packages
- `pnpm --filter @workspace/api-spec run codegen` — regenerate API hooks and Zod schemas from the OpenAPI spec
- `pnpm --filter @workspace/db run push` — push DB schema changes (dev only)
- Required env: `DATABASE_URL` — Postgres connection string

## Stack

- pnpm workspaces, Node.js 24, TypeScript 5.9
- API: Express 5
- DB: PostgreSQL + Drizzle ORM
- Validation: Zod (`zod/v4`), `drizzle-zod`
- API codegen: Orval (from OpenAPI spec)
- Build: esbuild (CJS bundle)
- Mobile: Expo (React Native) with Expo Router

## Where things live

- `artifacts/mobile/` — Expo mobile app
- `artifacts/mobile/app/` — screens (Expo Router file-based routing)
- `artifacts/mobile/context/AppContext.tsx` — global state (language, user, bookings)
- `artifacts/mobile/data/mockData.ts` — mock vet providers, animals list
- `artifacts/mobile/data/translations.ts` — EN/Urdu strings
- `artifacts/mobile/constants/colors.ts` — VetConnect brand palette

## Architecture decisions

- Frontend-only first build: all state in AsyncStorage (no backend needed for MVP)
- 5-agent AI pipeline simulated on-device with timed animations showing each step
- Full EN↔Urdu toggle with RTL text alignment via `isRTL` flag in AppContext
- Mock vet data covers Punjab/KPK with realistic distances and ratings
- Booking flow: Request → AI Processing → Results → Provider Detail → Booking Success

## Product

**VetConnect PK** connects Pakistani farmers with veterinary doctors. Key flows:
1. **Onboarding** — language select (EN/Urdu), location detection, animal registration
2. **Home** — describe problem (text/voice), animal shortcuts, emergency button
3. **AI Request** — 5-agent pipeline trace: Intent → Triage → Discovery → Matching → Booking
4. **Results** — ranked vet cards with Best Match highlighted
5. **Provider Detail** — time slot picker, booking confirmation
6. **Booking Success** — confirmation ID, reminders info, call vet
7. **Search Tab** — browse/filter vets by animal type
8. **Bookings Tab** — upcoming and past bookings
9. **Profile Tab** — language toggle, my animals

## User preferences

- Brand colors: Farm Green #3E7C3A, Sky Blue #5DADE2, Warm Cream #F7F4EC, Soft Orange #E8A24D
- Full Urdu RTL support via lang toggle in profile
- No emojis in UI (icons only via @expo/vector-icons)

## Gotchas

- AsyncStorage key prefix: `@lang`, `@user`, `@bookings`
- react-native-maps must be pinned to 1.18.0 if added (Expo Go compatibility)
- Do not add react-native-maps to app.json plugins

## Pointers

- See the `pnpm-workspace` skill for workspace structure, TypeScript setup, and package details
