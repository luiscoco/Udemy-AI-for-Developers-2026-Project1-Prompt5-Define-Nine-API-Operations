# Defining the API Contract: Step by Step

This note walks through how `packages/contract/openapi.yaml` was extended
with the nine API operations for the Equipment Maintenance Hub. It's meant
as a worked example for students learning to write API contracts with
OpenAPI.

## 1. Read the existing file first

Before adding anything, the current `openapi.yaml` was read in full. This
mattered because the `components.schemas` section already defined every
schema the new operations would need:

- `Asset`, `Technician`, `WorkOrder` — the core resources
- `NewWorkOrder`, `TransitionCommand`, `AssignmentCommand` — request bodies
- `DashboardSummary` — the aggregate response
- `ApiError` — the shared error shape
- `WorkOrderState`, `WorkOrderAction`, `Priority` — enums reused across schemas

Because these already existed, the task was purely about writing the
`paths` section and wiring each operation to the right schema with `$ref`
— no new schemas were needed.

## 2. Map each requirement to an OpenAPI path + method

The nine operations were translated one by one into `path: { method: {...} }`
blocks:

| # | Method & path | operationId |
|---|---|---|
| 1 | `GET /api/health` | `getHealth` |
| 2 | `GET /api/assets` | `listAssets` |
| 3 | `GET /api/technicians` | `listTechnicians` |
| 4 | `GET /api/work-orders` | `listWorkOrders` |
| 5 | `GET /api/work-orders/{id}` | `getWorkOrder` |
| 6 | `POST /api/work-orders` | `createWorkOrder` |
| 7 | `POST /api/work-orders/{id}/assignment` | `assignWorkOrderTechnician` |
| 8 | `POST /api/work-orders/{id}/transitions` | `transitionWorkOrder` |
| 9 | `GET /api/dashboard/summary` | `getDashboardSummary` |

Each `operationId` is camelCase and describes the action in plain English,
which is what code generators (and other developers) use as the function
name for that endpoint.

## 3. Add parameters where an operation needs input

- **Path parameters** (`{id}`) were declared with `in: path`, `required: true`,
  and typed as `uuid` — used on the single-work-order, assignment, and
  transition endpoints.
- **Query parameters** (`state`, `priority`) were declared with
  `in: query`, `required: false`, and reused the existing `WorkOrderState`
  and `Priority` enum schemas via `$ref` rather than redefining the list of
  valid values.

## 4. Add request bodies where the operation accepts a payload

`POST` operations each got a `requestBody` pointing at the matching schema:

- `createWorkOrder` → `NewWorkOrder`
- `assignWorkOrderTechnician` → `AssignmentCommand`
- `transitionWorkOrder` → `TransitionCommand`

## 5. Document every response, including the error cases

For each operation, every response mentioned in the requirements was added
as its own status code, not just the "happy path":

- Success codes return the resource (`200`) or the created resource (`201`).
- Failure codes (`400`, `404`, `409`) all return the shared `ApiError`
  schema, so consumers can rely on one consistent error shape everywhere.

This step is easy to skip but important: an OpenAPI spec that only
documents the 200 response doesn't tell frontend developers what to expect
when something goes wrong, and it can't be used to generate accurate
error-handling code.

## 6. Validate the YAML

After writing the `paths` block, the file was parsed with a YAML parser to
confirm it was syntactically valid and that all 8 path entries (one path,
`/api/work-orders`, has two methods, covering 9 operations total) were
present as expected.

## Running the project (Windows Terminal / PowerShell)

At this stage of the course the repository only contains the API contract
(`packages/contract/openapi.yaml`) plus empty `apps/backend` and
`apps/frontend` scaffolds — there's no Fastify server or Vite dev server
implemented yet, so the root `dev`/`build` scripts are placeholders. From
the repository root, run:

```powershell
npm install
npm run dev
```

`npm install` sets up the npm workspaces (`apps/*`, `packages/*`).
`npm run dev` currently just prints `no dev server configured yet` — once
the backend (Fastify) and frontend (Vite) apps are implemented in a later
step, this same command is where their real dev servers will be wired up.

## Takeaways for students

1. **Check what already exists** before adding new schemas — reuse `$ref`
   instead of duplicating definitions.
2. **Name operations clearly** (`operationId`) — this becomes the function
   name in generated clients.
3. **Model every documented response**, success and failure alike, so the
   contract is a complete source of truth for both frontend and backend
   implementers.
4. **Validate the file** after editing — a broken YAML file breaks every
   tool that consumes the contract (codegen, mock servers, linters).
