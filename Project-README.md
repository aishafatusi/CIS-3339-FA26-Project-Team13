# CIS 3339 Data Platform Project

## Project synopsis

Your team is taking ownership of an existing full-stack web application and improving it: understanding it, refactoring it, securing it, extending it, testing it, and getting it ready to deploy. Build on the codebase you're given. Don't start over with something new.

The application uses the MEVN stack:

- MongoDB for persistent data
- Express and Node.js for the backend REST API
- Vue 3, Vue Router, Pinia, Vite, and Tailwind CSS for the frontend

This is a senior undergraduate group project, so page creation alone isn't enough. We're looking for improvement in component design, application state, API behavior, security, data integrity, testing, setup, and deployment readiness.

Deadlines, team assignments, and submission links are posted in Canvas.

## Application background

The Data Platform supports nonprofit organizations and their Community Health Workers (CHWs). A CHW records clients, creates events, maintains the services an organization offers, and registers clients for events. The dashboard summarizes things like recent attendance and clients by ZIP code.

One database supports multiple organizations, and each deployed instance of the app is configured for a single organization through `ORG_ID`. Data belonging to one organization should never be visible to another.

The template already has client, event, service, organization, login, dashboard, and role-related code in it. Some of it is incomplete or inconsistent, and part of this project is finding those spots and fixing them.

## Learning objectives

By the end of this project, your team should be able to:

1. Read and explain an unfamiliar full-stack codebase.
2. Refactor a Vue application from the Options API to the Composition API.
3. Design reusable components and composables for real application behavior.
4. Connect frontend state and workflows to a REST API.
5. Enforce authentication, authorization, and tenant isolation on the server.
6. Improve data validation and integrity in MongoDB operations.
7. Automate database initialization and make setup reproducible.
8. Produce, verify, and document a deployable frontend build.
9. Test important user workflows and API security rules.
10. Collaborate through issues, branches, pull requests, reviews, and meaningful commits.

## Template overview

The repository has two applications:

```text
frontend/                 Vue 3 single-page application
  src/api/api.js          Axios API client
  src/components/         Chart components
  src/router/             Routes and navigation guard
  src/store/              Pinia login/session store
  src/views/              Dashboard and CRUD views

backend/                  Express REST API
  app.js                  Application setup and database connection
  auth/                   JWT authentication middleware
  models/                 Mongoose schemas and models
  routes/                 Client, event, service, org, and user routes
```

Before you touch any code, trace one complete workflow: pick a Vue view, follow it through `frontend/src/api/api.js`, and find where it lands in an Express route and a Mongoose model.

## Required implementation work

The tasks below describe what the application needs to do. You can reorganize files and pick whatever libraries make sense, but every acceptance criterion has to hold up in the end.

### F1. Migrate the frontend to the Vue Composition API

Convert every Vue view, component, and `App.vue` off the Options API (or whatever mixed style is currently there) and onto the Composition API using `<script setup>`.

The migration has to preserve everything that currently works: client, event, service, dashboard, login, validation, and role-dependent behavior. Use the Composition API features that fit, such as `ref`, `reactive`, `computed`, `watch`, lifecycle hooks, `defineProps`, `defineEmits`, `useRoute`, and `useRouter`.

Acceptance criteria:

- No Vue component under `frontend/src` still has `data()`, `methods`, `created`, or an Options API `export default` definition.
- Route parameters, form validation, Pinia state, API calls, charts, and CRUD workflows all still work.
- Refreshing the page or navigating around normally produces no Vue warnings or JavaScript errors in the console.

### F2. Add a simple async-state composable and centralized API error handling

Build one composable, `useApi(apiCall)`, with this exact shape:

```js
const { data, loading, error, execute } = useApi(() => api.getClients())
```

- `loading` is `true` while the request is in flight, `false` otherwise.
- `data` holds the result on success, and stays `null` until then.
- `error` is a short, human-readable string on failure, `null` when nothing's wrong.
- `execute()` runs the API call, whether that's on mount or as a retry.

Use it in the client, event, and service list views, and on the dashboard.

While you're in there, centralize the response handling too. Add a single Axios response interceptor in `frontend/src/api/api.js` that:

- turns any error response into one short message ("Unable to reach the server" for a network failure, or whatever the server sent back for a validation or server error), and
- on a `401`, clears the stored token and the Pinia session state and sends the user to `/login`.

Acceptance criteria:

