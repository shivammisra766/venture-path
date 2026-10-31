# Implementation Plan — Invite Existing Account Handling & Test User Reset

Status: shipped

## Objective
Prevent hiring manager invitees who already have an account from becoming trapped in a misleading "verification email sent" state by detecting existing accounts, directing eligible accounts to sign in, displaying role conflict notices for ineligible accounts, rejecting duplicate sign-ups, and resetting the test user `shivammisra766@gmail.com`.

## Current State
1. `/invite/[token]` only checks whether a session exists for the current browser. If unauthenticated, it always renders `<JoinInviteCard />` with a "Create account" form, even when the invited email is already registered in `user`.
2. When the user fills out "Create account" on `/invite/[token]`, `signUpForInviteAction` calls Better Auth's `signUpEmail`. Because `requireEmailVerification: true` is configured, Better Auth silently returns a synthetic duplicate response (no user creation, no verification email) to prevent user enumeration.
3. `signUpForInviteAction` treats this as a successful signup and returns `{ status: "sent", email }`. The UI transitions to `<CheckInbox />` ("We sent a verification link to...").
4. If the user clicks "Resend verification email", Better Auth sees `emailVerified: true` and no-ops. No email is ever sent, and no error is surfaced.
5. In the database, `shivammisra766@gmail.com` exists as a verified `user` role account with an active Google account link, preventing sign-up and invite acceptance.

## Desired State
1. `getInvite(token)` checks if a `User` record exists for `member.email` and returns `accountExists: boolean` and `existingRole: Role | null`.
2. On `/invite/[token]`:
   - If `accountExists` is true and `existingRole` is eligible (`null` or `hiring_manager`), the page informs the user that an account with their email already exists and renders a prominent "Sign in to accept invite" button and Google sign-in button.
   - If `accountExists` is true and `existingRole` is ineligible (`user` or `organisation`), the card explains that this email is already registered with another role and cannot be added as a hiring manager (per ADR-036), advising them to ask their organisation to invite a different email.
   - If `accountExists` is false, it renders the normal "Create account" form.
3. `signUpForInviteAction` checks `invite.accountExists` (or checks the DB). If an account already exists, it immediately returns an error toast (`An account with this email already exists. Please sign in to accept the invite.`) instead of calling `signUpEmail` and returning fake success.
4. The test user `shivammisra766@gmail.com` (`id: e3yKu4jwgIBPPOhhclCXPXTPv5bx3lXr`) is deleted from `account` and `user` tables in PostgreSQL as requested (Option 2), freeing the email to register as a fresh hiring manager.

## Blast Surface
- Files expected to change:
  - `src/lib/team.ts` (`getInvite` checks user existence and role)
  - `src/app/invite/[token]/page.tsx` (passes account existence and role to card)
  - `src/app/invite/[token]/join-invite-card.tsx` (renders sign-in UI or role conflict UI when account exists)
  - `src/app/invite/[token]/actions.ts` (`signUpForInviteAction` blocks existing accounts)
  - `tests/team.test.ts` (unit tests for `getInvite` with existing accounts)
  - `plan/README.md` (index updated)
- Files expected to be read but not changed:
  - `src/lib/auth.ts`
  - `src/lib/role-sign-in.ts`
  - `src/lib/roles.ts`
- Database:
  - Deletion of `account` and `user` rows for `e3yKu4jwgIBPPOhhclCXPXTPv5bx3lXr`.
- Dependencies: None.

## Implementation Sequence
1. Reset test user `e3yKu4jwgIBPPOhhclCXPXTPv5bx3lXr` in PostgreSQL.
2. Update `getInvite` in `src/lib/team.ts` to query `db.user` for `member.email` and return `accountExists` and `existingRole`.
3. Update `src/app/invite/[token]/page.tsx` to pass `accountExists` and `existingRole` to `<JoinInviteCard />`.
4. Update `src/app/invite/[token]/join-invite-card.tsx` to render:
   - "Account already exists: Sign in to accept invite" if eligible.
   - Ineligible role warning if already `user` or `organisation`.
   - "Create account" form only when no account exists.
5. Update `src/app/invite/[token]/actions.ts` in `signUpForInviteAction` to reject submissions if `invite.accountExists` is true.
6. Add unit tests in `tests/team.test.ts` verifying `getInvite` accurately reports `accountExists` and `existingRole`.
7. Run targeted and full tests (`vitest`, typecheck, lint).

## Test Strategy
- Vitest unit tests in `tests/team.test.ts`.
- `npm run typecheck` to verify TypeScript types across all invite routes.
- `npm run lint` for code style and conventions.
- Verify PostgreSQL state confirms user is removed and invite remains pending.

## Documentation Impact
- `plan/README.md`
- `docs/CHANGELOG.md`

## Risks & Mitigations
- Invite security / enumeration: The invite token is a high-entropy secret (32 random bytes) only possessed by the person who received the email. The email address is already rendered on `/invite/[token]`, so revealing whether that specific invited address has an account does not introduce any enumeration vector.
