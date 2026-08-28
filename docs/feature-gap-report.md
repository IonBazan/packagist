# Feature gap review

A read of the current feature set against the work people actually do every day — moderating
spam, chasing a broken update, running an organization — and the thirty places where the app
makes that work harder than it needs to be.

Reviewed from source and templates at `043e68b` without installing dependencies, so nothing here
is verified against a running instance. Route coverage came from the `#[Route]` attributes,
permissions from `security.yaml`, the voters and the `IsGranted` call sites. Effort labels
(XS/S/M/L) are relative to this codebase's existing patterns, not absolute estimates.

Priority: **now** = blocks daily work, **next** = removes recurring friction, **later** = worth
doing, not urgent.

---

## 1. What exists today

The app is substantially further along than upstream packagist.org on moderation tooling.
Everything proposed below is additive to this.

| Area | Implemented |
| --- | --- |
| Packages | Submit, edit repo URL, delete; abandon/unabandon with reason; maintainer add/remove/transfer; version soft-delete, hide, recover; dependents, suggesters, providers; download and PHP-version stats |
| Moderation | Suspect-package queue with ML classifier scores; package freeze (spam, malware, gone, temporary, remote-id mismatch); user freeze with package cascade; frozen-user review queue; malware filter lists with bulk toggle; nine fine-grained staff roles |
| Audit | Transparency log with 50+ record types; filters on actor, user, vendor, package, date; wildcard search and PII for auditors; indexed search table; per-organization audit log; moderation activity feed on `/admin/` |
| Organizations | Event-sourced aggregate with projections; admin-only creation with 2FA-gated owner; owners and all-members teams; invitations (send, resend, revoke, expire); rename, slug change, slug reservations |
| Accounts | Registration with email verification; GitHub OAuth login and account link; TOTP 2FA, backup code, trusted device; brute-force protection and reCAPTCHA; main + safe API token with rotation; favourites; GitHub hook sync |
| Public API | Composer v2 metadata and changes feed; create/edit/update package; Algolia search with type and tag facets; security advisories; download tracking; job status; six RSS/Atom feeds |

---

## 2. Admin & moderation

The individual moderation *actions* are well built. What's missing is everything around them:
finding the subject, seeing the queue, acting on more than one row, and closing the loop.

### ADM-01 — There is no way to look up a user or a package (now, M)

**Today.** Every moderation task starts with an exact URL. You reach a user only at
`/users/{name}/` and a package only at `/packages/{vendor}/{name}`, so you need the exact
username or package name before you can do anything. There is no search by email, GitHub id,
partial name, or anything else.

**Proposal.** An `/admin/users` directory searching username, email and GitHub id, with filters
for freeze reason, 2FA state, registration window, package count and GitHub link; and an
`/admin/packages` counterpart filtering on freeze state and reason, suspect flag, abandoned,
repository host and crawl staleness. The Pagerfanta + `QueryFilter` pattern from the transparency
log drops straight in.

**Evidence.** `UserRepository` has no search method beyond `findOneByUsernameOrEmail()` (used only
by admin org creation); `MenuBuilder::createAdminMenu()` offers five links, none a lookup.

### ADM-02 — No queue dashboard; you cannot see what the workers are doing (now, M)

**Today.** `Job` carries nine statuses and is indexed on type, status, creation, start and
completion, but the only UI reading any of it is the per-package update history. Nothing shows the
queue as a whole: no count of errored or timed-out jobs, no oldest-queued age, no sign a worker
has stopped, no way to requeue without the CLI.

**Proposal.** `/admin/jobs` listing by type, status and age, with per-row requeue
(`Job::reschedule()` already exists) and cancel, plus a worker-liveness tile derived from the
newest `startedAt`.

**Evidence.** `JobRepository` exposes only `getPackageUpdateJobsQueryBuilder()`,
`getLastGitHubSyncJob()`, `getScheduledJobIds()` and `markTimedOutJobs()`.

### ADM-03 — Moderation queues have no bulk actions (now, S)

