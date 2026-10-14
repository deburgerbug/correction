# correction

Three layouts are one layout copied. AdminLayout and EmployeeLayout differ only in nav items, brand color, label letter, and which metrics endpoint they poll (30s vs 60s). CustomerLayout is the same shell with a light theme. Each re-implements the nav-link style function, logout handler, and polling effect.
No primitive components. Repeated verbatim: the spinner class string (7x), three variants of the uppercase field label (14 uses), the primary button class (3x), the bordered input class (3x), the centered auth wrapper (5x). There is no Button, Input, Field, Badge, Card, Spinner or PageHeader. The only reusable overlay is DeleteConfirmModal.
Every page owns its data plumbing. 10 files hand-write loading/error state; TicketListPage and CustomerListPage carry 9 to 10 useState calls each for filters, paging and fetch state. Status and priority color maps are defined as functions inside TicketListPage, so any other page that shows a status must redefine them (likely true for TicketDetailsPage; not confirmed).
AG Grid is used on 4 list pages for plain server-paginated lists, and two of them add hand-built pagination around it. That is a heavy dependency solving a problem a single DataTable component solves.
Routes are tripled by role. /admin, /employee and /customer each repeat customers and support routes; pages derive basePath from user.role at runtime to compensate. Authorization is expressed as role names in routing but as permissions in the backend, so the two models are mixed.
Auth pages: 5 pages, about 760 LOC, 12 inline SVGs (password visibility toggles) duplicated across them.
Likely over-split: CustomerForm is 7 files with Card/Section pairs (AddressCard+AddressSection, CommunicationCard+CommunicationSection). Wrapper-plus-child pairs are the usual source of “unnecessary components”. Needs a read to confirm.
Dead weight: CustomerPage is a 17-line wrapper that only renders CustomerForm; the admin “My Profile” route renders a “coming soon” div; EmployeeDashboard renders one number from a full tickets query (limit 1 just to read totalCount).

REBUILD SEQUENCE (do in this order; each step unblocks the next)

Step 1. Fix the foundation decisions before touching any page. Design tokens in the Tailwind theme (one brand color, neutral scale, radius, spacing, type scale, status and priority semantic colors) so no page uses raw color names. One light theme with one set of tokens replaces the three nav themes.

Step 2. Build the primitive set, each used by at least three call sites or not built: Button (variants, loading state), Field (label, input, error, hint as one unit), Select, Badge (status and priority variants driven by one map), Card, Spinner and Skeleton, PageHeader, EmptyState, ErrorState, Modal with ConfirmDialog, Toast. Use headless accessible primitives (Radix or shadcn/ui on Tailwind 4) instead of hand-rolling dropdowns and dialogs.

Step 3. Build composites on top: DataTable with built-in pagination and filter bar (replaces AG Grid), StatCard, CommentThread, AuthCard layout plus PasswordField, and AddressFields and ContactFields as single field-group components so CustomerForm becomes one file plus two groups.

Step 4. Replace per-page data plumbing. One typed request helper (replaces the repeated fetch/parse/throw in all six API files) and TanStack Query hooks per resource (useTickets, useTicket, useCustomers, and so on). Loading, error, caching and polling (the 30s/60s badge refresh) move out of pages. Forms move to react-hook-form with one zod schema shared with the backend validation.

Step 5. One AppShell. Sidebar navigation (scales better than a top bar as modules are added) driven by a single nav config where each item declares its required permission. Route tree becomes one set of paths (/customers, /tickets, /employees, /departments) guarded by permission, not by role prefix. This deletes the three layouts, role-based basePath logic, and the tripled routes.

Step 6. Rebuild pages lowest risk first: auth pages, then shell, then ticket list and detail, then customers, then employees and departments, then dashboards (merge the two dashboards into one config-driven page of StatCards).

Step 7. Remove AG Grid and its CSS, the placeholder routes, and the pass-through wrappers. Add a single internal component catalogue page (or Storybook) and a11y checks (keyboard, focus, ARIA, contrast) on the primitives.

TARGETS (my estimates, to hold the rewrite accountable)

Client under about 3,000 LOC from 5,809.
No Tailwind class string repeated more than twice outside primitives.
No page over 150 lines; no page contains fetch, loading or pagination state.
One layout, one route tree, one status/priority map.

Constraint: the backend defects I listed earlier (public role selection on register, reset token returned by forgot-password, fallback JWT secrets) are still exploitable. The UI rewrite does not touch them, so fix those first or in parallel.

Decision needed before Step 1: confirm the stack (Tailwind 4 retained, Radix/shadcn primitives, TanStack Query, react-hook-form plus zod) and whether TypeScript migration is bundled into the rewrite. Once confirmed, I will specify the Step 1 token set and the exact component inventory with props contracts.

