# Rebuild prompt — Consortium OS on your own external Supabase

Copy everything inside the block below into a **new blank Lovable project** (connect your own Supabase first, from Connectors → Supabase, and do NOT enable Lovable Cloud).

---

```text
Build "Consortium OS" — a private, English-only, EU-grade collaboration workspace for the NGO
"Smart Society Development" (SSD) to build funding consortia and co-write EU proposals with partners.

BACKEND
Use my own external Supabase project (already connected). Do not enable Lovable Cloud.
Stack: TanStack Start + React 19 + Vite + Tailwind + shadcn/ui.
All backend logic goes through createServerFn (no Supabase Edge Functions).
Protected routes live under src/routes/_authenticated/ with the managed ssr:false auth gate.
Register the Supabase bearer attacher as functionMiddleware in src/start.ts.

DATABASE MIGRATION (one migration; every CREATE TABLE in public followed by GRANTs, then RLS, then policies)
enum app_role: super_admin, coordinator, partner_admin, partner_member, reviewer

Tables:
- organizations: name, short_name, country, country_code, pic_number, org_type, website,
  contact_person, contact_email, phone, expertise, capabilities, previous_projects,
  target_groups, infrastructure, staff, proposed_contribution, timestamps
- profiles: id -> auth.users.id, email, full_name, position, phone, bio, org_id, timestamps
- user_roles: user_id, role app_role, unique(user_id, role)   [roles NEVER on profiles]
- projects: code, title, programme, abstract, call_url, deadline, total_budget, status,
  coordinator_org_id, created_by, timestamps
- project_members: project_id, email, user_id, org_id, role app_role, invited_at, joined_at
- work_packages: project_id, number, title, objective, lead_org_id, deliverables, milestones,
  kpis, start_month, end_month, budget, progress
- tasks: project_id, wp_id, title, description, status, assignee_org_id, assignee_user_id,
  due_date, position
- ideas: project_id, author_id, title, body, ai_analysis jsonb, status
- idea_votes: idea_id, user_id, unique(idea_id, user_id)
- messages: project_id, wp_id, author_id, body
- documents: project_id, name, category, storage_path, size_bytes, mime_type, uploaded_by
- proposal_sections: project_id, part, section_key, title, content, status, position,
  contributed_by, contributed_org_id
- budget_lines: project_id, org_id, wp_id, category, description, amount
- call_requirements: project_id, requirement, needed_expertise, best_org_id, evidence, status
- evaluations: project_id, kind, scores jsonb, summary, created_by
- audit_log: user_id, action, entity, detail jsonb

Security-definer helper functions (search_path = public):
- has_role(_user_id uuid, _role app_role)
- is_project_member(_project_id uuid)      -> super_admin OR row in project_members
- can_manage_project(_project_id uuid)     -> super_admin OR coordinator/partner_admin member
- can_contribute(_project_id uuid)         -> super_admin OR any member except reviewer
- shares_project_with_org(_org_id uuid)    -> super_admin OR shares any project with that org

RLS: every project-scoped table SELECT via is_project_member(project_id),
INSERT/UPDATE via can_contribute(project_id), DELETE via can_manage_project(project_id).
organizations readable via shares_project_with_org. profiles: own row + super_admin, no delete.
user_roles: SELECT only from the client; all writes server-side with the service role.
audit_log: SELECT for super_admin only; inserts server-side.

Storage: private bucket "project-documents" with policies scoped to project membership;
downloads always via time-limited signed URLs.

Seed in the same migration: organization "Smart Society Development" (Estonia, EE) and project
DIGITAL-2026-BESTUSE-10-NETWORKSICs "Safer Internet Centres" with SSD as coordinator,
plus WP1..WP6 skeleton rows.

AUTHENTICATION
Passwordless magic link only. A public landing page (src/routes/index.tsx) with SSD branding and
an email field. Server function requestAccessLink: look up the email in project_members with the
service role; only then send the magic link. Always return a neutral
"if you are invited, a link has been sent" — never reveal membership.
Bootstrap rule: if there are zero auth users, the first sign-in is allowed and becomes super_admin.
Server function claimAccess (behind requireSupabaseAuth): upsert the profile, attach pending
project_members rows by email, grant super_admin if none exists, write to audit_log.
The _authenticated layout calls claimAccess on first visit.

DESIGN
Institutional European look derived from the SSD logo: deep navy sidebar, circuit-green accent,
cyan highlight, white surfaces, generous whitespace, light + dark mode, subtle motion.
Fonts: Space Grotesk headings, IBM Plex Sans body. All colours as semantic tokens in src/styles.css.
Fully responsive down to 360px. No purple/indigo gradients, no Inter/Poppins.

APP STRUCTURE
AppShell with navy sidebar: Dashboard, Projects, Organizations, Admin (super_admin only), Profile,
theme toggle, sign out.
Shared UI bits: PageHeader, StatCard, ProgressBar, ScoreRing, Countdown, EmptyState.
Data layer in src/lib/queries.ts — React Query hooks over Supabase for roles, projects, members,
organizations, work packages, tasks, ideas, votes, messages, documents, proposal sections,
budget lines, call requirements, plus generic useUpsert / useRemove mutations.

Routes:
- /                                   public landing + magic-link sign-in
- /dashboard                          my projects, deadline countdowns, readiness + progress
- /projects                           project list
- /projects/$projectId/$tab           workspace with tabs
- /organizations                      searchable partner directory (PIC, country flag, expertise)
- /profile                            edit own contact details and bio
- /admin                              super_admin: register organizations, open calls, invite partners, audit log

Workspace tabs (one component each under src/components/project/):
1. Overview     readiness factors, gap alerts, deadline countdown, call requirements checklist
2. Consortium   coordinator vs partners, country flags, org chart, invite partner form
3. Workflow     work-package manager + Kanban task board per WP
4. Ideas        idea board with voting/comments + live project chat (Supabase Realtime)
5. AI           call analysis, section drafting, EU evaluator simulation
6. Budget       budget lines with per-partner and per-category totals
7. Proposal     Excellence / Impact / Implementation section editor + "Compile Proposal" export
8. Library      document upload to storage, categories, signed download links

SCORING
src/lib/scoring.ts computes a Consortium Readiness score out of 100 from partner complementarity,
geographic coverage, WP completeness, proposal section completeness and budget allocation,
and explains exactly what is missing.

AI (server-side only, Coordinator and Super Admin)
Use the Lovable AI Gateway from server functions; never expose keys to the browser.
- analyzeCallText: extract requirements -> needed expertise -> best partner -> evidence -> status
- consortium gap analysis (technical, geographic, management, duplicate capacity)
- idea analysis: relevance, innovation, feasibility, impact, risks, suggested WP and KPIs
- draftProposalSection: formal EU prose per section, always hand-editable
- evaluateProposalDraft: strengths, weaknesses, missing requirements, indicative scores for
  Excellence / Impact / Implementation

SEO / metadata
Every route defines its own head() with a unique title, description, og:title, og:description,
og:type and twitter:card. Landing page uses the SSD logo and a single H1.

BUILD ORDER
1. Migration + RLS + storage bucket + seed
2. Design tokens, logo, landing page, magic-link auth, _authenticated gate
3. AppShell, queries.ts, dashboard, projects, organizations, profile, admin
4. Workspace tabs: overview, consortium, workflow, budget, proposal, library
5. Ideas + realtime chat, AI workspace, evaluation, readiness score, compile/export
```

---

## After the new project is created

1. Run the migration against your own Supabase.
2. Sign in once with your own email — that first account becomes super_admin.
3. Import your existing data: export the SQL from this project, then
   `psql "<your-connection-string>" -f export.sql` (data only, since the new schema already exists).
4. Re-upload the files from the `project-documents` bucket.
5. Publish from Lovable and point `projects.smartsociety.ee` at it via a CNAME in Hostinger.

Note: Hostinger Business shared hosting cannot run this app (it is PHP + MySQL, while this app needs a
Node/Edge runtime and PostgreSQL). Use Hostinger only for DNS, or deploy to Vercel/Netlify/Cloudflare.