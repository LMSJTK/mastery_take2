# Mastery-Based Training System — Implementation Plan

## Overview

Transform the existing playbook-based automated training system into an adaptive, mastery-focused training platform built on top of Moodle. The system will tag course content against NIST CSF 2.0 Functions, generate individualized proficiency paths using an LLM, adapt to learner performance, and visualize multi-dimensional competency via radar charts.

---

## Phase 1: Content Tagging & Metadata Schema

**Goal:** Establish the data foundation that everything else depends on.

### 1.1 Database Tables (Moodle plugin or external DB)

Create tables to store content metadata outside of Moodle's native schema (Moodle courses stay in Moodle; we add a metadata layer):

- **`csf_functions`** — Reference table for the 6 NIST CSF 2.0 Functions
  - `id`, `code` (GV/ID/PR/DE/RS/RC), `name`, `description`
  - Seed with the 6 Functions. Optionally include CSF Categories (22) and Subcategories for finer-grained tagging later.

- **`content_tags`** — Maps Moodle courses to CSF Functions with relevance scores
  - `id`, `moodle_course_id`, `csf_function_id`, `relevance_score` (0.0–1.0)
  - A course can have multiple rows (one per function it covers)

- **`content_metadata`** — Additional per-course metadata for pathing
  - `id`, `moodle_course_id`, `difficulty_score` (1–5 or 1–10), `mastery_points` (points earned on completion), `estimated_duration_minutes`, `content_type` (enum: course, assessment, email_simulation, video, etc.), `created_at`, `updated_at`

- **`content_prerequisites`** — DAG of prerequisite relationships
  - `id`, `course_id`, `prerequisite_course_id`

### 1.2 Admin Tagging Interface

Build a UI for in-house experts to tag and score content:

- List all Moodle courses (pulled via Moodle DB or web services API)
- For each course: set difficulty, mastery points, content type, duration
- Tag with one or more CSF Functions + relevance score per function
- Set prerequisite courses
- Bulk import/export via CSV for initial population

### 1.3 Work Items

- [ ] Design and finalize the DB schema
- [ ] Write migrations (or Moodle plugin install.xml if going the plugin route)
- [ ] Build admin tagging UI (likely a Moodle local plugin or standalone admin panel)
- [ ] Create CSV import tool for initial bulk tagging
- [ ] Seed CSF Functions reference data
- [ ] Expert review and tagging of existing course catalog

---

## Phase 2: Learner Mastery Tracking

**Goal:** Track per-learner, per-CSF-function mastery scores derived from course completions.

### 2.1 Database Tables

- **`learner_mastery`** — Running mastery scores per learner per CSF function
  - `id`, `moodle_user_id`, `csf_function_id`, `mastery_score` (accumulated), `last_updated`

- **`mastery_events`** — Audit log of every mastery-affecting event
  - `id`, `moodle_user_id`, `moodle_course_id`, `csf_function_id`, `points_earned`, `course_grade`, `event_type` (completion, assessment, simulation_response), `timestamp`

### 2.2 Mastery Calculation Logic

When a learner completes a Moodle course:

1. Look up the course's CSF tags (content_tags) and metadata (content_metadata)
2. For each tagged CSF function: `points_earned = mastery_points × relevance_score × grade_factor`
   - `grade_factor` = normalized course grade (e.g., 85% → 0.85)
3. Write mastery_events rows
4. Update learner_mastery running totals
5. Trigger proficiency path branching evaluation (Phase 3)

### 2.3 Integration with Moodle

- Hook into Moodle's **course completion events** (`\core\event\course_completed`)
- Either via a Moodle local plugin that listens to events, or via scheduled task that polls `mdl_course_completions`
- Pull grades from Moodle gradebook (`mdl_grade_grades`)

### 2.4 Work Items

- [ ] Design mastery calculation formula (decide on scaling, caps, grade weighting)
- [ ] Build event listener or scheduled task for Moodle course completions
- [ ] Implement mastery score calculation service
- [ ] Create learner_mastery and mastery_events tables
- [ ] Consider mastery decay (optional: scores degrade over time to drive re-engagement)
- [ ] Unit tests for mastery calculation edge cases

---

## Phase 3: Proficiency Path Generator

**Goal:** Use an LLM to generate adaptive learning paths from the tagged content catalog.

### 3.1 Database Tables

- **`proficiency_paths`** — Generated path definitions
  - `id`, `name`, `description`, `created_by` (operator user ID), `focus_functions` (JSON array of CSF function IDs + weight), `target_mastery_level`, `status` (draft/active/archived), `generation_params` (JSON — LLM prompt config, model used, etc.), `created_at`, `updated_at`

- **`path_steps`** — Ordered content within a path, with branching
  - `id`, `path_id`, `step_order`, `moodle_course_id`, `track` (enum: standard, challenge, remediation), `branch_condition` (JSON — e.g., `{"min_grade": 80}` for standard→challenge, `{"max_grade": 60}` for standard→remediation), `email_simulation_id` (nullable FK — if this step uses an informative sim instead of standard enrollment email)

