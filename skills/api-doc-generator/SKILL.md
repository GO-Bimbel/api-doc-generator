---
name: api-doc-generator
description: Analyze the endpoints in a backend source file or module (NestJS, Node.js, Go, or .NET) and produce a single plain-language Indonesian DOCX document covering, for each endpoint, the request payload/DTO, query parameters, response shape, database queries and table relations. Follows HTTP calls into sibling microservice repositories (e.g. `${SVC_REPO_PROJECT_B}/api/v1/foo`) to find the real DTOs, responses and queries, either one hop (`single`, default) or through every service in the chain (`deep`). Use when the user invokes the api-doc-generator skill (e.g. `/api-doc-generator <path> [--depth single|deep]`) or asks for an API documentation document/report of an endpoint from its source code. Also fills missing `@nestjs/swagger` annotations (`@ApiTags`, `@ApiOperation`, and `@ApiBody`/`@ApiQuery` with proxy DTOs) in the target NestJS project, applying them only after the user confirms. Not for authoring OpenAPI/Swagger specs by hand, README files, or inline code comments.
compatibility: Node.js plus bun, pnpm, or npm to install the `docx` dependency. No office suite required.
---

# API doc generator

Turn a backend module's source into one API document that a non-technical reader can follow: what to send, what comes back, and which data is read or written. You write a JSON file; the bundled scripts validate it and render the DOCX.

**One path in, one document out.** Whatever the target is (a file, a module, or a folder of many modules), write exactly one `<target-name>.json` and one `<target-name>.docx` containing every endpoint under it. Never ask whether to combine or split, and never produce per-endpoint or per-module files.

**Write for non-technical readers, in Bahasa Indonesia.** Short sentences, everyday words, and the business meaning of every field. Technical identifiers (field, class, table, env var, HTTP method) stay in their original form so engineers can trace them, but always sit next to a plain explanation.

**The document describes what the client of the target service sends and receives.** When the target only forwards a request to another service, the real DTO, response, and queries live in that other service, so they are documented from there.

## Analysis depth

The user picks how far cross-service calls are traced:

| Depth | Trigger | What is traced |
| --- | --- | --- |
| `single` (default) | `--depth single`, "single layer", or nothing said | The target service (proxy) plus **one hop**: the service it calls directly. Calls made by that service are listed in `downstream` but not opened. |
| `deep` | `--depth deep`, "deep", "deep analysis", "analisis mendalam", "sampai ke logic terdalam" | The target service plus **every service reachable** through outgoing HTTP calls, recursively, until a service makes no further calls. Every hop's DTO, response, queries and relations are documented. |

If the user does not state a depth, use `single` without asking.

`RUNTIME_ROOT` is this skill's directory. Run scripts with absolute paths, e.g. `node <RUNTIME_ROOT>/scripts/validate.mjs`.

## Swagger annotation gap-fill

After the DOCX is written, also fill missing NestJS Swagger annotations in the **target project only**. Downstream `SVC_*` repos stay read-only (read their DTOs, never edit them). NestJS only. Follow `spec/SWAGGER_ANNOTATION_SPEC.md` exactly.

- Add `@ApiTags('<Name>')` on a controller class only when missing.
- Add `@ApiOperation({ summary, description })` on handlers missing it (English, house style).
- For a handler param typed `any` (`@Body() body: any`, `@Query() params: any`): trace to the downstream DTO, create a proxy DTO class with `@ApiProperty({ required, example })` per field, and attach via `@ApiBody({ type })` / `@ApiQuery({ type })`. **Never change the handler signature.**
- Never add `@ApiResponse`, `@ApiConsumes`, `@ApiProduces`, `@ApiParam`, `@ApiBearerAuth`, or `@ApiProperty` to pre-existing real DTOs.
- Idempotent: only fill gaps; never rewrite or duplicate. Edits are written only after confirmation.

## References

