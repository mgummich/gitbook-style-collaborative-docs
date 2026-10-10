# GitBook-Style Collaborative Documentation Platform

## Summary

Build a self-hostable documentation and knowledge-management platform that combines GitBook's clean reading experience with Confluence-style collaboration. The first release serves both public product documentation and private company knowledge bases. Authors can edit in a browser or synchronize Markdown with any reachable Git repository.

The system uses separately deployable frontend, API, worker, database, queue, search, and object-storage services. It supports comments, mentions, roles, page history, review workflows, and controlled publishing, but excludes real-time co-editing and other advanced SaaS features from the first release.

## Goals

- Deliver an excellent public documentation reading experience.
- Support private and restricted internal knowledge spaces.
- Let authors use either a rich browser editor or Markdown in Git.
- Provide comments, mentions, revision history, roles, and approval workflows.
- Synchronize with any Git repository over HTTPS or SSH.
- Make installation, operation, backup, and upgrades practical for self-hosters.
- Keep services independently deployable without fragmenting the codebase.

## Non-goals for the first release

- Real-time multiplayer editing
- Custom domains managed by the application
- Advanced product analytics
- A third-party integration marketplace
- Native mobile applications
- Arbitrary MDX component execution
- Separate databases for each organization

## Product model

### Organizations and spaces

An installation can contain multiple organizations. An organization contains members and one or more spaces. Each space has its own page tree, access policy, publication state, Git connection, and member roles.

A space has one of three visibility modes:

- **Public:** the published revision is readable without authentication.
- **Private:** authenticated organization members can read it.
- **Restricted:** only explicitly assigned members can read it.

Draft content is never exposed through the public reader or public search index.

### Pages

Pages form an ordered tree and support drag-and-drop organization. Each page has a stable internal ID that does not change when its title, URL slug, parent, or Git path changes. A page may have an icon, cover image, template origin, redirects from previous slugs, and previous/next navigation.

The content model supports headings, paragraphs, lists, tasks, tables, callouts, code blocks, images, links, embeds, tabs, and other safe documentation primitives. Browser content has a deterministic Markdown representation so it can round-trip through Git. Unsupported or unsafe MDX components are not executed; the importer preserves them as fenced source or reports them as validation issues.

### Revisions and publishing

Drafts and published revisions are separate. Editing creates autosaved drafts without changing what readers see. Each immutable revision records its author, timestamp, source, content hash, and optional summary.

Publishing atomically changes the revision visible to readers. Restoring history creates a new draft from an older revision; it never mutates or deletes the historical record.

## User experience

### Reader

The reader closely follows GitBook's information architecture without copying its branding or proprietary assets:

- Collapsible page tree on the left
- Centered content column
- On-page heading outline on the right
- Prominent full-text search
- Breadcrumbs and previous/next navigation
- Responsive mobile navigation
- Light and dark themes
- Clear, restrained typography

Public pages use stable, crawlable URLs and appropriate metadata. Redirects preserve inbound links after page moves or slug changes.

### Authoring

The block-based editor provides slash commands, keyboard shortcuts, drag-and-drop blocks, asset upload, autosave status, and preview. Authors can create pages from templates, reorder the page tree, comment on selected content, mention teammates, and compare revisions.

When a page is part of a review, editing produces a new draft or updates the pending change according to permission. Review state is visible in the editor and page tree.

### Administration

The initial setup wizard configures the first owner, canonical site URL, email delivery, object storage, and authentication. Administrators can manage organizations, members, roles, space visibility, repository connections, audit logs, and system health.

## Roles and permissions

Roles are assigned per space:

- **Owner:** manages the space, integrations, members, settings, and publishing.
- **Editor:** creates and edits pages, manages drafts, resolves comments, and submits changes.
- **Reviewer:** reads drafts, comments, approves, or requests changes.
- **Viewer:** reads content permitted by the space visibility policy.

The API is the final authorization boundary. The web app may hide unavailable actions for usability, but backend permission checks govern every read and mutation. Search applies the same access rules and never returns restricted titles or snippets to unauthorized users.

## Collaboration workflow

1. An editor creates or changes one or more drafts.
2. The editor submits the change set for review with a summary.
3. Reviewers leave anchored comments, mention members, approve, or request changes.
4. Editors update the draft and resolve comments. Resolved threads remain in history.
5. An authorized reviewer or owner approves the change set.
6. Publishing atomically promotes the approved revisions.

Mentions and workflow events create in-app notifications. Email notifications are optional and depend on administrator configuration. The audit log records membership, permission, integration, review, and publication events.

## Git synchronization

### Connection

A space can connect to any Git repository reachable over HTTPS or SSH. The owner configures the repository URL, branch, content directory, credential, and synchronization mode:

- **Import only:** Git changes are imported into platform drafts.
- **Export only:** published platform changes are committed and pushed to Git.
- **Bidirectional:** both directions operate with conflict detection.

Repository credentials are encrypted at rest, redacted from logs, and never returned through the API after creation.

### File representation

Markdown files carry stable page IDs and supported metadata in YAML front matter. Directories and an optional navigation manifest describe the page tree. Assets may live in the repository or the configured object store. Export produces deterministic formatting to minimize noisy diffs.

PostgreSQL is the operational source of truth. A page view or edit never depends on live repository availability.

### Synchronization algorithm

The worker maintains a last-common-sync record containing the Git commit, platform revision, content hash, and path for each synchronized page.

Before a sync, it compares the current Git and platform states with that record:

