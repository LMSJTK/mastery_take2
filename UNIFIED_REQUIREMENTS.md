# Mastery-Based Automated Awareness Training (AAT) System
# Unified Requirements & Implementation Plan

## Document Version
- **Version:** 2.0 (Draft)
- **Date:** March 2026
- **Status:** Requirements Consolidation
- **Inputs:** January 2026 OCMS Integration Requirements (v1.0), March 2026 Mastery System Plan, NIST CSF 2.0 / SP 800-61r3

---

## Executive Summary

This document unifies two prior efforts into a single specification for the next-generation Automated Awareness Training (AAT) system:

1. **January 2026 OCMS Requirements** — A detailed engineering spec for porting OCMS content management into Moodle as a plugin suite with adaptive, tag-based content selection.
2. **March 2026 Mastery Plan** — A product design for shifting from playbook-based training to mastery-focused proficiency paths aligned with the NIST Cybersecurity Framework (CSF) 2.0.

The unified system combines the Moodle plugin architecture and adaptive selection engine from the OCMS work with the NIST CSF alignment, proficiency path generation, informative email simulations, and radar chart visualization from the mastery plan.

### Key Differentiators from Legacy AAT

| Aspect | Legacy AAT | New AAT |
|--------|-----------|---------|
| Enrollment | Time-based schedule (playbooks) | Goal-driven, adaptive proficiency paths |
| Content Selection | Curated sets, one-size-fits-all | AI-selected based on NIST-aligned tags and learner performance |
| Taxonomy | None (course categories only) | Dual-layer: NIST CSF Functions + granular topic tags |
| Competency Model | Completion only | Mastery scoring with levels (novice → expert), trend tracking |
| Delivery Cadence | Monthly fixed schedule | Adaptive based on learner progress and path branching |
| Visualization | Completion counts in manager digest | Radar chart proficiency dashboard with cohort/individual overlays |
| Email Simulations | N/A (separate phishing simulator product) | Integrated informative simulations as learning path steps |
| User Authentication | Standard Moodle login | Standard Moodle + optional passwordless token access |
| Content Selection Mode | Sequential only | Three modes: AI Adaptive, Structured Path (with branching), Sequential |
| Framework Alignment | None | NIST CSF 2.0 (Govern, Identify, Protect, Detect, Respond, Recover) |

---

## 1. Architecture

### 1.1 Plugin Structure

The system is implemented as a suite of Moodle plugins plus an optional external service for LLM operations and the proficiency dashboard.

#### Moodle Plugins

**`local_ocms`** — Core content management and mastery tracking engine
```
/local/ocms/
├── classes/
│   ├── content_processor.php       # Content analysis and processing
│   ├── claude_api.php              # Claude API integration
│   ├── tracking_manager.php        # Score and interaction tracking
│   ├── tag_scorer.php              # Tag-based competency scoring
│   ├── content_selector.php        # Adaptive content selection engine
│   ├── mastery_calculator.php      # CSF mastery score computation
│   ├── csf_manager.php             # NIST CSF taxonomy management
│   └── external/                   # Web services
├── db/
│   ├── install.xml                 # Table definitions
│   ├── upgrade.php                 # Schema migrations
│   └── services.php                # Web service definitions
├── lang/en/
│   └── local_ocms.php
├── lib.php
├── settings.php
└── version.php
```

**`mod_aatcontent`** — Activity module for AAT content delivery within courses
```
/mod/aatcontent/
├── classes/
│   ├── external.php                # Activity web services
│   └── event/                      # Activity events
├── backup/moodle2/                 # Backup/restore support
├── db/
│   ├── install.xml
│   └── access.php                  # Capabilities
├── lang/en/
│   └── mod_aatcontent.php
├── index.php
├── view.php                        # Content launcher
├── lib.php
├── mod_form.php
└── version.php
```

**`auth_aattoken`** — Passwordless authentication for external learners
```
/auth/aattoken/
├── auth.php
├── classes/
│   └── token_manager.php
├── db/
│   └── install.xml
├── lang/en/
│   └── auth_aattoken.php
└── version.php
```

**`tool_aat`** — Campaign/path management, dashboard, and admin tools
```
/tool/aat/
├── classes/
│   ├── campaign_manager.php        # Campaign CRUD and lifecycle
│   ├── path_manager.php            # Proficiency path management
│   ├── email_manager.php           # Email template rendering and sending
│   ├── simulation_manager.php      # Informative email simulation management
│   ├── branching_engine.php        # Path branching logic
│   ├── llm_goal_interpreter.php    # Natural language goal → tag mapping
│   └── llm_path_generator.php     # LLM-powered proficiency path generation
├── db/
│   └── install.xml
├── lang/en/
│   └── tool_aat.php
├── index.php                       # Campaign management UI
├── campaign.php                    # Campaign configuration
├── paths.php                       # Proficiency path browser
├── dashboard.php                   # Proficiency dashboard (radar chart)
├── simulations.php                 # Email simulation management
└── version.php
```

#### External Service (Optional)

For deployments wanting a richer dashboard UI or to offload LLM calls:

- **Purpose:** LLM path generation, proficiency dashboard frontend (modern JS framework), batch analytics
- **Stack:** Node.js or Python service + React/Vue frontend
- **Communication:** REST API to/from `local_ocms` web services
- **When to use:** When the Moodle PHP stack constrains dashboard interactivity or LLM integration complexity

The external service is optional — all core functionality works within the Moodle plugin suite alone. The external service adds a richer dashboard experience and offloads computationally intensive LLM orchestration.

### 1.2 Architecture Decision Record

| Approach | Pros | Cons | When to Use |
|---|---|---|---|
| **Moodle plugins only** | Single deployment, native integration, simpler ops | PHP constraints on dashboard UX, synchronous LLM calls | Smaller deployments, simpler requirements |
| **Hybrid (plugins + external service)** | Best dashboard UX, async LLM, scalable | More infra, data sync between systems | Larger deployments, dashboard-heavy use cases |

**Default recommendation:** Start with Moodle plugins only. Add the external service when/if dashboard requirements or LLM throughput demand it.

---

## 2. Taxonomy: Dual-Layer Tagging System

The system uses two complementary tagging layers:

### 2.1 Layer 1: NIST CSF 2.0 Functions (Strategic)

Six high-level functions from the NIST Cybersecurity Framework 2.0. These are the axes of the proficiency radar chart and the primary lens for mastery tracking.

| Code | Function | Training Focus |
|------|----------|---------------|
| GV | Govern | Policy awareness, risk culture, roles & responsibilities |
| ID | Identify | Asset awareness, risk assessment, threat landscape understanding |
| PR | Protect | Safeguards, access control, security hygiene, awareness & training |
| DE | Detect | Recognizing anomalies, spotting suspicious activity, monitoring |
| RS | Respond | Incident handling, reporting procedures, containment |
| RC | Recover | Business continuity, restoration, communication during recovery |

Each piece of content is tagged with one or more CSF Functions plus a **relevance score** (0.0–1.0) per function. This allows a single course to contribute mastery points across multiple functions, weighted by relevance.

**Optional extension:** Tag at the CSF Category level (22 categories) for finer-grained path generation, rolling up to Functions for the radar chart.

### 2.2 Layer 2: Topic Tags (Tactical)

Granular security topic tags for content selection, remediation routing, and compliance mapping. These are the tags the adaptive content selection engine uses to find specific content for specific skill gaps.

#### Initial Tag Library