- `spec/API_DOCUMENTATION_SPEC.md` — content rules for every part of the document and metadata defaults. Read it before writing JSON.
- `spec/example-api-document.json` — a complete, valid document with a forwarded endpoint and a local one. Match its shapes, tone, and level of detail.
- `schema/api-document.schema.json` — the structural contract the validator enforces.
- `spec/SWAGGER_ANNOTATION_SPEC.md` — exact annotation format and DTO rules for the swagger gap-fill phase. Read it before planning swagger edits.
- `spec/DOCX_RENDERING_SPEC.md` — renderer styling only. Do not read it unless changing `scripts/generate.mjs`.

## Workflow

1. **Resolve paths.**
   - `TARGET_ROOT`: the file or folder the user gave.
   - `<target-name>`: the folder name, or the filename without extension.
   - `PROJECT_ROOT`: the nearest ancestor of the target with a project manifest (`package.json`, `go.mod`, `*.csproj`, `*.sln`, `Cargo.toml`) that is inside the git root; fall back to the git root. In a monorepo this is the app folder (e.g. `apps/api`), never `src/` or a feature module.
   - `WORKSPACE_ROOT`: the folder holding all the service repositories — the current working directory when the target's repository is inside it, otherwise the parent folder of the target's git root.
   - `DEPTH`: `single` or `deep`, per [Analysis depth](#analysis-depth).
2. **Collect endpoints.** Find every endpoint anywhere under the target, recursively, unless the user named specific ones. Each endpoint is one item in `endpoints`; keep endpoints from the same module/subfolder next to each other. When the same handler is exposed under several routes (e.g. `/capaian/...` and `/web/capaian/...`), write one item and list the other routes in `aliases`.
3. **Check for existing output.** Only the exact files `<PROJECT_ROOT>/docs/<target-name>.json` and `.docx` count. If they exist, ask before overwriting. Other files in `docs/` are not a reason to ask: leave them untouched and list them in the final report as possibly obsolete.
4. **Install dependencies** if `<RUNTIME_ROOT>/node_modules/docx` does not exist: `node <RUNTIME_ROOT>/scripts/init.mjs`.
5. **Analyze the target service.** Start at each route handler and follow what it needs: DTO/validation, service, repository/entity/ORM model or raw SQL, HTTP client, and guards. Skip `node_modules`, build output, generated code, and unrelated modules. Trace shared pieces (guards, HTTP client wrapper, base entities) once and reuse the findings.
6. **Resolve outgoing HTTP calls to repositories.** A call whose base URL comes from an env var or config key — e.g. `${SVC_REPO_PROJECT_B}/api/v1/foo`, `process.env.SVC_REPO_PROJECT_B`, `configService.get('SVC_REPO_PROJECT_B')`, `os.Getenv(...)`, `IConfiguration[...]` — points at another service. Follow the HTTP client wrapper if there is one to find the env var and path.
   - **Map the env var to a folder by name.** Drop affixes such as `SVC_`, `SERVICE_`, `_URL`, `_HOST`, `_BASE_URL`, `_API`; lowercase; turn `_` into `-` (`SVC_REPO_PROJECT_B` → `repo-project-b`). Look for that folder directly under `WORKSPACE_ROOT`, first as an exact match, then as a unique folder whose name contains it.
   - **When there is no match or several matches**, collect every unresolved env var and ask the user once, in a single question, which folder each maps to. In `deep` mode, new env vars can surface at every hop; batch the ones found at each hop into one question rather than asking per call. If the user says the repo is not available, document the call from the caller's side only.
   - Do not read `.env`, helm, k8s, terraform, or CI files to resolve names.
7. **Analyze the called services.** In the mapped repository, find the handler for the called method and path, accounting for global prefixes, controller prefixes, and API versioning. From it, document the request DTO and validation, the response shape, the database queries, and the table relations, exactly as in step 5. How far to go depends on `DEPTH`:
   - **`single`: one hop only. Do not follow that service's own outgoing calls.** If repo B calls repo C, list the call in `downstream` with `handler` set to `-` and move on. Still resolve C's name with the same folder mapping (a folder lookup only, no code opened); if no folder matches, use the env var name exactly as written. Never make up a service name.
   - **`deep`: follow every outgoing call recursively.** If repo B calls repo C, resolve C (step 6), open its handler, and document it the same way; repeat for C's calls, and so on, until a handler makes no further HTTP calls. Every hop gets a `downstream` row with its real `handler`. Track visited `service + method + path` pairs and do not re-open one already traced (this also stops call cycles such as B → C → B); reuse the earlier findings instead. Use `-` as the `handler` only when the repo is not available.
   - For any call not made by the target service itself, append `(dipanggil oleh <caller-service>)` to `call` so readers can follow the chain.
   - Tag every query and relation with the service that owns it.
   - `flow` describes the whole chain in plain words, in source order, naming each service when it is called.
   - **Request fields come from what the target service accepts.** If the target passes the body or query through untouched, use the called service's DTO and say so in `request.dto`.
   - **The response is what the target service returns.** If it returns the called service's response as-is, use that shape. If it reshapes the data, document the final shape.
8. **Write** `<PROJECT_ROOT>/docs/<target-name>.json` following the spec and example. Fill metadata with the defaults in the spec, and list every service involved in `services`.
9. **Validate:**

   ```sh
   node <RUNTIME_ROOT>/scripts/validate.mjs <PROJECT_ROOT>/docs/<target-name>.json
   ```

   If it fails, fix the JSON and validate again. Do not render invalid JSON.
10. **Render:**

    ```sh
    node <RUNTIME_ROOT>/scripts/generate.mjs <PROJECT_ROOT>/docs/<target-name>.json <PROJECT_ROOT>/docs/<target-name>.docx
    ```

    Confirm the DOCX exists and is not empty.
11. **Plan swagger edits (NestJS targets only).** If the target project is NestJS, scan its controllers and build a plan: controllers missing `@ApiTags`, handlers missing `@ApiOperation`, `any` params needing a proxy DTO (with the fields traced from the downstream service), and the DTO files to create. Show the plan and **ask once** whether to apply it. The DOCX is already generated regardless. If the target is not NestJS, skip this step and say so.
12. **Apply swagger edits (only if the user confirms).** Apply the plan per `spec/SWAGGER_ANNOTATION_SPEC.md`: add annotations, create DTO files, add imports. Edit the target project only. Make no git commit, push, or other git change. If the user declines, skip.
13. **Report** in chat: the target, the project root, the analysis depth used, the services traced (with each env var → folder mapping), the JSON and DOCX paths, the validation result, the swagger edits proposed/applied/skipped, and any facts that could not be determined from source.

## Rules

- **The document states facts about the API, never about the analysis.** Do not write how a mapping was decided, which repos or files were not opened or traced, which env values or `.env` files were not checked, or side observations such as bugs and inconsistencies. Put anything the user needs to know about the analysis in the final report (step 13) instead.
- Service names are always the real repository folder names. Never derive a name from an env var when a matching folder exists.
- Source code is the only source of facts. Never infer authentication, status codes, validation rules, table names, or relations without evidence in source. `handler`, `request.dto`, and `downstream[].handler` carry the file and symbol so engineers can check.
- Write field types in everyday words (`Teks`, `Angka`, `Ya/Tidak`, `Tanggal dan jam`, `Daftar`), adding the format when it matters (`Teks (UUID)`, `Angka bulat, 1 sampai 100`).
- Every request and response field explains what it means for the user or the business process, not just its type.
- Relations come from entity/model definitions (`@ManyToOne`, `belongsTo`, GORM tags, EF navigation properties), migrations, or explicit joins in queries. A cross-service link through an ID stored without a foreign key is a relation too: mark it as `Referensi lewat API/ID, tanpa foreign key`.
- When a required value has no evidence in source, state the plain fact about the API (e.g. `Tidak perlu login` when no guard applies), never `null`, a made-up value, or a remark about the analysis. Use an empty array when a list genuinely has no entries.
- If the target contains no HTTP endpoint, or a route cannot be traced to a handler, stop and tell the user what was found instead of producing a partial document.
- **Swagger edits touch the target project only.** `SVC_*` repos are read-only; never edit files there.
- **Swagger changes are gap-fill and idempotent** — never rewrite, duplicate, or reformat existing annotations, and never change handler signatures.
- **Swagger text is English**; the DOCX stays Bahasa Indonesia.
- **The DOCX is always produced**; only writing the swagger edits is gated on confirmation.
- **No git operations** — edits are working-tree changes only; never commit, push, or amend.