Assumptions (reversible): Tailwind 4 stays; new code is TypeScript with allowJs during the transition; libraries are shadcn/ui (Radix, source copied into your repo), TanStack Query, TanStack Table, react-hook-form with zod, sonner for toasts, lucide-react for icons, and the react-router data router.

Current error handling, measured: 54 setError calls, 47 hand-styled red banners, 7 alert() calls and 2 window.confirm calls for failures and destructive actions, 42 catch/console.error sites, no error boundary, and unknown URLs silently redirect instead of showing a 404. The API client hard-reloads the page to /login on session loss. The backend returns status 400 for nearly every failure with no error code and no per-field errors, so the UI cannot distinguish error types. Fix that contract first (Step 1).

STEP 1. BACKEND ERROR CONTRACT (prerequisite for good UI errors)

Define one error envelope for every failure: correct HTTP status, a stable machine code (for example VALIDATION_FAILED, UNAUTHENTICATED, FORBIDDEN, NOT_FOUND, CONFLICT, RATE_LIMITED, INTERNAL), a user-safe message, an optional fieldErrors map keyed by field path, and a requestId.
Replace bare throw new Error in services with typed errors that carry status and code. The error middleware maps Mongoose ValidationError, duplicate-key and CastError into the envelope, and returns a generic message for unknown errors (log the detail server-side only).
Attach a request ID to every request and return it in a response header and in the envelope.
Done when: wrong password gives 401 with a code, missing ticket gives 404, a bad form gives 422 with fieldErrors, an unexpected fault gives 500 with a generic message.

STEP 2. PROJECT SETUP AND STRUCTURE

Create a branch. Install the libraries above. Enable TypeScript with strict mode; set a path alias for src.
Folder layout: app (providers, router, shell), components/ui (primitives), components/shared (composites), features/{auth, tickets, customers, employees, departments} each holding its own api, hooks, schemas, components and pages, lib (http client, error normalization, query client), styles (tokens).
Rule: pages compose; features own their data hooks; ui knows nothing about the API.
Done when: the app builds and runs with the new structure holding one placeholder page.

STEP 3. DESIGN TOKENS AND VISUAL DIRECTION

Define CSS-variable tokens in the Tailwind theme: one brand accent, a neutral scale, semantic colors (success, warning, danger, info), status and priority colors, radius scale, shadow scale, 4px spacing grid, a type scale (14px base for dense data views, one UI font such as Inter), and focus-ring style. Define light values and dark values now even if you ship light first; tokens make dark mode nearly free later.
Visual direction: calm neutral surfaces, one accent, generous whitespace, tables with 40 to 44px rows, status shown as dot-plus-label badges, soft borders instead of heavy shadows, consistent icon set, subtle 150 to 200ms transitions, visible keyboard focus.
Ban raw color classes in pages; only token classes. Add an ESLint rule or review checklist for this.
Done when: a single token file controls every color, radius and font in the app.

STEP 4. PRIMITIVES (build only what is used 3+ times)
Button (variants, sizes, loading state that blocks double-click), Field (label, control, hint and error as one unit, wired with aria-describedby), Input, Select, Textarea, Checkbox, Badge (status and priority variants from one map), Card, Skeleton, Spinner, Avatar, Tabs, Dropdown menu, Dialog, ConfirmDialog (title, consequence text, destructive style, async confirm with in-dialog error), Toast, Tooltip, PageHeader, EmptyState, ErrorState, Alert.
Done when: a component catalogue page renders every primitive in every state (default, hover, focus, disabled, loading, error).

STEP 5. ERROR HANDLING ARCHITECTURE (core of the request)