| Tag | Category | CSF Function(s) |
|-----|----------|-----------------|
| phishing | Security | DE, PR |
| social_engineering | Security | DE, PR |
| malware | Security | DE, PR, RS |
| ransomware | Security | DE, RS, RC |
| password_security | Security | PR |
| mfa | Security | PR |
| physical_security | Security | PR |
| data_protection | Compliance | PR, GV |
| pii | Compliance | PR, GV |
| hipaa | Compliance | GV, PR |
| pci_dss | Compliance | GV, PR |
| gdpr | Compliance | GV, PR |
| incident_response | Skills | RS, RC |
| safe_browsing | Skills | PR, DE |
| mobile_security | Skills | PR |
| remote_work | Skills | PR |
| cloud_security | Skills | PR, ID |
| email_security | Skills | PR, DE |
| insider_threat | Security | DE, RS |
| business_email_compromise | Security | DE, RS |
| vishing | Security | DE |
| smishing | Security | DE |
| asset_management | Skills | ID |
| risk_assessment | Skills | ID, GV |
| backup_recovery | Skills | RC, PR |

Tags map to CSF Functions (many-to-many). When a learner improves on a topic tag, the mastery score improvement propagates to the associated CSF Functions weighted by relevance.

### 2.3 How the Two Layers Interact

```
Learner completes "Phishing Awareness 101"
    ↓
Content has topic tags: phishing (0.95), social_engineering (0.60), email_security (0.40)
Content has CSF tags:   DE (0.85), PR (0.55), RS (0.20)
Content difficulty: 2/5, mastery points: 10
Learner scored: 88%
    ↓
Topic tag scores updated:
  - phishing:            score weighted by 0.95 confidence × 0.88 grade
  - social_engineering:  score weighted by 0.60 confidence × 0.88 grade
  - email_security:      score weighted by 0.40 confidence × 0.88 grade
    ↓
CSF mastery scores updated:
  - Detect:  10 points × 0.85 relevance × 0.88 grade = 7.48 mastery points
  - Protect: 10 points × 0.55 relevance × 0.88 grade = 4.84 mastery points
  - Respond: 10 points × 0.20 relevance × 0.88 grade = 1.76 mastery points
    ↓
Competency level recalculated per tag
Trend direction recalculated per tag
Radar chart updated for this learner
Branching engine evaluates next step in proficiency path
```

---

## 3. Database Schema

### 3.1 NIST CSF Reference Tables

```sql
-- NIST CSF 2.0 Functions (seeded, not user-editable)
CREATE TABLE {local_ocms_csf_functions} (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    code VARCHAR(2) NOT NULL UNIQUE,              -- GV, ID, PR, DE, RS, RC
    name VARCHAR(50) NOT NULL,                    -- Govern, Identify, etc.
    description TEXT,
    sort_order INT DEFAULT 0
);

-- NIST CSF 2.0 Categories (optional, for drill-down)
CREATE TABLE {local_ocms_csf_categories} (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    function_id BIGINT NOT NULL,
    code VARCHAR(10) NOT NULL UNIQUE,             -- GV.OC, PR.AT, DE.CM, etc.
    name VARCHAR(100) NOT NULL,
    description TEXT,
    sort_order INT DEFAULT 0,
    FOREIGN KEY (function_id) REFERENCES {local_ocms_csf_functions}(id)
);
```

### 3.2 Content Tables

```sql
-- Content library (OCMS content items — may be Moodle courses or standalone content)
CREATE TABLE {local_ocms_content} (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    moodle_course_id BIGINT,                      -- FK to mdl_course if linked to a Moodle course (NULL for standalone)
    title VARCHAR(255) NOT NULL,
    description TEXT,
    content_type ENUM('course', 'scorm', 'html', 'video', 'interactive', 'assessment', 'email_simulation') NOT NULL,
    content_path VARCHAR(500),                    -- Moodle file area reference
    external_url VARCHAR(500),                    -- For externally hosted content
    difficulty TINYINT DEFAULT 3,                 -- 1-5 scale
    mastery_points INT DEFAULT 10,                -- Points earned on successful completion
    duration_minutes INT,
    scorable TINYINT(1) DEFAULT 0,
    passing_score INT DEFAULT 80,
    thumbnail_path VARCHAR(500),
    ai_analysis_json LONGTEXT,                    -- Claude analysis cache
    source_type VARCHAR(50),                      -- 'moodle_course', 'upload', 'import', 'external'
    active TINYINT(1) DEFAULT 1,
    timecreated BIGINT NOT NULL,
    timemodified BIGINT NOT NULL,
    usermodified BIGINT NOT NULL,
    INDEX idx_moodle_course (moodle_course_id),
    INDEX idx_content_type (content_type)
);

-- Content → CSF Function mapping (Layer 1 tags)
CREATE TABLE {local_ocms_content_csf_tags} (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    contentid BIGINT NOT NULL,
    csf_function_id BIGINT NOT NULL,
    relevance_score DECIMAL(3,2) NOT NULL,        -- 0.00-1.00
    manually_assigned TINYINT(1) DEFAULT 0,       -- 1 = expert-assigned, 0 = AI-suggested
    timecreated BIGINT NOT NULL,
    FOREIGN KEY (contentid) REFERENCES {local_ocms_content}(id) ON DELETE CASCADE,
    FOREIGN KEY (csf_function_id) REFERENCES {local_ocms_csf_functions}(id),
    UNIQUE KEY idx_content_csf (contentid, csf_function_id)
);

-- Content → Topic tag mapping (Layer 2 tags)
CREATE TABLE {local_ocms_content_tags} (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    contentid BIGINT NOT NULL,
    tag_name VARCHAR(100) NOT NULL,
    tag_type ENUM('topic', 'skill', 'compliance', 'custom') DEFAULT 'topic',
    confidence_score DECIMAL(3,2),                -- 0.00-1.00 (from Claude or manual)
    manually_assigned TINYINT(1) DEFAULT 0,
    timecreated BIGINT NOT NULL,
    FOREIGN KEY (contentid) REFERENCES {local_ocms_content}(id) ON DELETE CASCADE,
    INDEX idx_tag_name (tag_name),
    INDEX idx_content_tag (contentid, tag_name)
);

-- Topic tag → CSF Function mapping (how topic tags roll up to CSF)
CREATE TABLE {local_ocms_tag_csf_mapping} (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    tag_name VARCHAR(100) NOT NULL,
    csf_function_id BIGINT NOT NULL,
    weight DECIMAL(3,2) DEFAULT 1.00,             -- How strongly this tag relates to this function
    FOREIGN KEY (csf_function_id) REFERENCES {local_ocms_csf_functions}(id),
    UNIQUE KEY idx_tag_csf (tag_name, csf_function_id)
);

-- Predefined tag library
CREATE TABLE {local_ocms_tag_library} (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    tag_name VARCHAR(100) NOT NULL UNIQUE,
    tag_category VARCHAR(50),                     -- 'security', 'compliance', 'skills'
    description TEXT,
    active TINYINT(1) DEFAULT 1,
    sort_order INT DEFAULT 0
);

-- Content prerequisites (DAG)
CREATE TABLE {local_ocms_content_prerequisites} (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    contentid BIGINT NOT NULL,
    prerequisite_contentid BIGINT NOT NULL,
    FOREIGN KEY (contentid) REFERENCES {local_ocms_content}(id) ON DELETE CASCADE,
    FOREIGN KEY (prerequisite_contentid) REFERENCES {local_ocms_content}(id) ON DELETE CASCADE,
    UNIQUE KEY idx_prereq (contentid, prerequisite_contentid)
);
```

