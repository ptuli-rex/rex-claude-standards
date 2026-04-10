# Rex Code Conventions

## General

- Write clean, readable code. Prefer clarity over cleverness.
- Don't add features, refactor, or "improve" code beyond what was asked.
- Don't add comments, docstrings, or type annotations to code you didn't change.
- Only add error handling at system boundaries (user input, external APIs).

## Git

- Commit messages: short imperative summary, then body explaining why.
- Format: `feat(scope): description`, `fix(scope): description`, etc.
- Always commit and push before creating release PRs.
- Never force-push to main or prod.
- Never skip pre-commit hooks (--no-verify).

## Backend (NestJS / TypeScript)

- Package manager: **pnpm** (not npm).
- Follow existing module structure: `module.ts`, `controller.ts`, `service.ts`, `dto/`, `models/`.
- Every schema change needs a migration file in `migrations/` (raw SQL, idempotent).
- DTOs use class-validator decorators. Global ValidationPipe strips unknown properties.
- Use `@Public()` decorator only for webhook/unauthenticated endpoints.
- Prefer `Logger` from `@nestjs/common` over `console.log` in new code.

## Frontend (React / TypeScript / MUI)

- Package manager: **yarn**.
- Use path aliases (`@components/*`, `@utils/*`, `@pages/*`, etc.), never relative paths.
- All user-facing text must use react-intl `<FormattedMessage>`. Add keys to both en.json and es.json.
- Use MUI `sx` prop for one-off styles, `styled()` for reusable components.
- Responsive design: use `useIsMobile()` hook. Drawers switch anchor on mobile.
- Forms: react-hook-form with Controller pattern.
- Notifications: react-hot-toast.