Normalize once. One function turns anything thrown (network failure, timeout, abort, non-JSON response, API envelope) into a single AppError with kind, status, code, user message, fieldErrors, requestId and a retryable flag. UI code reads only AppError, never raw responses or strings.
One copy table. Map code to user-facing text in a single file. Message style: what happened plus what to do next; never show raw server text or stack traces. This also makes later i18n trivial.
Present by scope:
Field error: inline under the field, linked by aria-describedby; on submit focus the first invalid field.
Form-level error (non-field): Alert at the top of the form with role alert; user input is preserved on failure.
Action error (delete, assign, change status, post comment): toast with a Retry action where the action is safe to repeat; the control returns to its prior state.
Query error (page or section data): ErrorState replacing only the failed region, with Retry and the request ID in small text so support can trace it.
Crash error: route-level errorElement plus a root error boundary with Reload and Go home; report to a logging hook.
404: a real NotFound page; stop redirecting unknown paths to the root.
403: inline “You do not have access” state with a way back; never log the user out and never loop redirects.
Session expiry: single-flight refresh stays; on failure navigate with the router (no full reload), show one “Session expired” toast, and return the user to the page they were on after login.
Offline: listen to online/offline events and show a persistent banner; disable mutations while offline.
Every data region has exactly three non-happy states: Skeleton shaped like the content (not a lone spinner), EmptyState with a call to action, ErrorState with retry. No blank screens.
Destructive actions: ConfirmDialog stating the consequence (for example what happens to the customer’s tickets), failure shown inside the dialog, dialog stays open. Remove every window.confirm and alert.
Mutations: disable the trigger while pending; optimistic update with rollback plus toast for status and assignee changes; never retry mutations automatically; retry GET requests with backoff, capped, and not on 4xx.
Forms: validate on blur first, then on change after first submit; one zod schema per form (mirror it on the backend); map server fieldErrors back onto fields; warn on navigating away with unsaved changes; keep the comment draft if posting fails.
Done when: you can trigger each error kind deliberately (kill the server, expire the token, submit invalid data, request a missing ticket, throw inside a component) and each produces the correct presentation from the list above.

STEP 6. DATA LAYER

One typed HTTP client: attaches the token, parses the envelope, throws AppError. Base URL from an environment variable.
A query hook and mutation hook per resource under each feature (tickets list, ticket detail, customers, employees, departments, metrics). Query keys centralised. The 30s and 60s badge polling becomes refetchInterval on the metrics hooks.
Filters and page number live in the URL search params, so refresh, back button and shared links work. This replaces the 9 to 10 useState calls per list page.
Done when: no page contains fetch, loading flag, error string or pagination state.

STEP 7. APP SHELL AND ROUTING

One AppShell: sidebar on desktop, drawer below the tablet breakpoint, top bar with search placeholder, notifications badge, user menu (profile, sign out). Navigation comes from one config array; each item declares the permission required and the badge hook it uses.
One route tree with permission guards (/tickets, /customers, /employees, /departments, /dashboard, /profile), replacing the /admin, /employee and /customer prefixes and the runtime basePath logic. Use the data router so errorElement works per route.
Auth pages use one centred AuthCard layout with a shared PasswordField (show/hide via lucide icon).
Done when: three layouts and the tripled routes are deleted and every role lands on a correct, permission-filtered shell.

STEP 8. REBUILD PAGES (in this order; each is shippable alone)

Auth set (login, register, forgot, reset, verify): the fastest win for reusability and the first place to exercise form and field errors.
Tickets list: DataTable with URL-synced filters, status and priority badges, assignee avatar, row click, skeleton and empty states. On mobile render rows as cards.
Ticket detail: two columns. Left: conversation thread with a sticky composer; internal notes visually distinct (tinted background, lock icon) and the internal toggle labelled clearly. Right: status, priority, assignee, customer and timestamps, with inline editing and optimistic updates.
Customers list and detail: detail becomes a summary header plus tabs (Overview, Tickets, Activity) instead of one 461-line page. The form becomes one form component with AddressFields and ContactFields groups, add/remove rows, primary marker, and a pincode lookup with its own loading and failure state that never blocks manual entry.
Employees and departments: DataTable plus dialog forms (create, edit) instead of separate pages where the form is small.
One dashboard driven by StatCards per permission: needs-attention tickets, unassigned, my open tickets, recent activity; each card has its own loading and error state so one failed query does not blank the page.
Done when: each rebuilt page is under about 150 lines and uses only ui and shared components plus its feature hooks.

STEP 9. CLEANUP AND QUALITY GATES

Delete AG Grid and its CSS, old layouts, DeleteConfirmModal, the placeholder profile route, pass-through wrappers and unused API functions.
Accessibility pass: keyboard-only walkthrough of every flow, focus management in dialogs, labels on all inputs, role alert and status live regions, contrast check on tokens.
Tests: component tests for primitives and the error presentation matrix; end-to-end tests for login, create ticket, assign, comment, expired session, offline. These double as proof for the error behaviour.
Measure against the targets: client under about 3,000 LOC, no class string repeated more than twice outside primitives, one layout, one route tree.

Estimated effort for one developer (rough): Steps 1 to 3 about 2 to 3 days, Steps 4 to 6 about 5 to 6 days, Step 7 about 2 days, Step 8 about 6 to 8 days, Step 9 about 2 to 3 days.

Start with Step 1 and Step 2 now. Reply with the backend error-envelope fields you want, or say “go” and I will take Step 1 and specify each service and middleware change in order, without code unless you request it.