### 3.3 Campaign & Proficiency Path Tables

```sql
-- Campaigns (operator-created training programs)
CREATE TABLE {tool_aat_campaigns} (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    courseid BIGINT,                              -- Optional Moodle course container
    cohortid BIGINT,                              -- Target cohort (NULL = manual enrollment)
    status ENUM('draft', 'active', 'paused', 'completed', 'archived') DEFAULT 'draft',

    -- Goal Configuration
    goal_description TEXT,                        -- Natural language goals from operator
    goal_tags_json LONGTEXT,                      -- LLM-parsed tag priorities
    goal_csf_focus_json LONGTEXT,                 -- CSF function focus + weights (e.g., [{"function":"DE","weight":0.8},{"function":"PR","weight":0.5}])
    goal_competency_targets_json LONGTEXT,        -- Target scores per tag/CSF function

    -- Content Selection Settings
    selection_mode ENUM('ai_adaptive', 'structured_path', 'tag_weighted', 'sequential') DEFAULT 'structured_path',
    proficiency_path_id BIGINT,                   -- FK to proficiency_paths if using structured_path mode
    min_content_per_learner INT DEFAULT 1,
    max_content_per_learner INT DEFAULT 10,
    content_pool_tags_json LONGTEXT,
    exclude_completed TINYINT(1) DEFAULT 1,

    -- Scheduling
    start_date BIGINT,
    end_date BIGINT,
    delivery_frequency ENUM('immediate', 'daily', 'weekly', 'biweekly', 'monthly') DEFAULT 'weekly',
    delivery_day_of_week TINYINT,
    delivery_time TIME,

    -- Email Configuration
    assignment_email_enabled TINYINT(1) DEFAULT 1,
    assignment_email_subject VARCHAR(255),
    assignment_email_body LONGTEXT,
    reminder_email_enabled TINYINT(1) DEFAULT 1,
    reminder_email_subject VARCHAR(255),
    reminder_email_body LONGTEXT,
    reminder_frequency_days INT DEFAULT 3,
    max_reminders INT DEFAULT 3,

    -- Manager Reporting
    manager_report_enabled TINYINT(1) DEFAULT 0,
    manager_report_frequency ENUM('daily', 'weekly', 'monthly') DEFAULT 'weekly',
    manager_report_recipients TEXT,

    timecreated BIGINT NOT NULL,
    timemodified BIGINT NOT NULL,
    usermodified BIGINT NOT NULL,
    FOREIGN KEY (cohortid) REFERENCES {cohort}(id) ON DELETE SET NULL
);

-- Proficiency Paths (LLM-generated structured learning paths)
CREATE TABLE {tool_aat_proficiency_paths} (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    created_by BIGINT NOT NULL,                   -- Operator who generated/approved this path

    -- CSF Focus Configuration
    focus_functions_json LONGTEXT NOT NULL,        -- [{"csf_function_id":4,"code":"DE","weight":0.8}, ...]
    target_mastery_level ENUM('developing', 'proficient', 'expert') DEFAULT 'proficient',

    -- Generation Metadata
    generation_params_json LONGTEXT,              -- LLM prompt config, model used, temperature, etc.
    generation_rationale TEXT,                     -- LLM's explanation of why this path was structured this way

    status ENUM('draft', 'active', 'archived') DEFAULT 'draft',
    timecreated BIGINT NOT NULL,
    timemodified BIGINT NOT NULL,
    FOREIGN KEY (created_by) REFERENCES {user}(id)
);

-- Path Steps (ordered content within a proficiency path, with branching)
CREATE TABLE {tool_aat_path_steps} (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    path_id BIGINT NOT NULL,
    step_order INT NOT NULL,                      -- Position in the path
    contentid BIGINT NOT NULL,                    -- FK to local_ocms_content
    track ENUM('standard', 'challenge', 'remediation') DEFAULT 'standard',

    -- Branching Conditions (evaluated against learner's performance on the triggering step)
    -- For standard track: these define when to branch OUT to challenge or remediation
    -- For challenge/remediation track: branch_return_to_step defines where to rejoin standard
    branch_condition_json LONGTEXT,               -- e.g., {"min_grade": 90} for challenge, {"max_grade": 60} for remediation
    branch_return_to_step INT,                    -- Step order to return to after challenge/remediation

    -- Email simulation (if this step uses an informative sim instead of or in addition to a course)
    simulation_id BIGINT,                         -- FK to email_simulations, nullable

    -- Metadata
    description TEXT,                             -- Why this step is here (from LLM rationale or operator notes)

    FOREIGN KEY (path_id) REFERENCES {tool_aat_proficiency_paths}(id) ON DELETE CASCADE,
    FOREIGN KEY (contentid) REFERENCES {local_ocms_content}(id),
    INDEX idx_path_order (path_id, step_order, track)
);

-- Campaign Enrollments
CREATE TABLE {tool_aat_enrollments} (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    campaignid BIGINT NOT NULL,
    userid BIGINT NOT NULL,
    status ENUM('pending', 'active', 'completed', 'withdrawn') DEFAULT 'pending',
    enrolled_date BIGINT,
    completed_date BIGINT,
    progress_percentage DECIMAL(5,2) DEFAULT 0,

    -- Path Position (for structured_path mode)
    current_path_step_id BIGINT,                  -- FK to tool_aat_path_steps
    current_track ENUM('standard', 'challenge', 'remediation') DEFAULT 'standard',

    -- Adaptive State (for ai_adaptive mode)
    current_focus_tags_json LONGTEXT,
    competency_snapshot_json LONGTEXT,
    next_content_recommendation BIGINT,

    timecreated BIGINT NOT NULL,
    timemodified BIGINT NOT NULL,
    FOREIGN KEY (campaignid) REFERENCES {tool_aat_campaigns}(id) ON DELETE CASCADE,
    FOREIGN KEY (userid) REFERENCES {user}(id) ON DELETE CASCADE,
    UNIQUE KEY idx_campaign_user (campaignid, userid)
);

-- Content Assignments (individual content pieces assigned to learners)
CREATE TABLE {tool_aat_assignments} (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    enrollmentid BIGINT NOT NULL,
    contentid BIGINT NOT NULL,
    path_step_id BIGINT,                          -- FK to path_steps if part of a structured path

    -- Assignment State
    status ENUM('pending', 'notified', 'in_progress', 'completed', 'expired') DEFAULT 'pending',
    assigned_date BIGINT NOT NULL,
    due_date BIGINT,
    started_date BIGINT,
    completed_date BIGINT,

    -- Selection Reason
    selection_reason ENUM('path_standard', 'path_challenge', 'path_remediation', 'ai_adaptive', 'goal_alignment', 'manual') DEFAULT 'path_standard',
    selection_context_json LONGTEXT,              -- Why this content was selected

    -- Scoring
    score DECIMAL(5,2),
    passed TINYINT(1),
    time_spent_seconds INT,
    attempts INT DEFAULT 0,

    -- Notification Tracking
    assignment_email_sent BIGINT,
    reminder_count INT DEFAULT 0,
    last_reminder_sent BIGINT,

    -- Access Token (for passwordless auth)
    access_token VARCHAR(64) UNIQUE,
    token_expires BIGINT,

    timecreated BIGINT NOT NULL,
    timemodified BIGINT NOT NULL,
    FOREIGN KEY (enrollmentid) REFERENCES {tool_aat_enrollments}(id) ON DELETE CASCADE,
    FOREIGN KEY (contentid) REFERENCES {local_ocms_content}(id) ON DELETE CASCADE,
    INDEX idx_token (access_token),
    INDEX idx_status (status)
);
```