**Today.** The suspect-package queue lists fifty packages a page and offers exactly one action —
mark a vendor safe. Freezing a package as spam means opening each package page and submitting the
freeze form there, one at a time. The frozen-users queue offers no action at all.

**Proposal.** Row checkboxes with bulk freeze-as-spam and bulk delete on the suspect queue, and
bulk unfreeze on the frozen-users queue. `Admin\FilterListController::bulk()` already implements
exactly this pattern (CSRF, selected ids, per-row audit records, one flush), so it is a port.

**Evidence.** `Admin\SpamController` has `viewSpamAction` + `markSafeAction` only.

### ADM-04 — Staff roles can only be granted by editing the database (now, S)

**Today.** The role hierarchy defines nine distinct staff capabilities and the code checks them
carefully, but roles live in a JSON column with no UI, so granting the filter-list role means an
`UPDATE` against `fos_user`. There is also no audit record type for a role change — the one class
of event that most needs a trail is the one the transparency log cannot see.

**Proposal.** An `/admin/staff` page under `ROLE_SUPERADMIN` listing role holders with grant and
revoke, writing a new `UserRolesChanged` audit record on every change.

**Evidence.** `User::$roles` is `#[ORM\Column(type: 'json')]` with `addRole()`; no controller
writes it. `AuditRecordType` has no role case.

### ADM-05 — The admin home is one feed with no queue counts (next, S)

**Today.** `/admin/` shows the last twenty moderation events and nothing else. To learn whether
anything needs attention you visit each queue in turn.

**Proposal.** A row of tiles above the feed: suspect packages pending, frozen users by reason
(overdue temporary holds called out), jobs errored in the last 24 hours, filter-list entries added
since last visit, invitations expiring this week. Each links to its queue and respects the
viewer's role.

**Evidence.** `templates/admin/index.html.twig` renders `recentModerationActivity` only;
`PackageRepository::getSuspectPackageCount()` already exists.

### ADM-06 — Freezing an account never tells the account (next, S)

**Today.** The freeze form already separates a public reason from an internal note — a design that
anticipates telling the user. Nothing sends it. Freezing writes an audit record and a flash; the
user discovers the freeze by failing to log in. Package freezes behave the same way.

**Proposal.** A "notify the account" checkbox on both freeze forms, sending the public reason and
an appeal address through the existing `UserNotifier`.

**Evidence.** `UserController::freezeUserAction()` persists `AuditRecord::userFrozen()` and
flashes; `UserNotifier` is called only for email changes and 2FA events.

### ADM-07 — Temporary holds have no review date and go stale silently (next, M)

**Today.** The frozen-users queue defaults to Temporary because those are, in the code's own
words, "the holds meant to be revisited". But a hold records only its reason and timestamp — no
due date, no overdue sort, no assignee.

**Proposal.** A `reviewAfter` date set when a temporary hold is applied, the queue sorted by most
overdue, and snooze actions (+7 / +30 days) that write an audit record. Extend to
`PackageFreezeReason::Temporary`.

**Evidence.** `User` stores `frozen` + `frozenAt`; `FrozenUserController` reconstructs context by
re-querying the latest `UserFrozen` audit record per row.

### ADM-08 — No way to pivot from one bad account to the ring behind it (next, M)

**Today.** Audit records store the actor's IP and auditors may see it, but IP is not a filter and
not in the search index. When you catch one spam account you cannot ask who else registered from
there, linked that GitHub id, or signed up in the same ten minutes.

**Proposal.** An IP / CIDR filter on the transparency log for `ROLE_AUDITOR`, and a "related
accounts" panel on the user profile matching shared IP, GitHub id, email domain and a narrow
registration window. Pairs directly with ADM-03.

**Evidence.** `security.yaml`: `ROLE_AUDITOR` "grants access to full audit records including PII
(IP, emails, ..)"; `AuditLogSearchType` indexes names only.

### ADM-09 — The audit log cannot be exported or linked to (next, S)

**Today.** Filters are good and the paginated view is fast, but a single record has no URL of its
own, and there is no CSV or JSON output, so any incident write-up is assembled by hand.

