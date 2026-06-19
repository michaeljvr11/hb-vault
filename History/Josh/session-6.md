**Date:** 2026-06-19
**Card:** AP-9 — Admin portal: user management + admin NestJS module
**Branch:** feat/N8P6OPPm-admin-user-mgmt
**PR:** https://github.com/michaeljvr11/hb-mono-repo/pull/11

**What changed:**
- Added four shared interfaces to `@hb/shared` (AdminUserDto, UpdateUserRoleRequest, SetUserActiveRequest, AdminUserListQuery)
- Built `apps/api/src/admin/` module from scratch: AdminService + AdminController + 4 DTOs; registered in AppModule
- Built Angular `AdminUsersComponent` replacing placeholder; added `UsersService`
- 17 API unit tests + 20 Angular Vitest specs; 62 total passing

**Key decisions:**
- Used `TypeOrmModule.forFeature([User])` in AdminModule directly rather than importing UsersModule — avoids circular import
- requestingUserId passed explicitly to service methods so they stay unit-testable
- Self-role-change and self-deactivation blocked at service layer; self-reactivation allowed
- ESLint post-edit hook fires after every Edit and blocks on unused imports — had to update all call sites and add guard tests in a single atomic edit to prevent the hook from firing with BadRequestException imported but unused

**Test/review outcome:** 62/62 tests pass, lint clean, build clean (2 pre-existing stylesheet budget warnings unrelated to AP-9)

**Follow-ups:**
- AP-11 (audit log) will consume the AdminModule — AuditService should be added to AdminModule or a separate module imported there
- "Last admin" guard (cannot deactivate the last admin in the system) was descoped from AP-9; still in the card's acceptance criteria as a follow-up