### 3.4 Mastery & Competency Tracking Tables

```sql
-- CSF Function Mastery Scores (Layer 1 — radar chart axes)
CREATE TABLE {local_ocms_user_csf_mastery} (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    userid BIGINT NOT NULL,
    csf_function_id BIGINT NOT NULL,
    mastery_score DECIMAL(10,2) DEFAULT 0,        -- Accumulated mastery points
    mastery_percentage DECIMAL(5,2) DEFAULT 0,    -- Score as % of full proficiency threshold
    last_updated BIGINT NOT NULL,
    FOREIGN KEY (userid) REFERENCES {user}(id) ON DELETE CASCADE,
    FOREIGN KEY (csf_function_id) REFERENCES {local_ocms_csf_functions}(id),
    UNIQUE KEY idx_user_csf (userid, csf_function_id)
);

-- Topic Tag Competency Scores (Layer 2 — granular skills)
CREATE TABLE {local_ocms_user_tag_scores} (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    userid BIGINT NOT NULL,
    tag_name VARCHAR(100) NOT NULL,

    -- Cumulative Scoring
    total_score_points DECIMAL(10,2) DEFAULT 0,
    total_attempts INT DEFAULT 0,
    passed_attempts INT DEFAULT 0,

    -- Derived Metrics
    average_score DECIMAL(5,2),
    pass_rate DECIMAL(5,2),
    competency_level ENUM('novice', 'developing', 'proficient', 'expert'),

    -- Trend Data
    recent_scores_json LONGTEXT,                  -- Last 10 scores with timestamps
    trend_direction ENUM('improving', 'stable', 'declining'),

    timecreated BIGINT NOT NULL,
    timemodified BIGINT NOT NULL,
    FOREIGN KEY (userid) REFERENCES {user}(id) ON DELETE CASCADE,
    UNIQUE KEY idx_user_tag (userid, tag_name),
    INDEX idx_tag_competency (tag_name, competency_level)
);

-- Mastery Events (audit log — every mastery-affecting event)
CREATE TABLE {local_ocms_mastery_events} (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    userid BIGINT NOT NULL,
    contentid BIGINT NOT NULL,
    assignmentid BIGINT,

    -- What changed
    event_type ENUM('completion', 'assessment', 'simulation_response', 'decay') NOT NULL,
    csf_function_id BIGINT,                       -- Which CSF function was affected
    tag_name VARCHAR(100),                        -- Which topic tag was affected
    points_earned DECIMAL(10,2),
    course_grade DECIMAL(5,2),

    timestamp BIGINT NOT NULL,
    FOREIGN KEY (userid) REFERENCES {user}(id) ON DELETE CASCADE,
    INDEX idx_user_time (userid, timestamp),
    INDEX idx_csf (csf_function_id)
);

-- Cohort Mastery Cache (pre-computed for dashboard performance)
CREATE TABLE {local_ocms_cohort_mastery_cache} (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    cohortid BIGINT NOT NULL,
    csf_function_id BIGINT NOT NULL,
    avg_mastery_percentage DECIMAL(5,2),
    min_percentage DECIMAL(5,2),
    max_percentage DECIMAL(5,2),
    member_count INT,
    computed_at BIGINT NOT NULL,
    FOREIGN KEY (cohortid) REFERENCES {cohort}(id) ON DELETE CASCADE,
    FOREIGN KEY (csf_function_id) REFERENCES {local_ocms_csf_functions}(id),
    UNIQUE KEY idx_cohort_csf (cohortid, csf_function_id)
);

-- Score Records (detailed per-attempt scoring)
CREATE TABLE {local_ocms_score_records} (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    assignmentid BIGINT NOT NULL,
    userid BIGINT NOT NULL,
    contentid BIGINT NOT NULL,
    score DECIMAL(5,2) NOT NULL,
    max_score DECIMAL(5,2) DEFAULT 100,
    passed TINYINT(1) NOT NULL,
    passing_score DECIMAL(5,2),
    scorm_data_json LONGTEXT,
    time_spent_seconds INT,
    completed_at BIGINT NOT NULL,
    timecreated BIGINT NOT NULL,
    FOREIGN KEY (assignmentid) REFERENCES {tool_aat_assignments}(id) ON DELETE CASCADE,
    FOREIGN KEY (userid) REFERENCES {user}(id) ON DELETE CASCADE,
    FOREIGN KEY (contentid) REFERENCES {local_ocms_content}(id) ON DELETE CASCADE,
    INDEX idx_user_content (userid, contentid)
);

-- Content Interaction Tracking (granular — clicks, views, etc.)
CREATE TABLE {local_ocms_interactions} (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    assignmentid BIGINT NOT NULL,
    userid BIGINT NOT NULL,
    interaction_type ENUM('view', 'click', 'submit', 'focus', 'complete', 'score') NOT NULL,
    tag_name VARCHAR(100),
    element_selector VARCHAR(255),
    interaction_data_json LONGTEXT,
    success TINYINT(1),
    timestamp BIGINT NOT NULL,
    duration_ms INT,
    FOREIGN KEY (assignmentid) REFERENCES {tool_aat_assignments}(id) ON DELETE CASCADE,
    FOREIGN KEY (userid) REFERENCES {user}(id) ON DELETE CASCADE,
    INDEX idx_assignment (assignmentid),
    INDEX idx_user_time (userid, timestamp)
);
```

### 3.5 Email Simulation Tables

```sql
-- Informative Email Simulations
CREATE TABLE {tool_aat_email_simulations} (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    threat_type VARCHAR(100) NOT NULL,            -- phishing, bec, vishing, etc.
    difficulty TINYINT DEFAULT 3,                 -- 1-5
    mastery_points INT DEFAULT 5,

    -- Email Content
    email_subject VARCHAR(255) NOT NULL,
    email_sender_name VARCHAR(255),
    email_sender_address VARCHAR(255),
    email_body_html LONGTEXT NOT NULL,

    -- Educational Annotations (the "here's how to spot this" content)
    annotations_json LONGTEXT,                    -- Array of {selector, annotation_text, indicator_type} for highlighting red flags
    education_page_html LONGTEXT,                 -- Full education page shown after annotation review

    -- Linking to next training
    next_contentid BIGINT,                        -- Content to enroll in after viewing annotations

    active TINYINT(1) DEFAULT 1,
    timecreated BIGINT NOT NULL,
    timemodified BIGINT NOT NULL,
    usermodified BIGINT NOT NULL,
    FOREIGN KEY (next_contentid) REFERENCES {local_ocms_content}(id) ON DELETE SET NULL
);

-- Simulation → CSF Function mapping
CREATE TABLE {tool_aat_simulation_csf_tags} (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    simulation_id BIGINT NOT NULL,
    csf_function_id BIGINT NOT NULL,
    relevance_score DECIMAL(3,2) NOT NULL,
    FOREIGN KEY (simulation_id) REFERENCES {tool_aat_email_simulations}(id) ON DELETE CASCADE,
    FOREIGN KEY (csf_function_id) REFERENCES {local_ocms_csf_functions}(id),
    UNIQUE KEY idx_sim_csf (simulation_id, csf_function_id)
);

-- Simulation → Topic tag mapping
CREATE TABLE {tool_aat_simulation_tags} (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    simulation_id BIGINT NOT NULL,
    tag_name VARCHAR(100) NOT NULL,
    confidence_score DECIMAL(3,2),
    FOREIGN KEY (simulation_id) REFERENCES {tool_aat_email_simulations}(id) ON DELETE CASCADE,
    INDEX idx_sim_tag (simulation_id, tag_name)
);

-- Learner interactions with simulations
CREATE TABLE {tool_aat_simulation_responses} (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    simulation_id BIGINT NOT NULL,
    userid BIGINT NOT NULL,
    assignmentid BIGINT,                          -- FK to assignment if delivered as part of a path

    -- Tracking
    delivered_at BIGINT NOT NULL,
    opened_at BIGINT,
    link_clicked_at BIGINT,
    annotations_viewed_at BIGINT,
    education_page_viewed_at BIGINT,
    next_training_clicked_at BIGINT,

    -- Response data for any interactive elements
    response_data_json LONGTEXT,

    -- Derived: did they engage with the educational content?
    fully_engaged TINYINT(1) DEFAULT 0,           -- Viewed annotations + education page

    timecreated BIGINT NOT NULL,
    FOREIGN KEY (simulation_id) REFERENCES {tool_aat_email_simulations}(id) ON DELETE CASCADE,
    FOREIGN KEY (userid) REFERENCES {user}(id) ON DELETE CASCADE,
    INDEX idx_user_sim (userid, simulation_id)
);
```