**Proposal.** A `/transparency-log/{id}` record page; `.csv` / `.json` formats on the list route
for auditors, capped and rate-limited; per-record pivot links to "everything by this actor" and
"everything for this package".

**Evidence.** `TransparencyLogController::viewAuditLogs()` is the only route; the table template
supports `linkSurroundingLogs` but no single-record view.

### ADM-10 — Admin organization tooling stops at list and create (next, M)

**Today.** Organizations can only be created by staff, which makes staff the support path for
everything that follows — yet there is no admin org detail page, no way to intervene in membership
when an owner leaves, and no soft-delete. `OrganizationStatus::Deleted` and
`OrganizationActions::SoftDelete` / `Restore` are declared and marked not implemented. Slug
reservations are documented as releasable "only by a packagist-admin", and no interface releases
one.

**Proposal.** An admin org detail view (members, teams, invitations, audit in one place), owner
transfer for the stranded-org case, soft-delete and restore behind the existing events, and a
slug-reservation release action.

**Evidence.** `Admin\OrganizationController` = `list()` + `create()`; `SlugReservation`'s docblock
names the admin release path.

### ADM-11 — Spam triage runs on the CLI while the UI shows the scores (later, S)

**Today.** The suspect queue computes a classifier score per row and shows whether the model would
auto-clear each package, but acting on those scores in bulk requires
`packagist:triage-spam-queue` on a shell. Same for `packagist:clean-spam-packages`,
`packagist:transfer-ownership` and `packagist:unfreeze-package`.

**Proposal.** A threshold control on the queue ("clear everything scored below *x*") with a
preview before applying, and per-package "force update" / "reindex" actions.

**Evidence.** `SpamController::computeSpamScores()` returns `{metadata, readme, safe}` per package;
`TriageSpamQueueCommand` is the only bulk consumer of that judgement.

### ADM-12 — User impersonation is configured but has no interface (later, XS)

**Today.** `switch_user` is enabled on the main firewall and `ROLE_SUPERADMIN` carries
`ROLE_ALLOWED_TO_SWITCH`, but no template emits a `_switch_user` link.

**Proposal.** An impersonate action on the profile with a persistent "viewing as" banner, an exit
link, and an audit record on entry. Cheapest large win for reproducing a user's bug report.

**Evidence.** `config/packages/security.yaml` firewall `main` declares `switch_user`; no grep hit
for `_switch_user` in templates.

### ADM-13 — Feature killswitches require a deploy (later, S)

**Today.** Five killswitches shed load by disabling stats, dependency links and package pages —
the right levers, checked in fifteen places — but they are PHP constants, so pulling one under
load means editing a file and shipping it.

**Proposal.** Back the switches with Redis (constant as the default) and add an admin page to flip
them, showing current state and who flipped it last. Keep the `Killswitch::isEnabled()` call sites
unchanged.

**Evidence.** `src/Util/Killswitch.php` — five `public const` booleans.

---

## 3. Structural gaps

Two features are substantially implemented but not connected to the thing that would make them
useful. Both have the hard parts done and are waiting on a relation and a permission check.

### STR-01 — Organizations do not own anything (now, L)

**Today.** There is a full organization subsystem — aggregate, event store, projections, teams,
invitations, audit log, admin creation — and no relation anywhere between an organization and a
package or vendor. The only point of contact is the slug guard, which checks that whoever claims
an org slug already maintains a package under that vendor prefix. Teams therefore grant nothing:
package permissions still resolve through the maintainer collection and staff roles alone, so a
member added to the owners team gains access to no packages.

**Proposal.** Bind vendors to organizations, teach `PackageVoter` to resolve through team
membership as well as the maintainer list, and list org-owned packages on the organization
overview. Until this lands, everything built on organizations is a membership directory.

**Evidence.** No `organization` reference in `Package` or `Vendor`; `OrganizationSlugClaimGuard`
is the only class touching both domains, via `PackageRepository::isVendorTaken()`.

### STR-02 — One fixed pair of API tokens per account (next, M)