- Each of the three list views has a visibly different loading state, empty-results message, and error message, and the error message includes a retry button that calls `execute()` again.
- If the backend is down and you reload a list, you get the error message, not a blank page and not `[object Object]`.
- The interceptor is the only place turning errors into messages. Components shouldn't have their own `try/catch` translation logic anymore.

### F3. Build one reusable data table component

Right now the client, event, and service list views each have their own near-identical table markup. Replace all three with one reusable `DataTable` component.

It needs two props: a `columns` array (each entry has a label, a field key, and optionally a custom render function) and a `rows` array. Sorting and pagination happen client-side, on whatever rows are already loaded into the page. You don't need a server-side query contract for this.

What it needs to do:

- Clicking a column header sorts ascending, then descending, then back to unsorted.
- Show pagination controls (10 rows per page is fine) along with something like "Showing X-Y of Z."
- Show a clear "No results" row when there's no data.

Acceptance criteria:

- The client, event, and service list views all go through this same `DataTable` component.
- Sorting and pagination behave correctly with at least 15 sample rows in each list.
- Changing pages after a search never leaves stale rows on screen from before.

### F4. Restore the login session on refresh and guard editor-only routes

When the app starts up, check storage for a saved token. If it's missing or expired, clear it and treat the user as logged out. If it's still good, restore the Axios authorization header and the Pinia login state so the person doesn't get logged out just from refreshing the page.

Also add `meta: { requiresEditor: true }` to editor-only routes, and one global navigation guard in the router that sends non-editors somewhere else if they try to reach those routes directly.

Acceptance criteria:

- Refreshing the browser while logged in keeps you logged in, as long as the token hasn't expired.
- An expired or corrupted token results in a clean logged-out state, not a crash.
- A viewer who types an editor-only URL directly gets redirected instead of seeing the editor page.
- Logging out clears both the stored token and the Pinia state.

### S1. Security: authentication, roles, and organization isolation

Add two small, reusable Express middleware functions:

- `requireAuth`, which rejects with `401` if there's no valid JWT.
- `requireRole('editor')`, which rejects with `403` if the logged-in user isn't an editor.

Use them according to one rule:

| Operation | Requires |
| --- | --- |
| Read clients, events, services | `requireAuth` |
| Create, update, delete, register, deregister | `requireAuth` + `requireRole('editor')` |

The organization isolation part matters more than anything else in this task. Every database query touching clients, events, or services needs to be scoped to the logged-in user's organization, which comes from the JWT (`req.user.orgId`), never from the request body or URL. Write one shared helper for this, something like `withOrg(filter, req)`, that merges `{ orgId: req.user.orgId }` into a query filter, and use it everywhere instead of writing the organization filter by hand in every route.

A good way to get this right without missing spots: get one route working correctly first, say `GET /clients/:id`, and use it as the template. Then apply the same three-part pattern (`requireAuth`, `requireRole('editor')` where needed, `withOrg(...)`) to the rest of the client, event, and service routes.

Two more changes worth making while you're in here, since they're cheap and matter a lot:

- Lock CORS down to the single origin in `CORS_ORIGIN` instead of `*`.
- On a failed login, return the same message either way ("Invalid username or password") instead of one message for a bad username and a different one for a bad password.

Acceptance criteria:

- Every client, event, and service route uses `requireAuth`, and every route that mutates data also uses `requireRole('editor')`.
- Every database query in these routes goes through the shared `withOrg` helper.
- A logged-in viewer who hits a create, update, delete, or register endpoint directly (with Postman or curl, not just the UI) gets `403`.
- A user from one organization can't read or modify a record that belongs to a different organization.
- CORS only allows the configured origin, and a failed login doesn't reveal whether the username existed.

### B1. Event registration, spelled out