### 3.6 Email & Notification Tables

```sql
-- Email Templates
CREATE TABLE {tool_aat_email_templates} (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    template_type ENUM('assignment', 'reminder', 'completion', 'milestone', 'manager_report', 'simulation_followup', 'custom') NOT NULL,
    subject VARCHAR(255) NOT NULL,
    body_html LONGTEXT NOT NULL,
    body_text LONGTEXT,
    is_default TINYINT(1) DEFAULT 0,
    active TINYINT(1) DEFAULT 1,
    timecreated BIGINT NOT NULL,
    timemodified BIGINT NOT NULL,
    usermodified BIGINT NOT NULL
);

-- Email Send Log
CREATE TABLE {tool_aat_email_log} (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    assignmentid BIGINT,
    campaignid BIGINT,
    userid BIGINT NOT NULL,
    email_type ENUM('assignment', 'reminder', 'completion', 'milestone', 'manager_report', 'simulation') NOT NULL,
    recipient_email VARCHAR(255) NOT NULL,
    subject VARCHAR(255),
    status ENUM('queued', 'sent', 'failed', 'bounced') DEFAULT 'queued',
    sent_at BIGINT,
    error_message TEXT,
    timecreated BIGINT NOT NULL,
    FOREIGN KEY (userid) REFERENCES {user}(id) ON DELETE CASCADE,
    INDEX idx_status (status),
    INDEX idx_campaign (campaignid)
);
```

---

## 4. Content Selection Modes

The system supports three content selection modes, configurable per campaign:

### 4.1 Structured Path Mode (Default — New)

Pre-generated proficiency paths with defined branch points. The LLM generates the complete path upfront; the operator reviews and approves it before any learner is enrolled.

**How it works:**
1. Operator selects CSF function focus areas and weights
2. LLM generates a complete path with standard/challenge/remediation tracks
3. Operator reviews, edits, approves
4. Learners follow the path; branching happens at predefined decision points based on performance

**Branch evaluation:**
```
On step completion:
  if score >= branch_condition.challenge_threshold (default 90%):
      → route to challenge track step
  elif score <= branch_condition.remediation_threshold (default 60%):
      → route to remediation track step
  else:
      → continue on standard track

After challenge/remediation step:
  → return to standard track at branch_return_to_step
```

**Advantages:** Predictable, auditable, operator can see exactly what learners will experience. Good for compliance-sensitive environments.

### 4.2 AI Adaptive Mode (From OCMS Spec)

Fully dynamic per-learner content selection. The AI picks the next content item for each learner each cycle based on their current competency state.

**How it works:**
```
function selectNextContent(enrollment):
    user = enrollment.user
    campaign = enrollment.campaign
    tag_scores = getUserTagScores(user)
    completed_content = getCompletedContent(enrollment)

    weak_tags = findWeakTags(tag_scores, campaign.goal_tags)
    strong_tags = findStrongTags(tag_scores, campaign.goal_tags)

    if weak_tags.hasRemediationNeeded():
        candidates = getContentByTags(weak_tags, difficulty='easier')
    elif not campaign.competency_targets_met(tag_scores):
        candidates = getContentByTags(campaign.primary_tags, difficulty='appropriate')
    else:
        candidates = getContentByTags(strong_tags, difficulty='harder')

    candidates = candidates.exclude(completed_content)
    return llmSelectBest(candidates, user, campaign)
```

**Advantages:** Most personalized. Every learner gets a unique journey. Best for large content libraries with many options per topic.

### 4.3 Tag-Weighted Mode

Distributes content based on tag priority weights without AI involvement. Deterministic, even distribution.

### 4.4 Sequential Mode

Delivers specific content in a defined order. Traditional LMS behavior. No adaptation.

### 4.5 Content Pool Validation

Regardless of mode, the system validates that the content pool is sufficient:
- **Warning** if available content < 2x max_content_per_learner
- **Error** if available content < min_content_per_learner
- **Warning** if any CSF function in the focus area has < 3 tagged content items

---

## 5. Competency Model

### 5.1 Competency Levels (Per Topic Tag)

| Level | Criteria |
|-------|----------|
| **Novice** | < 3 attempts OR average score < 50% |
| **Developing** | 3+ attempts AND average score 50–69% |
| **Proficient** | 5+ attempts AND average score 70–89% |
| **Expert** | 10+ attempts AND average score 90%+ AND pass rate 95%+ |

### 5.2 CSF Mastery Score (Per Function)

Accumulated mastery points per CSF function, expressed as a percentage of a configurable "full proficiency" threshold.

**Calculation on content completion:**
```
For each CSF function tagged on the content:
    points = content.mastery_points × csf_tag.relevance_score × (learner_grade / 100)
    user_csf_mastery.mastery_score += points
    user_csf_mastery.mastery_percentage = mastery_score / full_proficiency_threshold × 100
```

**Full proficiency threshold:** Configurable per CSF function per campaign (or system-wide default). Represents the mastery score at which a learner is considered fully proficient in that area.

### 5.3 Trend Tracking

- Compare last 5 scores to previous 5 scores on the same tag
- **Improving:** Recent average > Previous average + 5%
- **Declining:** Recent average < Previous average - 5%
- **Stable:** Within 5%

### 5.4 Mastery Decay (Optional)

Configurable per campaign. If enabled:
- Mastery scores degrade by a configurable percentage per period (e.g., 5% per month)
- Decay is paused while a learner is actively enrolled in a campaign covering that function
- Decay drives re-engagement by making mastery something that must be maintained
- Implemented as a scheduled task that processes all active learners

---

## 6. LLM Integration

### 6.1 Configuration

| Setting | Type | Description |
|---------|------|-------------|
| `claude_api_key` | Password | API key for Claude |
| `claude_model` | Select | Model to use (claude-sonnet-4-6, claude-opus-4-6, etc.) |
| `claude_max_tokens` | Integer | Max tokens for API responses |
| `enable_ai_content_selection` | Checkbox | Enable/disable AI-driven selection |
| `max_content_analysis_size` | Integer | Max content size for AI analysis (bytes) |

### 6.2 LLM Use Cases

