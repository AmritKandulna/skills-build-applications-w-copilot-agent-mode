# OctoFit Tracker architecture

OctoFit Tracker is a three-tier application for a school fitness program. The starter code includes API contracts and intentionally incomplete implementation points so you can use GitHub Copilot agent mode in a test-driven loop.

![Three-tier OctoFit architecture](../.github/images/octofit-architecture.svg)

## Tier boundaries

- **Presentation tier**: React 19, Vite, React Router, Bootstrap, and a shared API client in `octofit-tracker/frontend/src`.
- **Logic tier**: Express and TypeScript routes/services in `octofit-tracker/backend/src` expose `/api/*` behavior on port `8000`.
- **Data tier**: Mongoose models and seed data in `octofit-tracker/backend/src/models` and `src/scripts/seed.ts` persist to MongoDB on port `27017`.

## Ports and environment rules

- MongoDB: `27017`
- Express API: `8000`
- Vite frontend: `5173`
- The browser calls relative `/api` paths. Vite proxies them to `http://localhost:8000`, so the unauthenticated API port stays private in Codespaces. `VITE_API_BASE_URL` can override the base URL.

## API contract

| Capability | Method and path | Expected behavior |
| --- | --- | --- |
| Activity tracking | `GET /api/activities`, `POST /api/activities` | List and create valid activity records. POST accepts `userId`, `teamId`, `type`, and `minutes`; `userId` and `teamId` must be well-formed IDs for an existing user and team; otherwise respond with a deliberate 4xx (never a 500 from a cast error). A created activity stores the submitted `user`, `team`, `type`, and `minutes` plus computed `points` and `occurredAt`. |
| Teams | `GET /api/teams`, `POST /api/teams` | List and create teams with members. Created and listed teams keep the submitted `name` and `mascot`. Reject a team without a name with `400`. |
| Leaderboard | `GET /api/leaderboard` | Return teams ordered by the sum of their activity points, highest first. Each entry has `teamName`, `totalPoints`, and a 1-based `rank`. |
| Workout suggestions | `GET /api/workouts/suggestions/:userEmail` | Recommend workouts from recent activity patterns for the matching seeded user's email address. The first suggestion's `focus` matches that user's goal or recent activity, with a `reason` explaining why. An email with no matching user returns a deliberate 4xx (for example, `404`). |

## Implementation decisions

### Data tier
- Define Mongoose models for `User`, `Team`, `Activity`, and `Workout` with schema validation.
- **User**: Require `name`, `email`, and `fitnessGoal`; `email` must be unique.
- **Team**: Require `name` and `mascot`; include `members` array referencing User IDs.
- **Activity**: Require `type` (e.g., "run", "swim"), `user` (ObjectId), `team` (ObjectId), and `minutes`; compute `points` based on type and minutes; set `occurredAt` to current date if not provided.
- **Workout**: Require `title`, `focus` (e.g., "endurance", "strength"), `minutes`, and `difficulty` (1–5); used for suggestions.
- Implement seed data in `src/scripts/seed.ts` for all models idempotently.

### Logic tier
- **Activities endpoints**: 
  - `GET /api/activities`: Return all activities.
  - `POST /api/activities`: Accept `userId`, `teamId`, `type`, `minutes`; validate that user and team exist; compute and store `points`; return 400 if validation fails (never 500).
- **Teams endpoints**:
  - `GET /api/teams`: Return all teams with populated members.
  - `POST /api/teams`: Accept `name` and `mascot`; return 400 if `name` is missing or empty.
- **Leaderboard endpoint**:
  - `GET /api/leaderboard`: Return teams ordered by total activity points (sum across all team members' activities), highest first; include 1-based `rank`, `teamName`, and `totalPoints`.
- **Workout suggestions endpoint**:
  - `GET /api/workouts/suggestions/:userEmail`: Find user by email; analyze their recent activities to determine focus (e.g., most common type); return workouts matching that focus; include `reason` explaining the suggestion (e.g., "matches your recent running activity"); return 400 or 404 for invalid or missing user.
- Separate concerns: routes in `src/routes`, services/business logic in `src/services`, models in `src/models`.

### Presentation tier
- Replace all "Implementation pending" placeholder pages with real data-fetching components.
- **Activities page**: Fetch and display all activities; include columns for user, team, type, minutes, points, and date; add a form to create new activities with validation feedback.
- **Teams page**: Fetch and display all teams; show team name, mascot, and member count; add a form to create new teams.
- **Leaderboard page**: Fetch and display ranked teams; show rank, team name, and total points; sort by points descending.
- **Workout suggestions page**: Prompt user for email; fetch and display workout suggestions; show each suggestion's title, focus, duration, difficulty, and reason.
- Add empty states (e.g., "No activities yet") and accessible error alerts for failed API calls (e.g., "Failed to load activities").
- Use Bootstrap for consistent styling and responsive layout.

## Test plan

### Implementation order and testing strategy
1. **Data tier**: Implement Mongoose models, then run data contract tests to validate schema and seeding.
2. **Logic tier**: Implement API routes and services, then run API contract tests to validate endpoint behavior and error handling.
3. **Presentation tier**: Implement React components, then run frontend tests to validate UI rendering and user interactions.

### Test commands and expected results

| Phase | Command | Expected success |
|-------|---------|------------------|
| **Data-tier validation** | `npm run test:data --prefix octofit-tracker/backend` | All 5 tests pass: User, Team, Activity, Workout models validate required fields; seed data loads idempotently |
| **API-tier validation** | `npm run test:api --prefix octofit-tracker/backend` | All 4 tests pass: Activities and Teams accept valid input (201), reject invalid input (400); Leaderboard returns teams ranked by total points; Workout suggestions include title, focus, duration, difficulty, and reason |
| **Frontend validation** | `npm test --prefix octofit-tracker/frontend` | All 15 tests pass: Activities page renders fetched activities; Teams page renders fetched teams; Leaderboard page renders ranked teams; Workout suggestions page fetches and displays suggestions; all pages include empty states and error alerts |
| **Backend build** | `npm run build --prefix octofit-tracker/backend` | TypeScript compiles without errors; routes and services are type-safe |
| **Frontend build** | `npm run build --prefix octofit-tracker/frontend` | React components build successfully; no console errors during build |

### Test failure diagnostics

- **Data-tier failures**: Missing Mongoose schema validation; run `npm run test:data` to identify which fields lack required validation.
- **API-tier failures**: Route handlers not implemented or returning 501; invalid input not validated (should return 400); aggregate queries not calculating totals or ranking correctly.
- **Frontend failures**: Components showing "Implementation pending"; API calls not made or data not rendered; missing empty states or error alerts.
- **Build failures**: TypeScript type errors or module resolution issues; check `src/routes`, `src/services`, and `src/models` for syntax or import errors.