- **`learner_path_assignments`** — Which learners/cohorts are on which paths
  - `id`, `path_id`, `moodle_user_id` (nullable for cohort assignment), `moodle_cohort_id` (nullable), `current_step_id`, `current_track`, `assigned_at`, `status` (active, completed, paused)

### 3.2 LLM Path Generation Flow

1. Operator selects target CSF functions and desired emphasis/weighting
2. System queries content catalog for all courses tagged with those functions
3. System builds a structured prompt containing:
   - Available courses with their tags, difficulty, mastery points, prerequisites, content types
   - The selected CSF focus areas and weights
   - Constraints (e.g., max path length, desired pacing, available email simulations)
4. LLM generates a structured response (JSON) containing:
   - Ordered standard track steps
   - Branch points with challenge and remediation alternatives
   - Placement of email simulations within the flow
   - Rationale for the ordering (shown to operator for review)
5. Operator reviews, adjusts, approves the generated path
6. Path is saved and becomes assignable

### 3.3 Adaptive Branching at Runtime

When a learner completes a step:

1. Evaluate branch_condition against the learner's grade/performance
2. Route to next step on appropriate track (standard, challenge, or remediation)
3. If remediation track: after remediation content, route back to standard track
4. Update learner_path_assignments with current position
5. Trigger enrollment in next Moodle course + send appropriate messaging

### 3.4 Work Items

- [ ] Design the LLM prompt template for path generation
- [ ] Build the path generation service (API call to LLM, parse structured response)
- [ ] Build operator UI: select focus areas, generate path, review/edit, approve
- [ ] Implement branching engine (evaluates conditions, routes learners)
- [ ] Build automated enrollment + messaging for next-step courses
- [ ] Integrate email simulation placement into path steps
- [ ] Handle edge cases: content retired mid-path, learner already completed a course in the path, etc.

---

## Phase 4: Informative Email Simulations

**Goal:** Create teaching-oriented email simulations that serve as both learning content and enrollment triggers.

### 4.1 Database Tables

- **`email_simulations`** — Informative simulation definitions
  - `id`, `name`, `threat_type`, `difficulty_score`, `csf_function_id`, `email_subject`, `email_body_html`, `annotations_html` (the "here's how to spot this" overlay/highlights), `next_course_id` (FK to Moodle course this links to), `mastery_points`, `created_at`

- **`simulation_responses`** — Learner interactions with simulations
  - `id`, `simulation_id`, `moodle_user_id`, `delivered_at`, `opened_at`, `link_clicked_at`, `annotations_viewed`, `response_data` (JSON for any interactive elements)

### 4.2 Integration Points

- Delivery via existing email infrastructure (or new if needed)
- Tracking via unique tracking pixels/links per learner per simulation
- These are NOT the existing phishing simulator — different product, different intent
- Links in the simulation email go to an annotation/education page, then to course enrollment
- Completion of the simulation (viewing annotations + clicking through) counts as a mastery event

### 4.3 Work Items

- [ ] Design email simulation template format
- [ ] Build simulation authoring interface
- [ ] Build delivery mechanism (integrate with existing mailer or build new)
- [ ] Build tracking (opens, clicks, annotation views)
- [ ] Build the annotation/education landing page
- [ ] Connect simulation completion to mastery tracking (Phase 2)
- [ ] Tag simulations with CSF functions and difficulty (same schema as courses)

---

## Phase 5: Proficiency Dashboard (Radar Chart Visualization)

**Goal:** Multi-dimensional visualization of cohort and individual mastery across CSF functions.

### 5.1 Database Tables / Views

- **`cohort_mastery_cache`** — Pre-computed cohort averages (refreshed periodically or on-demand)
  - `id`, `moodle_cohort_id`, `csf_function_id`, `avg_mastery_score`, `min_score`, `max_score`, `member_count`, `computed_at`

### 5.2 Visualization

- Radar/spider chart with 6 axes (one per CSF Function)
- Each axis scaled 0–100% of "full proficiency" (define max mastery per function)
- **Default view:** Single shaded polygon showing cohort average
- **Toggle overlay:** Click individual learners to overlay their polygon on the cohort average
  - All individuals off by default (privacy)
  - Manager/operator can toggle specific learners on/off
- **Drill-down:** Click a spoke/axis to see breakdown by Category within that Function
- **Comparison mode:** Overlay multiple cohorts (e.g., department A vs. department B)
- **Export:** PDF/PNG export of the chart for reporting

### 5.3 Technology Choices

- Frontend charting library: Chart.js (has radar chart type built in), D3.js (more customizable), or Apache ECharts
- If building as Moodle plugin: embedded in Moodle page via a local plugin
- If standalone dashboard: separate web app consuming an API

### 5.4 Work Items