**A. Content Analysis and Auto-Tagging**
- Analyze uploaded content to suggest topic tags and CSF function mappings
- Store results in `ai_analysis_json` cache on content record
- Expert reviews and confirms/adjusts suggestions

**B. Goal Interpretation (Campaign Setup)**
- Accept natural language training objectives from operator
- Return structured interpretation: priority tags, CSF focus areas, competency targets, recommended strategy

Example input:
```
"We want to improve our team's ability to detect and respond to phishing
and business email compromise attacks. We also need HIPAA compliance coverage."
```

Example output:
```json
{
    "csf_focus": [
        {"function": "DE", "weight": 0.9, "rationale": "Primary goal is detection of email threats"},
        {"function": "RS", "weight": 0.7, "rationale": "Response to detected threats is secondary goal"},
        {"function": "GV", "weight": 0.3, "rationale": "HIPAA compliance has governance component"},
        {"function": "PR", "weight": 0.3, "rationale": "Protective measures against email threats"}
    ],
    "primary_tags": [
        {"tag": "phishing", "priority": 1, "target_competency": "proficient"},
        {"tag": "business_email_compromise", "priority": 2, "target_competency": "proficient"}
    ],
    "secondary_tags": [
        {"tag": "hipaa", "priority": 3, "target_competency": "developing"},
        {"tag": "incident_response", "priority": 4, "target_competency": "developing"}
    ],
    "recommended_mode": "structured_path",
    "estimated_content_per_learner": 6,
    "estimated_duration_weeks": 8
}
```

**C. Proficiency Path Generation**
- Input: Tagged content catalog, CSF focus areas, constraints
- Output: Ordered path steps with standard/challenge/remediation tracks, simulation placements, rationale
- Operator reviews, edits, approves before activation

**D. Adaptive Content Selection (AI Adaptive Mode)**
- Per-learner, per-cycle: given learner state and candidate content, pick the best next item
- Used only in AI Adaptive mode, not Structured Path mode

### 6.3 Prompt Injection Prevention

- All LLM inputs are sanitized and validated
- Content analysis uses structured extraction prompts, not freeform
- LLM outputs are parsed as structured JSON; unexpected formats are rejected
- Operator-supplied goal descriptions are treated as untrusted input in prompt construction

---

## 7. Informative Email Simulations

### 7.1 Concept

Informative email simulations are **teaching tools**, not "gotcha" phishing tests. They present realistic threat-based emails to learners with built-in educational annotations that highlight detection indicators.

**Flow:**
1. Learner receives a realistic-looking threat email (as part of their proficiency path)
2. Email contains subtle indicators of compromise (spoofed sender, suspicious links, urgency tactics, etc.)
3. Learner clicks through to an **annotation page** that reveals and explains each red flag
4. Annotation page links to a deeper **education page** with context about this threat type
5. Education page links to the **next training course** in their path
6. Completion of the full flow (annotations viewed + education page) counts as a mastery event

### 7.2 Key Distinction from Phishing Simulator

| Aspect | Standard Phishing Simulator | Informative Email Simulation |
|--------|---------------------------|------------------------------|
| Purpose | Test/catch users | Teach users |
| Framing | Deceptive (user doesn't know it's a test) | Transparent (part of their learning path) |
| Failure mode | "You clicked a bad link" shame page | N/A — all paths lead to education |
| Tracking | Click/no-click binary | Engagement depth (opened → annotations → education → next training) |
| Integration | Standalone product | Embedded in proficiency path as a step |
| Scoring | Pass/fail | Engagement-based mastery points |

### 7.3 Delivery

- Delivered via the same email infrastructure as assignment notifications
- Tracked via unique tokens per learner per simulation (reuse `access_token` pattern)
- Can replace a standard assignment notification email at configured path steps
- Simulation emails are visually distinct from system emails once the learner reaches the annotation page (clear "This was a training exercise" framing)

---

## 8. Proficiency Dashboard

### 8.1 Radar Chart (Primary Visualization)

- **6 axes** corresponding to NIST CSF 2.0 Functions (GV, ID, PR, DE, RS, RC)
- **Scale:** 0–100% of full proficiency threshold per axis
- **Default view:** Single shaded polygon showing **cohort average**
- **Individual overlays:** Toggle specific learners on/off to compare against cohort average
  - All individuals off by default (privacy)
  - Manager/operator can toggle specific members
- **Cohort comparison:** Overlay multiple cohort polygons (e.g., Engineering vs. Finance)
- **Drill-down:** Click an axis to see breakdown by CSF Category within that Function
- **Export:** PDF/PNG for reporting, inline SVG for manager digest emails

**Technology:** Chart.js (built-in radar type, lightweight) or Apache ECharts (richer interactivity). D3.js if maximum customization is needed.

### 8.2 Learner Self-View Dashboard

| Widget | Description |
|--------|-------------|
| My Assignments | Pending/in-progress content with launch buttons, due dates |
| My Completed Training | History with scores, dates, time spent |
| My Skills Radar | Personal radar chart with comparison to cohort average |
| My Trend | Improving/stable/declining indicators per CSF function |
| Achievements | Milestones reached, streaks (optional) |

### 8.3 Operator/Manager Dashboard

| Widget | Description |
|--------|-------------|
| Campaign Summary | Active campaigns with progress bars and completion rates |
| Cohort Radar Chart | Aggregate proficiency with individual overlays |
| Skills Heatmap | Learners × Tags matrix, color-coded by competency level |
| Overdue Assignments | Past-due items requiring attention |
| Completion Timeline | Calendar view of completions |
| Stuck Learners | No progress in X days alert |
| Content Effectiveness | Which content items drive the most mastery improvement |

### 8.4 Skills Heatmap

Matrix view: rows = learners, columns = topic tags (or CSF functions). Cells color-coded by competency level:
- Red: Novice
- Yellow: Developing
- Green: Proficient
- Blue: Expert

Filterable by campaign, cohort, date range. Identifies at-a-glance who needs help where.

---

## 9. Passwordless Authentication

### 9.1 Purpose

Allow external learners (who may not have Moodle accounts) to access assigned training content via token links in emails. Primarily for MSP customers or organizations with external trainees.

### 9.2 Token Design

- 64-character hex token: `{assignment_id_encoded}_{random_bytes}_{checksum}`
- Cryptographically secure generation
- Configurable expiration (default: 30 days)
- Stored as hashed values (like passwords)
- Rate limited: max 10 validation attempts per IP per minute

### 9.3 Access Modes

1. **Token-Only:** User accesses only their assigned content via token URL
2. **Token-to-Session:** Token validates user and creates a limited Moodle session

### 9.4 User Account Handling

- **Existing Moodle users:** Token links to existing account via email match
- **New users:** Auto-create minimal account with `auth_aattoken` method; cannot log in via standard Moodle login
- **Account upgrade:** Optional path to "claim" account by setting a password

### 9.5 Access URL Format
```
https://{moodle_site}/auth/aattoken/access.php?token={access_token}
```

---

## 10. Campaign Management

### 10.1 Campaign Creation Wizard

**Step 1: Basics** — Name, description, dates

**Step 2: Audience** — Select cohort(s), preview member count, CSV upload option, exclusion list

**Step 3: Training Goals** — Natural language goal description → LLM interprets → suggests CSF focus, tags, competency targets, strategy. Operator accepts/modifies.

**Step 4: Content Mode Selection**
- Structured Path: Generate or select a proficiency path
- AI Adaptive: Configure content pool and adaptation parameters
- Tag-Weighted / Sequential: Traditional modes

