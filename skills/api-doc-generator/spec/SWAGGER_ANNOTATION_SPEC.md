# Swagger Annotation Specification

> This spec governs the swagger gap-fill phase (workflow steps 11–12). It is for NestJS targets
> only. Annotation text is **English**; the DOCX stays Bahasa Indonesia.

## When this applies

The target project is NestJS when its `package.json` depends on `@nestjs/common` **and** it has at
least one `*.controller.ts` containing `@Controller(`. If it is not NestJS, skip the whole phase and
say so in the report; the DOCX is still produced.

**Only the target project is edited.** Downstream `SVC_*` / `db-*` repos are read-only: open them to
read the real DTO, never write to them.

## What to add

Gap-fill only. Before inserting anything, check whether it already exists and skip if so. Never
rewrite, duplicate, or reformat an existing annotation, and never change a handler signature.

### 1. `@ApiTags` on the controller class

Add only when the class has no `@ApiTags`. Place it next to `@Controller(...)` (either order is fine;
match the file's existing style).

```ts
import { ApiOperation, ApiTags, ApiBody, ApiQuery } from '@nestjs/swagger';

@ApiTags('User Needs')
@Controller('user')
export class UserController {
```

Name: a short human-readable group for the controller's domain (e.g. `User Needs`, `Tob`, `Option`).
Prefer the same wording the controller already uses in route comments; otherwise derive it from the
controller path.

### 2. `@ApiOperation` on each handler

Add only when the handler has no `@ApiOperation`. Place it above the HTTP-verb decorator.

```ts
  @ApiOperation({
    summary: 'Get Role',
    description: 'Get Role using param',
  })
  @Get('role')
  getRoleByModule(@Query() params: any) { ... }
```

- `summary`: a short phrase naming the action (e.g. `Get Role`, `Patch Blokir Aplikasi`).
- `description`: one short sentence about the endpoint. Keep it English and factual.
- Match the target file's existing `@ApiOperation` style where one is already present.

### 3. Body / query schema for `any` params

When a handler parameter is typed `any` (`@Body() body: any`, `@Query() params: any`) the generated
spec has no schema. Fix it by creating a proxy DTO and attaching it — **without touching the
handler signature**.

Steps:

1. Trace the handler's downstream call (workflow steps 6–7) to the `SVC_*` repo and find the real
   DTO the service uses for that request.
2. Create a DTO class in the **target project** with one `@ApiProperty` per field of that real DTO.
3. Attach it with `@ApiBody({ type: XxxDto })` (body) or `@ApiQuery({ type: XxxDto })` (query).

```ts
  @ApiBody({ type: CreateMenuBodyDto })
  @Post('menu')
  createMenu(@Body() body: any) { ... }   // signature unchanged
```

```ts
export class CreateMenuBodyDto {
  @ApiProperty({ required: true, example: 'Dashboard' })
  name: string;

  @ApiProperty({ required: false, example: 1 })
  order?: number;
}
```

Rules:

- **One DTO per `any` param.** A handler with both an `any` body and an `any` query gets two DTOs,
  each attached to its own decorator.
- **Never invent fields.** Only fields present in the downstream DTO.
- **No `class-validator` decorators** on created DTOs — only `@ApiProperty`. The signature is left as
  `any`, so no runtime validation is introduced.
- If the downstream repo/folder cannot be resolved, skip that DTO and note it in the report; still
  add `@ApiTags` and `@ApiOperation`.

## DTO creation details

- **Naming.** Match the target project's existing DTO naming convention. If unclear, use
  `<Action><Context>Dto` in PascalCase, e.g. `CreateMenuBodyDto`, `QueryRoleBodyDto`.
- **File location.** Follow the project's existing convention:
  - if `<PROJECT_ROOT>/src/dto/` exists, put the file there;
  - otherwise, if a module-local `dto/` folder exists next to the controller, put it there;
  - otherwise, create module-local `dto/<name>.dto.ts` next to the controller.
  Never import a DTO from the `SVC_*` repo.
- **Imports.** Extend the existing `@nestjs/swagger` import (add `ApiBody` / `ApiQuery` /
  `ApiProperty` as needed); add the new DTO import line next to the other DTO imports. Never
  duplicate an import.

## Field derivation from the downstream DTO

For each field of the downstream DTO:

| `@ApiProperty` option | Source |
| --- | --- |
| field name | the downstream DTO's property / `json` name |
| `required` | `false` when the field is `@IsOptional()` or optional (`?`); otherwise `true` |
| `example` | the downstream field's own `@ApiProperty({ example })` if present; else the first `@IsIn` / enum value; else a realistic literal matching the field's type |

TypeScript type: mirror the downstream field's type (e.g. `string`, `number`, `boolean`, `string[]`).

## Do not add

These are not part of the target project's house style and must not be introduced:

`@ApiResponse`, `@ApiOkResponse`, `@ApiCreatedResponse`, `@ApiConsumes`, `@ApiProduces`,
`@ApiParam`, `@ApiBearerAuth`, `@ApiExcludeEndpoint`, `@ApiExtraModels`.

Do not add `@ApiProperty` to pre-existing real DTOs; only created proxy DTOs get `@ApiProperty`.

## Notes

- If the project enables the `@nestjs/swagger` CLI plugin (`nest-cli.json` → `introspectComments`),
  explicit `@ApiProperty` on the created DTO is still fine and takes precedence.
- If a handler already has `@ApiOperation` but an `any` body, add only the body/query DTO and its
  decorator; leave the existing `@ApiOperation` untouched.
- No git operations: edits are working-tree changes only. Never commit, push, or amend.