**Today.** Every account has exactly two tokens, main and safe, and rotation replaces both at
once, so one leaked CI token means re-pasting credentials into every integration. Nothing records
when a token was last used or from where, so a token cannot be checked for liveness before
revoking, and the compromised one cannot be revoked alone.

**Proposal.** Named tokens with scopes, keeping the existing `ApiType::Safe` / `Unsafe` split as
the scope backbone (create, edit, update-only). Record last-used timestamp and IP, allow
individual revoke and optional expiry, and keep the current pair working as legacy tokens.

**Evidence.** `User::$apiToken` / `$safeApiToken`, both `length: 40`;
`ProfileController::rotate_token` regenerates both together.

---

## 4. Maintainer experience

### USR-01 — Maintainers cannot see why their own package stopped updating (now, S)

**Today.** A maintainer whose package stopped updating gets one toast with the last failure. The
full history — every run, its timing, its log — is staff-only, deliberately, because the log
carries internal exception classes. The most common support question has no self-service answer,
even though the data is in the job table.

**Proposal.** A maintainer-facing view of the same list showing timestamps, duration, status and
the already-sanitised `result.message`, withholding payload, result JSON and raw exception detail.
Gate it on the `update` voter rather than the role.

**Evidence.** `view_package_update_history` is `#[IsGranted('ROLE_UPDATE_PACKAGES')]`; its docblock
explains the role gate is because the history "exposes internal exception classes/messages".

### USR-02 — Notification settings are a single checkbox (now, M)

**Today.** The profile form offers "Notify me of package update failures" and nothing else. Seven
transactional mails exist, none controllable, and none covering a new advisory against a package
you maintain, a maintainer added to your package, or an ownership transfer.

**Proposal.** A notifications page with per-event toggles — update failures, advisories affecting
my packages, maintainer and ownership changes, new releases of favourites — plus an
immediate-or-weekly-digest choice. A single `notificationPreferences` JSON column covers it.

**Evidence.** `ProfileFormType` adds `failureNotifications` only; `templates/email/` holds seven
unconditional templates.

### USR-03 — Security advisories are a catalogue, not a service (next, M)

**Today.** Advisories can be browsed globally, per package, and queried through the API — all
read-only, all pull. A maintainer with thirty packages has no page answering "which of mine are
affected right now", and no way to be told when that changes.

**Proposal.** An "advisories affecting my packages" panel on the profile, an opt-in alert mail on
new matches, and an advisory RSS/Atom feed alongside the six `FeedController` already serves.

**Evidence.** `security_advisories`, `view_advisory`, `view_package_advisories` and
`api_security_advisories` are all listings; no per-user query exists.

### USR-04 — Favourites collect packages but produce nothing (next, S)

**Today.** Favourites are stored in Redis and rendered as a list. Feeds exist for new packages, new
releases, a vendor, a single package and extensions — but not for the set the user curated, so
following ten packages means subscribing to ten feeds.

**Proposal.** A token-authenticated `/feeds/favorites.{rss,atom}` and an optional weekly digest.
The feed builder and the release query both already exist.

**Evidence.** `FeedController` has six routes; `user_favorites` renders a plain package list.

### USR-05 — No session or device management (next, M)

**Today.** Remember-me cookies last a year and 2FA trusted devices persist, but a user who loses a
laptop has one remedy: `sessionBuster`, which signs out every device everywhere. There is no list
of what is signed in, so an unexpected session cannot even be detected.

**Proposal.** An active-sessions and trusted-devices list (first seen, last seen, user agent,
coarse location) with individual revoke, and a sign-in alert mail for a new device.

**Evidence.** `remember_me.lifetime: 31104000` with
`signature_properties: ['password', 'sessionBuster']`; `scheb/2fa-trusted-device` installed with no
listing UI.

### USR-06 — TOTP is the only second factor, and 2FA is never enforced (later, M)

**Today.** Second-factor support is TOTP plus a single backup code — no passkeys, no hardware keys,
no recovery path beyond that one code. Organization creation requires the owner to have 2FA
enabled, but nothing re-checks it afterwards and nothing requires it of anyone else.