**Step 5: Path Generation** (Structured Path mode only) — LLM generates path → operator reviews branching, simulation placements, rationale → approve

**Step 6: Delivery Schedule** — Frequency, preferred day/time, duration limits

**Step 7: Email Configuration** — Select/customize templates for assignments, reminders, milestone notifications. Preview with placeholders resolved.

**Step 8: Manager Reporting** — Enable/disable manager digest, frequency, recipients, include radar chart

**Step 9: Review & Launch** — Summary of all settings, dry-run preview, launch or save as draft

### 10.2 Campaign Lifecycle

```
Draft → Active → Paused → Active → Completed → Archived
                                 ↗
```

### 10.3 Cohort Sync

- Users added to cohort → auto-enrolled in active campaigns targeting that cohort
- Users removed from cohort → optionally withdrawn
- Sync: event-based (immediate) or scheduled

---

## 11. Messaging & Notifications

### 11.1 Email Types

| Type | Trigger | Content |
|------|---------|---------|
| Assignment | Content assigned to learner | Content title, access link, due date, path context |
| Reminder | Approaching/past due date | Days remaining, access link, reminder count |
| Completion | Learner completes content | Score, next step preview, encouragement |
| Milestone | Learner reaches mastery threshold | "You've reached 75% proficiency in Detect!" |
| Manager Digest | Scheduled (weekly/monthly) | Cohort radar chart, completion stats, stuck learners, mastery trends |
| Simulation | Path step triggers simulation | Realistic threat email (the simulation itself) |
| Simulation Follow-up | Learner completes simulation | Summary of what was learned, link to next training |

### 11.2 Mastery-Aware Context

All emails include path context when applicable:
- "Step 3 of 8 in your Detect & Respond path"
- "Your Protect proficiency: 62% → 71% after this training"
- Manager digests include inline radar chart images

### 11.3 Supported Placeholders

| Placeholder | Context | Description |
|-------------|---------|-------------|
| `{{LEARNER_FIRSTNAME}}` | All | User's first name |
| `{{LEARNER_LASTNAME}}` | All | User's last name |
| `{{LEARNER_FULLNAME}}` | All | User's full name |
| `{{LEARNER_EMAIL}}` | All | User's email address |
| `{{CONTENT_TITLE}}` | Assignment | Content piece title |
| `{{CONTENT_DESCRIPTION}}` | Assignment | Content description |
| `{{CONTENT_DURATION}}` | Assignment | Estimated duration |
| `{{ACCESS_LINK}}` | Assignment | Full URL with token |
| `{{ACCESS_BUTTON}}` | Assignment | HTML button element |
| `{{DUE_DATE}}` | Assignment | Due date formatted |
| `{{DAYS_REMAINING}}` | Assignment/Reminder | Days until due |
| `{{CAMPAIGN_NAME}}` | All | Campaign name |
| `{{COMPANY_NAME}}` | All | From site config |
| `{{CURRENT_YEAR}}` | All | Current year |
| `{{SUPPORT_EMAIL}}` | All | From AAT config |
| `{{REMINDER_COUNT}}` | Reminder | Which reminder (1st, 2nd) |
| `{{MAX_REMINDERS}}` | Reminder | Total reminders configured |
| `{{SCORE}}` | Completion | Achieved score |
| `{{PASSED_STATUS}}` | Completion | Passed/Failed |
| `{{PATH_NAME}}` | Path context | Proficiency path name |
| `{{PATH_STEP_CURRENT}}` | Path context | Current step number |
| `{{PATH_STEP_TOTAL}}` | Path context | Total steps in path |
| `{{CSF_FUNCTION_NAME}}` | Milestone | CSF function that hit milestone |
| `{{MASTERY_PERCENTAGE}}` | Milestone | Current mastery % for that function |
| `{{MASTERY_CHANGE}}` | Completion | Points gained this session |

---

## 12. Moodle Integration Points

### 12.1 Event System

| Event | Trigger | Data |
|-------|---------|------|
| `\tool_aat\event\campaign_created` | Campaign created | campaignid, userid |
| `\tool_aat\event\campaign_launched` | Campaign goes active | campaignid |
| `\tool_aat\event\user_enrolled` | User enrolled in campaign | campaignid, userid |
| `\tool_aat\event\content_assigned` | Content assigned to user | assignmentid, userid, contentid |
| `\tool_aat\event\content_viewed` | User starts content | assignmentid, userid |
| `\tool_aat\event\content_completed` | User completes content | assignmentid, userid, score |
| `\tool_aat\event\score_recorded` | Score submitted | assignmentid, userid, score |
| `\tool_aat\event\mastery_updated` | CSF mastery score changes | userid, csf_function, new_score |
| `\tool_aat\event\competency_updated` | Topic tag competency changes | userid, tag, new_level |
| `\tool_aat\event\path_branched` | Learner branches to challenge/remediation | enrollmentid, from_step, to_step, track |
| `\tool_aat\event\simulation_delivered` | Simulation email sent | simulationid, userid |
| `\tool_aat\event\simulation_engaged` | Learner completed simulation flow | simulationid, userid |
| `\tool_aat\event\campaign_completed` | User completes campaign | campaignid, userid |

### 12.2 Gradebook Integration

- AAT content scores sync to Moodle gradebook when content is delivered through a course context
- Grade item per AAT content piece in "AAT Training" grade category
- Immediate sync on score submission; batch sync option available
- Configurable per campaign (enable/disable gradebook sync)

### 12.3 Completion Tracking

- AAT content completion triggers Moodle activity completion
- Supports "require passing grade" and "require view" conditions
- Course completion can depend on AAT content completion

### 12.4 Moodle 5 Custom Reporting

**Report source:** "AAT Training Data" exposing:
- Entities: campaign, enrollment, assignment, score, tag_competency, csf_mastery
- Standard columns, filters, conditions for all entities

**Pre-built report templates:**
1. Campaign Completion Report
2. Skills Gap Analysis (competency level = Novice or Developing)
3. Overdue Training Report
4. Manager Summary Report
5. CSF Proficiency Report (mastery % per function per cohort)

### 12.5 Web Services API

**Content Management:**
- `local_ocms_upload_content`, `local_ocms_list_content`, `local_ocms_get_content`, `local_ocms_analyze_content`

**Campaign Management:**
- `tool_aat_create_campaign`, `tool_aat_update_campaign`, `tool_aat_get_campaign`, `tool_aat_list_campaigns`, `tool_aat_launch_campaign`, `tool_aat_pause_campaign`

**Path Management:**
- `tool_aat_generate_path`, `tool_aat_get_path`, `tool_aat_list_paths`, `tool_aat_activate_path`

**Enrollment:**
- `tool_aat_enroll_users`, `tool_aat_get_enrollments`, `tool_aat_get_user_progress`

**Mastery & Reporting:**
- `tool_aat_get_tag_scores`, `tool_aat_get_csf_mastery`, `tool_aat_get_campaign_stats`, `tool_aat_get_skills_matrix`, `tool_aat_get_cohort_radar_data`

### 12.6 Privacy API

- Export user data: enrollments, scores, interactions, mastery data, simulation responses
- Delete user data on request
- Configurable data retention policies

---

## 13. External System Integration

### 13.1 AWS SNS Notifications

Publish events to configured SNS topic:
```json
{
    "event_type": "mastery_updated",
    "timestamp": "2026-03-11T10:30:00Z",
    "moodle_site": "https://training.example.com",
    "data": {
        "user_id": 12345,
        "user_email": "john@example.com",
        "campaign_id": 100,
        "csf_function": "DE",
        "mastery_percentage": 72.5,
        "previous_percentage": 65.0,
        "tags": ["phishing", "social_engineering"]
    }
}
```