A client registers for an event by creating a record that links `clientId` and `eventId`, however the template already models that (an `attendees` collection, an array on the event, whatever's there already).

Rules for the register and deregister endpoints (keep whatever route names the template already uses):

1. Both the client and the event have to exist and belong to the logged-in user's organization, or you return `404`.
2. Registering a client who's already registered shouldn't create a duplicate. Respond `200` with `{ "message": "Already registered" }` instead.
3. Deregistering a client who isn't registered shouldn't error out. Respond `200` with `{ "message": "Not registered" }` instead.
4. Both endpoints require `requireAuth` and `requireRole('editor')`.

Acceptance criteria:

- Registering the same client for the same event twice in a row still results in exactly one attendee record.
- Deregistering someone who was never registered doesn't throw an error or crash the server.
- Registering a client from organization A for an event in organization B returns `404`.
- Both endpoints return one of the two message shapes above, or the created/removed record on the first successful call.

### B2. Centralized error handling and a health check

Add one Express error-handling middleware, registered after all your routes, that catches errors and always returns the same JSON shape:

```json
{ "error": "Human readable message" }
```

Stack traces, Mongo error details, and secrets should never make it into a response.

Add `GET /health` too, returning something like `{ "status": "ok", "db": "connected" }` (or `"disconnected"`), so it's easy to confirm the app is actually running after it's deployed.

Acceptance criteria:

- A validation error, a not-found error, and an unexpected server error all come back in the same JSON shape, with status codes `400`, `404`, and `500` respectively.
- No response ever contains a stack trace or a raw Mongo error object.
- `GET /health` works without needing to be logged in.

### D1. Automate database initialization and seed data

Get rid of the manual collection setup and the one-off password hash utility, and replace them with a real initialization script, run with something like:

```powershell
cd backend
npm run db:init
```

Running it needs to produce the minimum data required to actually use and grade the application:

- one organization whose ID matches `ORG_ID`
- the required viewer account, username `user`, password `user`
- the required editor account, username `admin`, password `admin`
- some representative clients, services, and events, including at least one event registration
- whatever indexes or constraints your implementation depends on

Both demo accounts need to work right after initialization:

| Role | Username | Password |
| --- | --- | --- |
| Viewer | `user` | `user` |
| Editor | `admin` | `admin` |

It's fine for the script to read these initial values from environment variables, but passwords must be hashed with bcrypt before they're written. The `users` collection should never have a plaintext password in it.

For grading, commit `backend/.env` and `frontend/.env` with a full working configuration. The backend file needs the database connection string, organization ID, JWT secret, CORS origin, and the two seed-account credentials:

```env
MONGO_URL=<working course-project MongoDB connection string>
PORT=3000
ORG_ID=<course-project organization ID>
JWT_SECRET=<course-project-only JWT secret>
CORS_ORIGIN=http://localhost:5173
SEED_VIEWER_USERNAME=user
SEED_VIEWER_PASSWORD=user
SEED_EDITOR_USERNAME=admin
SEED_EDITOR_PASSWORD=admin
```

And the frontend file needs the backend URL used for grading:

```env
VITE_ROOT_API=http://localhost:3000
```

Normally you'd never commit real credentials to a repo. We're making an exception here so grading is reproducible, not because it's good practice. Use a dedicated database and a least-privilege database user, don't reuse anything personal or production, and rotate or disable these credentials once grading is done.

Acceptance criteria:

- Running `npm run db:init` against an empty, correctly configured database creates everything listed above.
- Running it again doesn't create duplicate organizations, users, or sample records.
- The script checks its configuration first, prints a useful summary, and exits with a nonzero code if something's wrong.
- The normal command doesn't destroy existing data. If you add a reset option, it needs its own explicit flag and clear documentation.
- Both `.env` files are committed with a complete, working configuration, including the `user`/`user` and `admin`/`admin` accounts.
- A grader can run the script and log in without ever touching MongoDB directly.

### D2. Deliver and verify a deployable frontend build

The project needs a deployable static frontend, built with Vite:

```powershell
cd frontend
npm install
npm run build
```

`frontend/dist` needs to actually work on a static web host, which means the runtime API configuration, router history fallback, asset paths, and cross-platform path handling all need to be sorted out and documented, not just working by accident on your machine.

Acceptance criteria:

- `npm run build` exits with code 0 and there are no unresolved imports.
- `npm run serve` (or `npm run preview`) serves the production build locally so you can check it.
- Refreshing a nested route like `/clientdetails/:id` works once the SPA fallback is set up and documented.
- The build reads `VITE_ROOT_API` the way it's documented to, with no hard-coded localhost dependency left in.
- The deployed frontend can actually reach the deployed backend, given the CORS policy you configured.
- The README says exactly what hosting setup and deployment steps you used.
- The final demo runs against the production build, not the Vite dev server.

## Bonus: automated tests

Automated tests aren't required to complete the core project. A team that adds them can earn bonus points on top of the 100-point total (see the rubric below).

If your team attempts this, add a real `npm test` command to each application (a placeholder script doesn't count), and cover whichever of these fit the work your team completed:

- the `useApi` composable, both success and failure (F2)
- the login/session-restore workflow (F4)
- a viewer getting `403` and an editor succeeding on the same mutation route (S1)
- organization isolation, meaning a user can't read or modify another organization's record (S1)
- the duplicate-registration and not-registered rules from B1

Write down the exact commands used to run these tests. Test data shouldn't depend on the team's production database.

## Documentation requirements

Update the repository README so a new developer, or a grader, can reproduce the whole project from scratch. It should cover:

- prerequisites and the supported Node.js version
- frontend and backend install and startup commands
- the committed `backend/.env` and `frontend/.env` files, with a short note on what each variable does
- how to run database initialization
- build commands, and test commands if your team attempted the bonus testing task
- the demo users and their roles
- deployment instructions, including SPA fallback and CORS configuration
- known limitations left over after the project

Whatever ends up in the committed `.env` files needs to be dedicated to this course project only. Don't put personal credentials, credentials reused elsewhere, production secrets, or any other real access token in the repo, in screenshots, in reports, or in the presentation.

## Sprint deliverables

### Sprint 1: functional specification and project plan

Submit one PDF through Canvas, containing:

1. A high-level explanation of the frontend, backend, database, and how they interact (an architecture diagram belongs here).
2. At least two visual user-flow diagrams based on the current template.
3. A trace of one existing request, from the Vue view all the way to the database and back.
4. A gap analysis mapping every task in this guideline to the current code.
5. A testable functional specification for what you're proposing to change.
6. A timeline with owners, dependencies, review assignments, and expected completion dates.
7. An appendix describing what AI tools you used, what you asked them, how you verified the output, and which decisions were actually made by the team.

AI tools can help your team understand or explain the template. Your team is still responsible for verifying anything they generate.

### Sprint 2: frontend implementation

Implement F1 through F4, D2, and whatever frontend behavior and API contracts S1 and B1 need. You can mock backend responses that don't exist yet, but the mocks need to match the agreed API shapes and be easy to swap out later.

Submit the GitHub repository link through Canvas. The main branch needs to build and run. Include a short demo of the required workflows (and frontend test results too, if your team is attempting the bonus testing task).

### Sprint 3: integrated backend, database, security, and deployment

Finish S1, B1, B2, D1, D2, and deployment documentation. Replace every frontend mock with a real API call.

Submit through Canvas:

- the GitHub repository link
- the required technical report as a PDF
- the committed grading `.env` files, with a working database connection and the required demo accounts
- the deployed application URLs, if your instructor requires deployment
- a 10-15 minute group presentation video, submitted where Canvas specifies

The video needs to show database initialization, logging in as both roles, a denied viewer action, an organization isolation test, the client/event/service workflows, `npm run build`, and the production build running (locally served or deployed).

You can still make frontend corrections during Sprint 3 if integration requires it.

### Sprint 4: individual peer evaluation

Complete the peer evaluation in Canvas. Your instructor will provide the details and deadline.

## Repository and teamwork expectations

- Use the GitHub repository for your team project.
- Agree on a branch, review, and merge process before you start implementing.
- Track required work with issues or an equivalent project board.
- Use pull requests and peer review for anything substantial.
- Every member needs multiple meaningful commits reflecting their own implementation and testing work.
- Comments, formatting-only changes, generated build output, and commits made on someone else's behalf don't count as an equal contribution.
- Only the default branch gets graded unless your instructor says otherwise.
- Don't submit a ZIP archive unless Canvas specifically asks for one.

## Suggested evaluation rubric

Each row is graded against the acceptance criteria for every task it covers, including that area's documentation.

| Area | Points |
| --- | ---: |
| Sprint 1 specification, analysis, flows, and plan | 10 |
| Frontend and API migration  | 35 |
| Backend and integration  | 45 |
| Peer evaluation  | 10 |
| **Total** | **100** |
| Bonus: automated tests (see above) | up to 20 |

The instructor may adjust this rubric or apply deductions in Canvas. If the database can't be initialized (D1), or the frontend can't be built and served (D2), using the documented commands, that deliverable may receive no credit.

## Definition of done

The project is done when someone else can clone the repository and, using only the committed documentation and their own environment values, do all of this:

1. install frontend and backend dependencies
2. initialize an empty database with `npm run db:init`
3. start the backend and frontend
4. log in as a viewer and as an editor
5. verify the documented authorization and organization-isolation rules
6. complete the client, event, and service workflows
7. create and serve the production frontend with `npm run build`