- If only Git changed, it imports a new platform draft.
- If only the platform changed, it exports the appropriate published revision.
- If both sides changed to equivalent content, it advances the sync marker.
- If both sides changed differently, it creates a conflict and changes neither source.

An editor resolves a conflict by choosing the browser version, choosing the Git version, or manually merging. Resolution creates a new revision and then resumes synchronization.

Jobs are idempotent and locked per space. Failures use bounded retries with exponential backoff. Invalid Markdown, unreachable repositories, authentication errors, non-fast-forward pushes, and protected-branch rejections are recorded clearly and never replace the last valid published revision.

## Architecture

The product is a monorepo containing separately deployable services and shared packages.

### Web application

- Next.js and React
- Public reader, authoring interface, review tools, search, and administration
- Server rendering for public documentation
- Versioned API client generated from the API contract

### API service

- TypeScript, NestJS, and Fastify
- Versioned REST API
- Authentication, authorization, validation, content workflows, and audit logging
- OpenAPI contract used by clients and contract tests

### Worker service

- Git checkout, fetch, merge analysis, commit, and push operations
- Markdown parsing and deterministic serialization
- Search indexing
- Notification delivery
- Asset processing and other long-running work

### Data services

- **PostgreSQL:** organizations, users, memberships, spaces, page trees, drafts, revisions, comments, reviews, sync state, and audit records
- **Redis and BullMQ:** queued work, distributed locks, retries, and short-lived caching
- **Meilisearch:** full-text indexes split by public and access-controlled scope
- **S3-compatible storage:** uploaded images and files; MinIO is included in the default deployment

Shared packages define API schemas, editor primitives, permission rules, Markdown conversion, design tokens, and reusable UI components. Service boundaries communicate through versioned API contracts and versioned job payloads.

## Data flow

### Browser edit

1. The editor loads the current draft through the API.
2. The web app submits validated incremental saves with an optimistic concurrency token.
3. The API rejects stale writes with a conflict response and preserves both recoverable states.
4. Accepted changes create or update the draft and enqueue indexing or notification work as needed.
5. Approval and publication create immutable revisions in a database transaction.

### Reader request

1. The web app resolves the space, path, and published revision through the API.
2. The API applies visibility and membership rules.
3. Public responses may be cached; private responses remain user-scoped.
4. The reader renders the stored safe document model and asset URLs.

### Search

1. Publication or private-content changes enqueue an indexing job.
2. The worker builds normalized documents with space and permission metadata.
3. The API issues filtered search queries based on the caller's access.
4. Results link only to revisions the caller can read.

## Authentication and security

Local email/password authentication is available by default. OIDC can delegate authentication to systems such as Keycloak, Authentik, Okta, or Microsoft Entra ID. Session cookies are secure, HTTP-only, and protected against cross-site request forgery.

The platform sanitizes rendered content, validates uploads, restricts embed origins, applies rate limits to authentication and public APIs, and records security-sensitive actions. Secrets are supplied through environment variables or mounted secret files. Stored integration credentials use application-level envelope encryption with a separately supplied master key.

The supported Markdown subset does not execute arbitrary JavaScript or untrusted MDX components.

## Reliability and operations

The default distribution is Docker Compose with pinned image versions, health checks, readiness checks, persistent volumes, and conservative resource settings. Kubernetes manifests or a Helm chart may follow after the first release and are not required for initial acceptance.

All services emit structured logs with correlation IDs. Operators can inspect queue failures, Git sync history, index status, email delivery, and audit events. Database migrations are forward-versioned and run as an explicit deployment step.

Documented backup and restore procedures cover PostgreSQL and object storage. Meilisearch and Redis are rebuildable from durable state. Upgrade documentation identifies supported version paths and any required migration or rollback steps.

## Error handling

- Validation errors identify the affected field or content block and preserve user input.
- Stale edits return a conflict state with recovery options rather than overwriting newer work.
- Failed background jobs expose a human-readable status and an operator-level diagnostic record.
- Search outages degrade to navigation and direct page access.
- Email outages retain in-app notifications and retry delivery.
- Git outages or conflicts never block reading, browser editing, or access to the last published revision.
- Object-storage failures prevent incomplete publication of affected assets.

## Testing strategy

- Unit tests cover permission decisions, Markdown round trips, page-tree operations, review state transitions, and Git conflict detection.
- API integration tests run against real PostgreSQL and Redis services.
- Contract tests verify compatibility between the web app, API, and worker job payloads.
- End-to-end tests cover onboarding, editing, review, publication, search, visibility, permissions, comments, mentions, and each Git synchronization mode.
- Accessibility tests target WCAG 2.2 AA for reader and authoring workflows.
- Visual regression tests cover desktop and mobile reader/editor layouts in light and dark themes.
- Operational tests cover migrations, upgrades, backup/restore, job retries, and rebuilding derived services.
- Security tests cover authorization boundaries, unsafe Markdown, credential redaction, upload validation, and restricted search results.

## Acceptance criteria

The first release is complete when an operator can install the system from documented Docker Compose instructions, create an organization and space, invite members with appropriate roles, author and review pages in the browser, publish public or private documentation, search only authorized content, inspect and restore page history, and synchronize Markdown bidirectionally with an arbitrary HTTPS or SSH Git repository using explicit conflict resolution.

The reader must work responsively on current desktop and mobile browsers, and the documented backup/restore process must recover a functioning installation with its content, permissions, and assets intact.