### 13.2 Future Enhancements

- LTI 1.3 Provider: Expose AAT content to external LMS consumers
- xAPI: Learning Record Store integration for enterprise reporting

---

## 14. Security Requirements

### 14.1 Capabilities

| Capability | Description | Default Roles |
|------------|-------------|---------------|
| `local/ocms:managecontent` | Upload, edit, delete content | Manager |
| `local/ocms:viewcontent` | View content library | Manager, Teacher |
| `local/ocms:managetaxonomy` | Manage CSF/tag mappings | Manager |
| `tool/aat:managecampaigns` | Create, edit campaigns | Manager |
| `tool/aat:managepaths` | Generate, edit proficiency paths | Manager |
| `tool/aat:managesimulations` | Create, edit email simulations | Manager |
| `tool/aat:viewcampaigns` | View campaign list and details | Manager, Teacher |
| `tool/aat:enrollusers` | Enroll users in campaigns | Manager |
| `tool/aat:viewreports` | View AAT reports and dashboards | Manager, Teacher |
| `tool/aat:viewcohortmastery` | View cohort-level radar chart | Manager, Teacher |
| `tool/aat:viewindividualmastery` | View individual learner mastery overlays | Manager |
| `tool/aat:viewownprogress` | View own progress and radar | Authenticated User |
| `mod/aatcontent:view` | Access assigned AAT content | Authenticated User |

### 14.2 Data Protection

- Encrypt stored API keys using Moodle's encryption
- Access tokens stored as hashed values
- Input validation: MIME checking, virus scanning, size limits, filename sanitization
- XSS sanitization on all user-generated content
- LLM prompt injection prevention (structured extraction, output validation)

---

## 15. Performance Requirements

### 15.1 Scalability Targets

- 100,000+ users per Moodle instance
- 1,000+ concurrent content sessions
- 10,000+ content library items
- 1,000+ active campaigns

### 15.2 Response Times

- Dashboard page load: < 2 seconds
- Radar chart render: < 1 second (using cached cohort data)
- Content launch: < 3 seconds
- API endpoints: < 500ms
- LLM path generation: < 30 seconds (async, operator waits with progress indicator)

### 15.3 Caching

- Content metadata: Moodle MUC cache
- Cohort mastery cache: Refreshed on mastery events, 30-second TTL for dashboard
- AI analysis results: Permanent until content modified
- Tag competency calculations: Recalculate on score entry, cache otherwise

### 15.4 Background Processing (Moodle Tasks)

**Ad-hoc tasks:** Email sending, content AI analysis, bulk enrollment, report generation, SNS publishing, path generation

**Scheduled tasks:**
- Daily: Assignment processing (new content selection for AI Adaptive mode), reminder email processing
- Hourly: Campaign status updates, cohort mastery cache refresh
- Configurable: Mastery decay processing, manager digest generation

---

## 16. Testing Requirements

### 16.1 Unit Tests (PHPUnit)

- Content selection algorithm (all modes)
- Mastery calculation (both layers)
- Competency level transitions
- Branching engine logic
- Token generation/validation
- Grade sync logic
- Mastery decay

### 16.2 Integration Tests (Behat)

- Campaign creation wizard (all steps)
- Proficiency path generation and approval flow
- Learner content access via token
- Path branching on completion
- Dashboard rendering with real data
- Simulation delivery and engagement flow
- Report generation

### 16.3 Performance Tests

- 500 concurrent content launches
- 10,000 simultaneous dashboard views
- Bulk enrollment of 50,000 users
- Radar chart rendering with 1,000-member cohort

---

## 17. Migration & Coexistence

### 17.1 Legacy Playbook Coexistence

- The existing playbook-based AAT continues to operate for MSP customers
- New mastery system is an **alternative mode**, not a forced replacement
- Sites can run both systems simultaneously
- Admin setting to choose default mode for new campaigns

### 17.2 Migration Path: Playbook → Proficiency Path

For sites wanting to transition:
1. Convert playbook course sequences into "flat" proficiency paths (sequential, no branching)
2. Retroactively tag playbook courses with CSF functions and topic tags
3. Import historical completion data into mastery tracking tables
4. Generate branching paths from the flat paths using LLM
5. Gradually transition cohorts from playbook mode to path mode

### 17.3 OCMS Content Migration

For sites with existing standalone OCMS instances:
- Import tool maps OCMS content IDs to Moodle content records
- Preserves tag assignments and AI analysis data
- Maps OCMS interaction history to new tracking tables

### 17.4 Installation

Standard Moodle plugin installation. Post-install setup wizard:
1. Configure Claude API key
2. Configure AWS credentials (optional)
3. Seed CSF Functions reference data
4. Initialize topic tag library
5. Create default email templates
6. Set system defaults (full proficiency thresholds, decay settings, etc.)

---

## 18. Implementation Phases

### Phase 1: Content Tagging Foundation
- CSF reference tables + topic tag library
- Content metadata tables
- Admin tagging UI
- CSV bulk import
- AI-assisted auto-tagging
- **Expert review and scoring of existing content catalog** (parallel human effort)

### Phase 2: Mastery Tracking Engine
- Mastery event listener (Moodle course completion hook)
- Dual-layer score calculation (CSF mastery + topic competency)
- Competency level computation
- Trend tracking
- Mastery events audit log

### Phase 3: Proficiency Dashboard (ship early for stakeholder buy-in)
- Radar chart component
- Cohort mastery cache
- Individual learner overlays
- Skills heatmap
- Learner self-view dashboard
- API endpoints for mastery data

### Phase 4: Proficiency Path Generator
- LLM goal interpretation
- LLM path generation (structured JSON output)
- Operator review/edit/approve UI
- Path step management
- Branching engine

### Phase 5: Campaign Management & Assignment
- Campaign creation wizard
- Enrollment management
- Content assignment workflow (structured path + AI adaptive modes)
- Progress tracking views
- Cohort sync

### Phase 6: Email Simulations
- Simulation authoring interface
- Simulation delivery mechanism
- Annotation/education pages
- Engagement tracking
- Integration with mastery tracking
- Integration with proficiency paths as steps

### Phase 7: Messaging & Notifications
- Email template system with mastery-aware placeholders
- Assignment/reminder/milestone notifications
- Manager digest with inline radar charts
- Simulation follow-up emails

### Phase 8: Passwordless Auth & External Integrations
- Token authentication plugin
- AWS SNS integration
- Moodle 5 custom reporting integration
- Web services API finalization

### Dependency Graph

```
Phase 1 (Tagging)
  ├──→ Phase 2 (Mastery Tracking)
  │      ├──→ Phase 3 (Dashboard)        ← ship early
  │      ├──→ Phase 4 (Path Generator)
  │      │      ├──→ Phase 5 (Campaigns)
  │      │      ├──→ Phase 6 (Email Sims)
  │      │      └──→ Phase 7 (Messaging)
  │      └──→ Phase 6 (sims need mastery tracking)
  └──→ Phase 4 (path gen needs tagged content)

Phase 8 (Auth & Integrations) — independent, can parallel with Phase 3+
```

---

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-01-26 | Initial OCMS integration requirements draft |
| 2.0 | 2026-03-11 | Unified with mastery system plan; added NIST CSF alignment, proficiency paths, email simulations, radar dashboard, dual-layer taxonomy |