**Proposal.** WebAuthn as an additional factor, multiple single-use recovery codes, and an
org-level policy flag requiring 2FA of members, surfaced as a per-member column and enforced at
join time.

**Evidence.** `scheb/2fa-totp` + `2fa-backup-code`; `User::$backupCode` is a single `length: 8`
string. `Admin\OrganizationController::create()` checks `isTotpAuthenticationEnabled()` once.

### USR-07 — Search filters on type and tags, and nothing else (later, M)

**Today.** The Algolia index declares two facetable attributes, `type` and `tags`. The questions
people bring to a package search — does it support my PHP version, what licence, has it been
touched this year, is it abandoned — cannot be asked, even though abandonment is already indexed
for ranking.

**Proposal.** Add `abandoned`, `license`, a PHP-requirement bucket and a last-release bucket to
`attributesForFaceting`, expose them as facets, and default to hiding abandoned packages with a
visible toggle.

**Evidence.** `config/algolia_settings.yml` → `attributesForFaceting: ["type", "searchable(tags)"]`;
`abandoned` appears under `customRanking` only.

### USR-08 — No path to claim an abandoned vendor namespace (later, M)

**Today.** Ownership moves only from the inside: a current maintainer adds someone or transfers the
package. When the maintainer is gone, the only route is an email to `contact@packagist.org`,
handled off-platform with no record in the audit log.

**Proposal.** A claim request form on the package page opening an admin queue item, with evidence
attached (fork, GitHub org membership, last maintainer activity), a mandatory notice period to the
current maintainer, and an audit record on resolution.

**Evidence.** `add_maintainer` and `transfer_package` both require the `PackageVoter` to pass for
the acting user; `TransferOwnershipCommand` is the admin-side CLI escape hatch.

---

## 5. Quality-of-life wins

Each is a day or less and independently shippable.

| Ref | Item | Today | Change |
| --- | --- | --- | --- |
| QOL-01 | Dark mode | No `prefers-color-scheme` rule anywhere in `css/app.scss` | Token pass over the Bootstrap variables and a theme toggle |
| QOL-02 | Vendor pages | `view_vendor` renders a bare package list | Show the verified badge `Vendor` already stores, aggregate downloads, maintainers, owning organization |
| QOL-03 | Admin audit context | Frozen-user rows re-query the latest audit record per page load | Denormalise the freeze reason onto the user, or cache it — the queue does an N-row audit scan per render |
| QOL-04 | Suspect-queue side effect | Merely *viewing* `/admin/spam` auto-verifies any vendor with >10 downloads, then redirects | Make it an explicit action; a GET that mutates state and redirects to itself is surprising and hard to audit |
| QOL-05 | Localisation | Full translation plumbing, one catalogue: `messages.en.yml` | Either invite translations or drop the indirection |
| QOL-06 | Package page actions | Freeze, delete and edit are scattered across the page body | One staff action bar showing moderation state (frozen, reason, suspect, by whom, when) in place |
| QOL-07 | Invitation visibility | Pending invitations appear only inside the org's members page | Show invitations awaiting *you* on your own profile — today an invitee only learns from the email |

---

## 6. Checked, not a gap

Worth recording so these don't reappear on a future list.

- **Audit coverage is genuinely broad** — 50+ record types with dedicated display classes,
  including version soft-deletes, reference-change blocks and every organization event.
- **Role separation is real**, not decorative: nine roles, each checked at its own call site, with
  `ROLE_AUDITOR` properly gating PII and wildcard search.
- **Freeze semantics are carefully thought through** — the distinction between suppressing reasons
  (spam, malware: hidden and purged) and gentle ones (gone, temporary: still served, just not
  updated) is correct and consistently applied.
- **Bulk actions already exist for filter lists**, with CSRF, per-entry audit records and a single
  flush, which is why the same pattern is the recommendation for the other queues.
- **The search UI does expose type and tag facets** via InstantSearch, so USR-07 is about which
  attributes are facetable, not about missing UI.
- **Package transfer has a proper UI** with maintainer-list validation; the CLI command is an admin
  escape hatch, not the primary path.