- [ ] Design proficiency scoring scale and "full proficiency" thresholds per function
- [ ] Build API endpoints for cohort and individual mastery data
- [ ] Build cohort mastery cache and refresh mechanism
- [ ] Implement radar chart component
- [ ] Add individual learner overlay toggle
- [ ] Add drill-down by CSF Category
- [ ] Add cohort comparison mode
- [ ] Add export functionality
- [ ] Build the dashboard page/view (Moodle plugin or standalone)

---

## Phase 6: Operator/Manager Path Assignment & Management

**Goal:** Let operators and managers assign proficiency paths to learners and cohorts, and monitor progress.

### 6.1 UI Features

- Browse available proficiency paths (filtered by CSF focus, difficulty, length)
- Assign path to individual learners or Moodle cohorts
- View assignment status: who's on which path, which step, which track
- Pause/resume/reassign learners
- Manager digest emails (evolve the existing digest to show mastery progress, not just completion)

### 6.2 Work Items

- [ ] Build path browsing and selection UI
- [ ] Build assignment workflow (individual + cohort)
- [ ] Build progress tracking view (table + status indicators)
- [ ] Update existing manager digest to include mastery data
- [ ] Role-based access control (who can generate paths, assign paths, view dashboards)

---

## Phase 7: Messaging & Notifications Overhaul

**Goal:** Replace the existing playbook-based welcome/reminder emails with mastery-aware messaging.

### 7.1 Changes

- Welcome emails now reference the proficiency path, not just the course
- Reminders are context-aware: "You're on step 3 of 8 in your Detect & Respond path"
- Positive reinforcement on mastery milestones: "You've reached 75% proficiency in Protect!"
- Email simulations replace standard enrollment emails at configured path steps
- Manager digests show mastery radar charts (small inline versions) instead of just completion counts

### 7.2 Work Items

- [ ] Design new email templates for mastery-aware messaging
- [ ] Build template rendering with mastery data context
- [ ] Integrate with path step transitions (trigger emails on step completion/advancement)
- [ ] Build milestone notification logic
- [ ] Update manager digest generation

---

## Cross-Cutting Concerns

### Architecture Decision: Moodle Plugin vs. External Application

| Approach | Pros | Cons |
|---|---|---|
| **Moodle local plugin** | Native integration, uses Moodle's DB/auth/events, single deployment | Constrained by Moodle's PHP stack, harder to use modern frontend frameworks, Moodle upgrade risk |
| **External app + Moodle API** | Tech stack freedom (Python/Node + React/Vue), easier LLM integration, independent deployment | Needs Moodle web services API access, separate auth, more infra to manage, data sync complexity |
| **Hybrid** | Plugin for Moodle event hooks + data sync, external app for LLM generation + dashboard | Best of both worlds but more components to maintain |

**Recommendation:** Hybrid approach. A lightweight Moodle local plugin handles event listening (course completions), data sync, and admin UI within Moodle. A separate service handles LLM path generation and serves the proficiency dashboard (which benefits from a modern JS frontend).

### API / Integration Layer

- Moodle Web Services API for reading courses, users, cohorts, enrollments, grades
- Custom API (REST) between the external service and the Moodle plugin for mastery data, path definitions, simulation tracking
- LLM API (Claude API or similar) for path generation

### Data Privacy & Access Control

- Mastery data is PII-adjacent (individual performance data)
- Respect Moodle's role system for access control
- Dashboard visibility governed by Moodle roles (manager sees cohort, admin sees all, learner sees own)
- GDPR/privacy considerations for data export and deletion

### Migration from Playbook System

- Existing playbook system continues to run for MSP customers
- New mastery system is an alternative mode, not a forced replacement
- Possible migration path: convert a playbook's course sequence into a "flat" proficiency path (no branching) as a starting point, then enhance

---

## Suggested Implementation Order

1. **Phase 1** (Content Tagging) — Foundation; everything depends on this. Start here.
2. **Phase 2** (Mastery Tracking) — Core engine. Can be built in parallel with expert tagging work.
3. **Phase 5** (Dashboard) — Provides immediate visible value once Phase 1+2 data exists. Ship early for stakeholder buy-in.
4. **Phase 3** (Path Generator) — The big feature. Needs Phase 1 data to be meaningful.
5. **Phase 6** (Assignment UI) — Operational layer on top of Phase 3.
6. **Phase 4** (Email Simulations) — Can be developed in parallel with Phase 3 but integrated after.
7. **Phase 7** (Messaging Overhaul) — Last mile; depends on most other phases being functional.

---

## Rough Dependency Graph

```
Phase 1 (Tagging)
  ├──→ Phase 2 (Mastery Tracking)
  │      ├──→ Phase 5 (Dashboard)  ← ship this early
  │      └──→ Phase 3 (Path Generator)
  │             ├──→ Phase 6 (Assignment)
  │             ├──→ Phase 4 (Email Sims)
  │             └──→ Phase 7 (Messaging)
  └──→ Phase 3 (also depends directly on tagging data)
```
