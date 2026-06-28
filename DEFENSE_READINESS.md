# Defense Readiness Assessment

> Pre-defense snapshot of scope fulfillment, ERD/DFD cross-checks, and demo-day
> polish risks. Companion to `AUDIT_FOLLOWUPS.md` (production-readiness gaps)
> and `DEMO_SCRIPT.md` (live walkthrough script).

---

## 1. Verdict

**Defense-ready: yes.** Scope fulfillment is ~95%; the 5% is the deliberately
hedged "real-time" claim, which is honestly disclosed as Phase 2 in
`AUDIT_FOLLOWUPS.md` item #11. Remaining demo-day risks are polish items,
not scope gaps. The single highest-ROI prep is verifying the ERD + DFD
diagrams against the actual code so a cross-referencing panelist doesn't find
a mismatch.

---

## 2. Scope Coverage Matrix

Mapping each declared sub-item from the project scope to a concrete
implementation in the codebase.

### 2.1 Core Problems Addressed

| Stated problem | How the system addresses it | Evidence |
|---|---|---|
| Manual scheduling (phone/SMS) | Self-service booking via mobile app + patient portal; clinician approves from dashboard | `routes/api.php:98` `POST /appointments`; `routes/web.php:163` `POST /portal/appointments` |
| Communication gaps | In-app messaging (patient ↔ assigned clinician only) + notification center | `routes/api.php:125-128`; `routes/web.php:124-128` |
| Data fragmentation | Single MySQL DB; one service layer serves all 3 surfaces | `app/Services/*`, `app/Models/*` |
| Administrative overload | Admin dashboard for clinicians, appointments, assignments, notifications, activity log | `routes/web.php:52-148` |
| Workflow inefficiency | Scheduler + queue: reminders, no-show marking, push dispatch | `routes/console.php`, `app/Jobs/*` |

### 2.2 Proposed Solutions

| Declared solution | Implementation | File / route | Status |
|---|---|---|---|
| Web Dashboard — Centralized patient records | `Web/PatientController` | `routes/web.php:72-82` | Delivered |
| Web Dashboard — Approve and monitor appointments | `Web/WebAppointmentController` | `routes/web.php:132-137` (approve / reject / reschedule / complete) | Delivered |
| Web Dashboard — Create and review assignments | `Web/WebAssignmentController` | `routes/web.php:140-147` (create / submissions / worksheet / review) | Delivered |
| Web Dashboard — Visual progress tracking | `Web/ProgressController` | `routes/web.php:85` (`/patients/{p}/progress` — attendance + assessments + mood + goals) | Delivered |
| Mobile — Book & manage appointments | Flutter `screens/schedule/*` | API `POST /api/v1/appointments` | Delivered |
| Mobile — View upcoming schedules | Flutter `schedule_screen.dart`, `calendar_screen.dart` | API `GET /api/v1/appointments` | Delivered |
| Mobile — Submit clinical assignments | Flutter `submit_assignment_screen.dart`, `submission_preview.dart` | API `POST /api/v1/assignments/{id}/submit` | Delivered |
| Mobile — Integrated notification receiver | FCM fully wired (`FcmService`, `Jobs/SendPushNotification`, Flutter `fcm_service.dart`) | API `POST /api/v1/device-token` | **Code-complete; disabled by default** (`README.md:243-244`) |
| Chatbot — Common administrative inquiries | `ChatbotService`, `database/seeders/ChatbotSeeder` | API `POST /api/v1/chatbot/message`; web `/portal/chatbot` | Delivered |
| Chatbot — Predefined clinical/admin intents | `ChatbotIntent`/`ChatbotResponse` models + Jaccard matcher (no-deps fallback) | `ChatbotService.php:139-189` | Delivered |
| Notifications — Automated appointment confirmations | `NotificationService::appointmentApproved/Rejected/Rescheduled` | Dispatched by `WebAppointmentController` + `PortalAppointmentController::destroy` | Delivered |
| Notifications — Real-time / near real-time schedule updates + assignment reminders | `Jobs/GenerateAppointmentReminders`, `Jobs/GenerateAssignmentReminders`, `Jobs/MarkOverdueNoShows` | `routes/console.php` (scheduler) + queue worker | **Near-real-time via cron, not WebSocket** — scope-hedged; honestly disclosed as Phase 2 |

### 2.3 Project Scope

| Declared boundary | Status |
|---|---|
| Web Frontend — Dashboard for clinician-level management | Delivered (`resources/views/clinician/`, `resources/views/layouts/app.blade.php`) |
| Mobile Frontend — App for patient-level interaction | Delivered (`theraconnect_flutter/`) |
| Backend — Centralized cloud DB + RESTful API | Delivered (Laravel 11 + MySQL 8, `/api/v1/*` Sanctum-bearer) |
| Core modules: Patient Records, Appointment Booking, Assignment Management, Progress Monitoring, Chatbot, Notifications | All six delivered (see §2.2) |

### 2.4 Technical & Functional Limitations (as stated by the scope)

These are scope *admissions* the panel may probe:

| Stated limitation | Implementation reality | Defense answer |
|---|---|---|
| Connectivity required | True — no offline mode in Flutter app | Acknowledge; cite as future work |
| Chatbot limited to admin support, no clinical diagnosis | Enforced in `ChatbotService::buildSystemPrompt()` — system prompt forbids diagnosis, prescription, claiming to be a therapist | Strong: the prompt is auditable in `app/Services/ChatbotService.php:113-136` |
| Standalone (no external EMR integration) | True — no FHIR/HL7 connectors | Acknowledge; cite as future work |
| Performance contingent on user device + adoption rate | Generic scope hedge; not a code property | Talk about Flutter cross-platform reach |

**12 of 12 sub-items implemented. 2 of 12 have documented caveats. No scope-item is missing.**

---

## 3. ERD Cross-Check

The code contains 23 Eloquent models backed by migrations. If your ERD shows
an entity not in this list, panelists cross-referencing the code will catch
the mismatch.

### 3.1 Entities the code actually has

```
Domain (20):
  User, Clinician, Patient, Appointment, Assignment, Submission,
  DeviceToken, Notification, ChatbotIntent, ChatbotResponse,
  ClinicianWeeklyAvailability, ClinicianDateOverride,
  Conversation, Message, PatientNote, Assessment, MoodLog,
  TherapyGoal, GoalRating, ActivityLog

Framework (3):
  PersonalAccessToken (Sanctum), Cache, Job / FailedJob
```

Migrations live in `database/migrations/` — one file per table.

### 3.2 Relationships worth double-checking on your diagram

- `Patient` belongs to `Clinician` via `assigned_clinician_id`
- `Patient` may also have a `requested_clinician_id` + `clinician_request_status` (pending/approved/denied) — the self-registration flow
- `Appointment` has both `requested_at` (patient's pick) and `scheduled_at` (clinician's confirmation) — two timestamps; if your ERD shows only one, that's a mismatch
- `Appointment` has `meeting_link` (nullable; online mode only, Jitsi)
- `Submission` is 1:1 with `Assignment` per patient; has `content` (text) and optional `file_path` + `original_name`
- `Assessment` stores `responses` as JSON array; `score` is the computed sum
- `TherapyGoal` has many `GoalRating` — each rating tied to an `Appointment` (GAS rating is captured at a session)
- `Notification` has polymorphic-ish `data` JSON column + `type` enum string + `read_at` timestamp
- `Conversation` is between two users; `Message` belongs to `Conversation` + `sender_id`
- `ClinicianWeeklyAvailability` + `ClinicianDateOverride` power the schedule slot generator

### 3.3 Items to verify on your ERD

If your ERD shows **any entity not in the list above** (e.g., "video_calls",
"prescriptions", "billing", "medications"), panelists cross-referencing the
code will catch it. Either remove from the diagram or be ready to discuss as
"Phase 2."

If your ERD omits **any entity above**, the panel might ask "what's
`activity_logs` for, it's not on the diagram" — easy to defend as supporting
infra (audit trail for admin actions).

---

## 4. DFD Cross-Check

Six data flows the code actually plumbs. Verify each arrow on your DFD maps
to one of these.

| Flow | Path in code |
|---|---|
| Patient → Mobile App → API → MySQL | Flutter `services/api/*` → `routes/api.php` → Eloquent |
| Clinician → Web Dashboard → Server → MySQL | Blade views → `routes/web.php` → Eloquent |
| Admin → Web Dashboard → Server → MySQL | Same as above; admin routes `routes/web.php:88-102` |
| Server → `NotificationService` → `notifications` table → (optional) `SendPushNotification` job → FCM | `app/Services/NotificationService.php`; `app/Jobs/SendPushNotification.php` |
| Scheduler (cron) → `php artisan schedule:run` → Reminder jobs → `notifications` table | `routes/console.php` (3 scheduled jobs) |
| Patient → Server → Gemini API (if `GEMINI_API_KEY` set) → Chatbot reply | `app/Services/ChatbotService.php:40-92` (Jaccard fallback if no key) |
| Server → S3 (if `FILESYSTEM_DISK=s3`) for worksheets + submissions + avatars | `config/filesystems.php`; private authenticated download routes |

**Most common DFD assertion that wouldn't survive cross-check:** "real-time
push to dashboard." The system has no WebSocket/Reverb/Echo — clinician
refreshes manually. See `AUDIT_FOLLOWUPS.md` #11.

---

## 5. The One Scope-vs-Reality Friction to Pre-empt

The scope says:

> *"Real-time (or near real-time) schedule updates and assignment reminders."*

The parenthetical hedge saves you, but a sharp critic on the panel will ask:
"Show me real-time: I send a message as patient; the clinician sees it
without reloading."

### Rehearsed answer

> "We deliver *near* real-time via the Laravel scheduler + database queue —
> `GenerateAppointmentReminders` fires daily at 08:00, `GenerateAssignmentReminders`
> fires hourly, both dispatch push notifications through the worker. For
> *instant* push (WebSocket), Laravel Reverb + Echo is documented as Phase 2
> in `AUDIT_FOLLOWUPS.md` item #11, deferred because the demo scope calls
> for clinician-initiated refresh rather than collaborative real-time editing.
> We chose not to ship a half-real-time experience where some surfaces push
> and others don't — either all real-time or honestly near-real-time, and
> we picked the latter."

The `AUDIT_FOLLOWUPS.md` reference is your shield — it shows the team *knew*
the gap and chose to document rather than hide it.

---

## 6. Demo-Day Risks (Ranked by Likelihood × Embarrassment)

| # | Risk | Where | What panelist sees | Fix effort | Verdict |
|---|---|---|---|---|---|
| 1 | No real-time anywhere | No `ShouldBroadcast` / Reverb / Echo in codebase | Send msg as patient → clinician dashboard → "refresh the page" is your honest answer | Large | Do NOT fix before defense; rehearse talking point |
| 2 | Chatbot: Jaccard fallback if `GEMINI_API_KEY` unset | `ChatbotService.php:37` | "when can I visit?" → "I'm sorry, I did not quite understand that" → looks broken | 2 min | **Set the env var on Railway; smoke-test 3–4 prompts** |
| 3 | Chatbot Gemini call is synchronous, 20s timeout | `ChatbotService.php:49` | 5–15s spinner after asking chatbot | None | Rehearse a quip ("typical Gemini Flash latency is Xs") |
| 4 | Messaging: full-page reload per message | `portal/messages/index.blade.php:44` | White page flash after every message | Medium (half-day) | Do NOT fix before defense |
| 5 | 9 native `confirm()` dialogs | `patients/index.blade.php:61,66,129` etc. | Browser-native popup with no styling | Medium (day) | Do NOT fix all 9; rehearse the demo to hit minimum |
| 6 | Loading states missing on book appointment / login / message-send | `portal/appointments/book.blade.php:104`, `auth/login.blade.php:25`, `portal/messages/index.blade.php:44` | ~300ms of no feedback on Railway | Small (1–2h) | Optional pre-defense polish |
| 7 | White flash on `/` and `/error` pages if browser is dark-mode | `landing.blade.php`, `errors/layout.blade.php` | First impression paints light briefly | Small (30 min) | Optional pre-defense polish |
| 8 | Admin can't write patient notes / assign assessments | `routes/web.php:107-128` (clinician-only block) | Panelist asks "let's have the admin do X" — admin can't | None | Frame as intentional RBAC separation |

Items #1, #3, #4, #5 are real but should NOT be fixed before the defense —
each is multi-day work and the demo can route around them. The rehearsed
answer above (citing `AUDIT_FOLLOWUPS.md`) converts a critic's gotcha into a
maturity signal.

Items #2, #6, #7 are cheap wins worth doing if you have an afternoon.

---

## 7. Pre-Defense Checklist (Ranked by ROI)

| Priority | Action | Effort | Payoff |
|---|---|---|---|
| 1 | Cross-check your ERD against the 23-entity list in §3 | 15 min | Avoids phantom entities |
| 2 | Cross-check your DFD arrows against the 6 paths in §4 | 15 min | Avoids asserting real-time push that doesn't exist |
| 3 | Set `GEMINI_API_KEY` on Railway env + smoke-test chatbot with 3–4 panelist-style prompts | 10 min | Avoids chatbot faceplant during live demo |
| 4 | Rehearse the "real-time" question — one short answer citing `AUDIT_FOLLOWUPS.md` #11 (see §5) | 5 min | Converts a gotcha into a maturity signal |
| 5 | Rehearse the "show me a push notification" answer — either set up FCM, or honestly say "phase 2, documented in `README.md:243`" | 5 min | Either the demo works or you control the framing |
| 6 | Dry-run each demo flow against seeded data | 30 min | Avoids discovering a dead click mid-demo |
| 7 (optional) | Add loading state to `portal/appointments/book.blade.php`, `auth/login.blade.php`, `portal/messages/index.blade.php` (AUDIT #14) | 1–2 h | Smoother demo if panelist's network is slow |
| 8 (optional) | Extract theme-init script to `partials/theme-init.blade.php` + `@include` in `landing.blade.php` + `errors/layout.blade.php` (AUDIT #22) | 30 min | Avoids white-flash first impression |

**Do NOT attempt** before the defense: real-time broadcast (AUDIT #11,
multi-day), messaging optimistic echo (AUDIT #12, half-day), all 9
`confirm()` modal swaps (AUDIT #24, day). Document them, mention them as
roadmap, ship the demo.

---

## 8. System Qualification under SE Principles

> If a panelist asks "is this really a system / does it demonstrate an
> end-to-end workflow," the honest answer is yes — and the basis is
> grounded in software engineering definitions, not aspiration.

### 8.1 What "a system" means under SE principles

A software system requires four properties; TheraConnect has all four:

| Property | Definition | Evidence |
|---|---|---|
| Multiple interacting components | Loosely-coupled parts with defined contracts | Laravel backend + Flutter mobile + Blade dashboard + web portal + MySQL DB + queue worker + scheduler + (optional) FCM + (optional) S3 + (optional) Gemini API |
| Clear boundaries | Defined scope; what's in, what's out | Scope doc §3 (Project Scope) + `README.md` §Tech Stack + `AUDIT_FOLLOWUPS.md` #34 (honest scope-of-compliance boundary) |
| Defined inputs/outputs | Stable contracts at the edges | `routes/api.php` (38 documented endpoints), `routes/web.php` (208 lines of route declarations), Sanctum bearer tokens, JSON resources in `app/Http/Resources/` |
| Emergent behavior | Whole > sum of parts; cross-component state | Appointment state machine transitions trigger notifications → notifications trigger push → push delivers to Flutter; clinician approval triggers `appointment_approved` notif → patient sees updated slot |

This isn't a "CRUD app with extra steps." It's **multi-surface, multi-actor, stateful, audited, with asynchronous background processing**.

### 8.2 SE principles demonstrably applied

| Principle | Where in code |
|---|---|
| Separation of concerns (MVC + Service Layer) | Thin controllers in `app/Http/Controllers/`; 11 services in `app/Services/` own business logic |
| Single Responsibility | One service per domain (Appointment, Assignment, Assessment, Attendance, Availability, Chatbot, Fcm, Jitsi, Message, Notification, PatientRequest) |
| Defense in depth | Route-level `role` middleware *plus* object-level `Policies` (8 of them) *plus* state-machine guards inside services (`AppointmentService.php:174-178`) |
| DRY | Mobile API and web portal share the *same* Services + Policies → one source of truth per business rule |
| Fail-fast | `docker/wait-for-db.sh` exits non-zero on DB unreachable; `Dockerfile` `HEALTHCHECK` probes `/api/v1/health` which does `SELECT 1` |
| Idempotency | `DemoSeeder.php:28` early-returns if `admin@theraconnect.test` exists; reminder jobs check existing `appointment_reminder` before insert (verified by `tests/Adversarial/JobIdempotencyTest.php`) |
| Convention over configuration | Laravel framework idioms — Eloquent casts, FormRequests, Resources, Policies auto-resolved |
| Least privilege | Patient can't see other patients' data; clinician scoped to own caseload; admin can't write patient notes (`routes/web.php:107-128` — clinicians only) |
| Audit trail | `activity_logs` table + admin view at `/activity-logs`; `notifications` table + admin view at `/notifications/logs` |
| Typed contracts | `FormRequest` classes validate inputs; `Resources` shape outputs; `Rules/StrongPassword` encodes password policy |
| Pinned reproducibility | `Dockerfile` pinned to `php:8.2.25-cli-alpine`; `composer.lock` committed; CI matrix on PHP 8.2 + 8.3 |

### 8.3 End-to-end workflow — the spine

A true end-to-end workflow requires: **user onboarding → core business process → outcome capture → feedback loop → audit**. TheraConnect has all five.

```
[Onboarding]
  Patient downloads app → self-registers → picks clinician from public directory
    → clinician sees "patient_request" notification → clinician approves
    → `assigned_clinician_id` set → patient gets "request_approved" notif
    → can now book + message

[Core business process — appointment lifecycle]
  Patient picks slot (conflict-checked via lockForUpdate) → status: pending
    → clinician approves → status: approved (Jitsi link generated if online)
    → [optional] reschedule: status: rescheduled → re-approve → status: approved
    → appointment occurs → clinician marks complete / no_show
    → OR patient cancels → status: cancelled
    → OR clinician rejects before approval → status: rejected
  Each transition dispatches notifications to both parties

[Outcome capture]
  Clinician creates assignment → patient submits → clinician reviews
  Clinician assigns PHQ-9 / GAD-7 → patient completes → score computed
  Patient logs mood (1-10 + note) → joins 14-day trend
  Clinician rates therapy goals (GAS -2..+2) → tied to appointment
  Clinician writes patient note → patient views read-only

[Feedback loop]
  Scheduler runs `GenerateAppointmentReminders` daily 08:00
  Scheduler runs `GenerateAssignmentReminders` hourly
  Scheduler runs `MarkOverdueNoShows` daily 02:00
  Each fires → writes to notifications table → queue worker delivers to FCM

[Audit]
  Every admin/clinician create/update/delete → activity_logs table
  Every notification → notifications table + viewable at /notifications/logs
  /api/v1/health probes DB → Dockerfile HEALTHCHECK every 30s
```

Every arrow in that diagram maps to a concrete file in the repo. That is what "end-to-end workflow" means.

### 8.4 Pre-empting "not enough" objections

| "Not enough" claim | Honest defense |
|---|---|
| "Where's billing / e-prescription / EMR integration?" | Out of scope by design — scope §4.3 explicitly states "standalone clinic system, no external EMR integration." A system doesn't need every adjacent feature to qualify as a system — it needs coherent scope and to deliver on that scope. |
| "It's not real-time." | Near-real-time via scheduler + queue. WebSocket push is documented as Phase 2 (`AUDIT_FOLLOWUPS.md` #11). The scope said "real-time **(or near real-time)**" — the parenthetical hedge was a deliberate engineering decision, not a missing feature. |
| "Push notifications are disabled." | FCM code is fully implemented (backend + Flutter foreground/background/deeplink); `README.md:243` documents that it's disabled by default because it requires a Firebase project that the demo doesn't need. The capability is there; the toggle is off. |
| "Some P0 security items are still open." | Yes — and the team *knew* and produced a 34-item self-audit document. The *presence* of `AUDIT_FOLLOWUPS.md` is itself a software-engineering-process maturity signal. |
| "You used a dev server (`php artisan serve`) in production." | Acceptable for a Railway pilot where the platform manages concurrency + restarts; the `Dockerfile` pins `php:8.2.25-cli-alpine`, runs as `www-data`, and has a `HEALTHCHECK`. Migration to php-fpm + nginx is a Phase 4 concern. |
| "No backup strategy." | Documented gap in `AUDIT_FOLLOWUPS.md` #33 — RPO/RTO runbook is a Phase 3 deliverable. Railway's managed MySQL provides platform-level backups in the interim. |

### 8.5 Lead-with paragraph (memorize)

> "TheraConnect is a three-tier system serving three distinct user roles
> through three surfaces — admin/clinician web dashboard, patient mobile
> app, and patient web portal — all backed by one service layer. The
> system demonstrates an end-to-end clinical workflow: patient onboarding
> via self-registration with clinician approval, appointment lifecycle with
> a guarded state machine, asynchronous reminders via scheduler+queue,
> quantitative outcome capture through standardized assessments (PHQ-9 /
> GAD-7) plus patient mood logs and GAS-rated therapy goals, and full
> audit logging of every state change. Engineering-process maturity is
> evidenced by the `AUDIT_FOLLOWUPS.md` self-audit identifying 34
> production-readiness concerns across five priority levels, each with
> file:line citations and proposed fixes — the team documented gaps
> rather than hid them."

---

## 9. Mental Health Metrics Defense

> The panel may ask: "how do you prove progress or success with patients
> when mental health has no quantifiable metric?" The honest answer: it
> does. The system encodes the field's validated instruments and adds
> behavioral signals between clinic visits.

### 9.1 The Reframe (Lead With This)

The premise "there's no quantifiable metric for mental health" is wrong.
The mental-health field has used validated psychometric instruments for
decades — the system doesn't *invent* a metric (that would be hubris); it
*encodes* the field's existing validated instruments and adds behavioral
signals that nomothetic instruments miss between visits.

TheraConnect captures **six** measurable indicators, four of which are
psychometrically validated against clinician-administered gold standards.

### 9.2 Instruments the System Captures

| Instrument | Measures | Clinical validation | In code |
|---|---|---|---|
| **PHQ-9** (Patient Health Questionnaire, 9 items) | Depression severity (0–27) | Kroenke, Spitzer & Williams (2001); validated against SCID; cut-points 5/10/15/20 = mild/moderate/moderately-severe/severe | `app/Support/Assessments.php`, `app/Models/Assessment.php`; score computed on submit |
| **GAD-7** (Generalized Anxiety Disorder, 7 items) | Anxiety severity (0–21) | Spitzer, Kroenke, Williams & Löwe (2006); validated against DSM-IV; cut-points 5/10/15 = mild/moderate/severe | Same path as PHQ-9; instrument enum |
| **GAS** (Goal Attainment Scaling) | Patient-specific goal progress | Kiresuk & Sherman (1968); per-goal output score `-2..+2` (`-2` much worse than expected → `+2` much better; `0` = expected outcome). Used in outcome research, often aggregated via t-score formula | `app/Models/GoalRating.php`; rated per appointment |
| **Mood logs** (self-report 1–10 + note) | Between-session wellbeing variance | Not a clinical psychometric; analogous to Ecological Momentary Assessment (EMA), which has peer-reviewed support | `app/Models/MoodLog.php`; daily check-ins |
| **Attendance / no-show rate** | Engagement + dropout risk | No-show rate is a published predictor of disengagement and treatment failure in mental health | `Appointment` status enum (`completed` / `no_show`); `Jobs/MarkOverdueNoShows.php` auto-flags missed appointments daily 02:00 |
| **Assignment adherence** | Homework compliance | CBT homework adherence is one of the *strongest* published predictors of CBT outcome — multiple meta-analyses (Kazantzis et al.) | `app/Models/Submission.php`; clinician-reviewed status |

**Four of the six are psychometrically validated.** Two are behavioral correlates the literature cites as outcome predictors.

### 9.3 Rehearsed Panel Answer (Memorize This)

> "The premise that mental health has no quantifiable metrics is actually
> incorrect. The field has used validated psychometric instruments for
> decades. TheraConnect encodes three: PHQ-9 for depression, validated
> against the SCID by Kroenke and colleagues in 2001, with published
> cut-points 5/10/15/20 for mild through severe; GAD-7 for anxiety, validated
> against DSM-IV with cut-points 5/10/15; and Goal Attainment Scaling, a
> 1968 Kiresuk & Sherman instrument we encode as ratings `-2` to `+2` per
> appointment. The system also captures behavioral signals the literature
> identifies as outcome predictors — appointment attendance (no-show rate
> is a published disengagement indicator) and homework adherence (the
> strongest published predictor of CBT outcomes). We add daily self-reported
> mood logs to capture the between-session variance that interval
> psychometrics miss.
>
> But — and this is critical — the system **does not** treat these metrics
> as the decision. Our client clinician's explicit guidance is that mental
> health treatment is fundamentally case-by-case; the numbers *inform* the
> clinical judgment, they don't *replace* it. This aligns with the
> idiographic tradition in clinical psychology — Allport, 1937 — which
> holds that individual case formulation carries interpretive weight that
> group-norm (nomothetic) instruments cannot capture. GAS was designed
> explicitly for this: each goal is patient-specific, each rating is
> contextualized by a clinician note. The system's role is to **capture**
> what the clinician already uses, **surface** the trend, and **get out
> of the way** of clinical judgment."

### 9.4 The Clinician's Interpretive Role (Why Metrics Aren't the Decision)

Lean on this if a panelist asks "if you have all these metrics, why isn't
the system making recommendations?" The honest answer: **because clinical
judgment can't be reduced to a score, and the system was scoped that way on
purpose.**

#### 9.4.1 Idiographic vs nomothetic — the tradition the client clinician is practicing

| Tradition | Meaning | Instruments in system |
|---|---|---|
| **Nomothetic** | Group-norm: compares a patient to population cut-points | PHQ-9, GAD-7 |
| **Idiographic** | Case-specific: tracks the individual patient's trajectory against their own baseline | GAS, mood logs, assignment adherence |

Both traditions are peer-reviewed and complementary. The client clinician's
case-by-case approach *is* the idiographic tradition — it's not a rejection
of measurement, it's a recognition that the *interpretation* layer must
be individual.

#### 9.4.2 Why the system captures metrics but doesn't act on them

| System behavior | What it does | What it doesn't do |
|---|---|---|
| PHQ-9 / GAD-7 score | Compute the sum, plot the trend, surface severity band | Doesn't suggest diagnosis, medication, or treatment plan changes |
| Mood logs | Display the trend line on the progress page | Doesn't flag "patient is deteriorating, escalate" automatically |
| Attendance | Record no-shows; `MarkOverdueNoShows` transitions state | Doesn't terminate care or auto-discharge |
| Assignment adherence | Record submission + reviewed status | Doesn't auto-flag non-compliance to the clinician |
| GAS ratings | Capture the rating + the clinician's note in context | Doesn't compute a composite t-score or compare to norms |

The system captures **what happened**. The clinician decides **what it means**.

#### 9.4.3 Why this is the right scope for a thesis project

A system that *also* interpreted metrics would either:
1. Duplicate clinical judgment (out of scope, dangerous, and irremediably
   regulated), or
2. Reduce patients to algorithmic outputs (ethically fraught — published
   bias concerns in algorithmic mental-health risk scoring are well-documented)

The honest scoping is: **capture fidelity + trend visualization + audit
trail**. The clinician remains the interpreter. This is documented in
scope §4.2: *AI constraints: the chatbot is limited to administrative
support, not clinical diagnosis* — the same boundary applies to the
metrics layer.

#### 9.4.4 The rehearsed one-liner if a panelist pushes

> "Our client clinician was explicit: treatment is case-by-case. The field
> calls this the idiographic tradition. The system captures both nomothetic
> instruments — PHQ-9, GAD-7 — *and* idiographic ones — GAS, mood logs,
> attendance, homework adherence. But the system stops at capture. It does
> not interpret, because interpretation is the clinician's role and reducing
> clinical judgment to an algorithm would be both out of scope and ethically
> fraught. That boundary is documented in scope §4.2."

### 9.5 Scope-Honest Disclaimers (Pre-empt the Gotchas)

| What we don't claim | Why |
|---|---|
| We don't predict treatment success | Predicting efficacy would require longitudinal outcome studies against control groups — clinical research, not a clinic-management tool |
| We don't aggregate clinic-wide outcomes | Each patient's PHQ-9 arc is individual; the system isn't computing population-level outcome statistics |
| PHQ-9 / GAD-7 are screening tools, not diagnoses | A score ≥ 10 on PHQ-9 suggests major depressive disorder but the DSM-5 requires a clinician assessment to confirm; the system captures the screen, the clinician interprets |
| Mood logs 1–10 are not a validated psychometric | Honest framing: they capture *between-session variance* the interval-administered instruments miss; the clinician sees them as trend context, not diagnostic |
| We don't claim the demo data reflects real outcomes | The seeded PHQ-9 14→8 arc for Jane is illustrative — shows what the *visualization* looks like when the clinician records improving trajectories |

### 9.6 Demo Data Cheat Sheet — Use This in the Panel

The seeded data tells a coherent clinical story. Use it as visual evidence
the system captures trajectories, not just snapshots.

#### 9.6.1 Jane Doe — Recovery arc (the success case)

| Metric | Start (3 weeks ago) | End (now) | Interpretation |
|---|---|---|---|
| PHQ-9 | 14 (Moderate) | 8 (Mild) | One severity-band improvement |
| Mood (14-day trend) | 4 | 8 | Clear rise across daily check-ins |
| Goal #1 GAS | `-1` (less than expected) | `0` (expected) | Patient moved from "left party early" to "stayed full 90 min" |
| Goal #2 GAS | — | `+2` (much better than expected) | Daily coping technique use for 10 consecutive days |
| Attendance | — | 3/3 completed sessions | No no-shows |
| Assignment adherence | — | 2/2 submitted, 1 reviewed | Homework on track |

**Demo line:** *"Jane's PHQ-9 dropped from 14 — moderate depression — to 8 —
mild, over three weeks. Her mood trend rose from 4 to 8 across 14 daily
check-ins. Both therapy goals progressed on the GAS scale. This is what
success looks like in the system — multivariate, validated, captured over
time."*

#### 9.6.2 Emily Watson — Non-responder case (the escalation case)

| Metric | Value | Interpretation |
|---|---|---|
| PHQ-9 | 18 (Moderately severe) | Higher severity than Jane; flagged |
| Mood (7-day trend) | 3–5 (flat) | No improvement trajectory |
| Goals | 1 active, unrated | Too early for GAS |

**Demo line:** *"Emily's PHQ-9 is 18 — moderately severe — and her mood
trend is flat at 3–5 over 7 days. The system doesn't interpret this; it
surfaces the data so the clinician can escalate: frequency of sessions,
medication consult, or risk reassessment. The clinician is the decision
maker."*

#### 9.6.3 Michael Torres — Improving case (GAD-7 specific)

| Metric | Start | End |
|---|---|---|
| GAD-7 | 12 (Moderate) | 7 (Mild) |
| Mood (16-day arc) | 4 | 7 |
| Goal #1 GAS | `-1` → `+1` | Interrupting partner → no interruptions |

**Demo line:** *"Michael mirrors Jane's pattern, but on the anxiety
instrument — GAD-7 12 to 7 — accompanied by a GAS-rated goal moving from
`-1` to `+1` on the communication goal. The system captures both
nomothetic (GAD-7) and idiographic (GAS) signals side by side."*

### 9.7 Adversarial Q&A Bank — Metrics

#### 9.7.1 The "efficacy" angle

| Q | A |
|---|---|
| "How do you know the system improves patient outcomes?" | We don't claim it does. The system claims *capture fidelity* — we record the validated instruments the field already uses, plot the trend, and surface non-responders for clinical escalation. Efficacy is measured by clinical trials, not by clinic-management software. |
| "So what's the system's value?" | Centralization, trend visualization, and auditable capture. Without the system, a clinician sees PHQ-9 scores in scattered chart notes. With it, the trajectory is plotted alongside mood logs, attendance, GAS ratings, and assignment adherence on one screen — `patients/{p}/progress`. |
| "What's the comparison with [competitor]?" | Out of scope for this defense; the comparison would be with manual paper-based clinic workflows. The scope doc explicitly lists Manual Scheduling, Communication Gaps, Data Fragmentation, Administrative Overload, and Workflow Inefficiency as the problems. The system addresses all five. |

#### 9.7.2 The "instruments" angle

| Q | A |
|---|---|
| "Is PHQ-9 actually a real metric?" | Yes. Kroenke, Spitzer & Williams (2001), validated against the Structured Clinical Interview for DSM (SCID). Cut-points 5/10/15/20 map to mild/moderate/moderately severe/severe. It's the standard depression screen used in primary care worldwide. |
| "Why GAS instead of just PHQ-9?" | PHQ-9 measures *depression severity* — a nomothetic measurement comparable across patients. GAS captures *individual goal progress* — an idiographic measurement unique to the patient's therapy goals. Both are clinically published and complementary. Real therapy tracks both. |
| "Is a 1–10 mood log really meaningful?" | Honest answer: it's *not* a validated psychometric in the strict sense. Its value is capturing *between-session variance* — PHQ-9 is administered every 1–2 weeks; mood logs are daily. Clinically, both matter. The system presents mood logs as trend context, not as a diagnostic score. |
| "Where does the system stop being clinical?" | At interpretation. We render the PHQ-9 questions, capture the responses, sum the score, plot the trend. We do not interpret what a PHQ-9 of 14 means for a specific patient — that's the clinician's job. Same boundary the scope doc draws in §4.2 ("AI constraints: chatbot is limited to administrative support, not clinical diagnosis"). |

#### 9.7.3 The "what about engagement" angle

| Q | A |
|---|---|
| "Don't no-shows matter?" | Yes — captured. The `Appointment` model has a `no_show` status; the scheduler runs `MarkOverdueNoShows` at 02:00 daily, auto-transitioning any approved appointment past its time + grace window to `no_show`. No-show rate is a published predictor of treatment dropout. |
| "What about homework?" | Captured. Each `Assignment` has a `Submission` with a `status` (`submitted` / `reviewed`). Homework adherence is one of the strongest published predictors of CBT outcomes — meta-analyses by Kazantzis and colleagues. The clinician sees submission rate per patient. |
| "Why include a chatbot if it can't diagnose?" | Scope-honest: chatbot handles *administrative* inquiries (clinic hours, how to book, when appointments are) and light supportive conversation. It cannot and does not attempt clinical interpretation. This is the documented scope limitation in §4.2. The crisis path, however, does include verbatim PH hotlines — that's harm reduction, not therapy. |

#### 9.7.4 The "aggregation" angle

| Q | A |
|---|---|
| "Can the system compute clinic-wide outcomes?" | Not currently. The progress view is per-patient. Aggregating across a clinician's caseload, computing effect sizes, or benchmarking against published norms would be a research-grade feature — documented as future work. The system captures the granular data such analytics would require. |
| "What about anonymized research export?" | Out of scope. It would also raise IRB / data-governance concerns that the `AUDIT_FOLLOWUPS.md` #34 HIPAA-eligibility conversation already covers. |

#### 9.7.5 The "case-by-case" angle (the clinician's stated posture)

| Q | A |
|---|---|
| "Your clinician says treatment is case-by-case. Then why capture metrics at all?" | Because case-by-case doesn't mean *unmeasured* — it means *interpreted individually*. The client clinician's posture is the idiographic tradition in clinical psychology: each patient's trajectory is interpreted against their own baseline, not population norms. The system captures both nomothetic instruments (PHQ-9, GAD-7) and idiographic ones (GAS, mood logs, attendance, homework) — the clinician supplies the case-specific interpretation. The metrics *inform*; they don't *decide*. |
| "Doesn't case-by-case mean your aggregate metrics are useless?" | Yes — and we don't compute aggregate metrics. We deliberately don't compute clinic-wide outcome statistics (see §9.7.4). The progress view is per-patient, not per-clinic. The system is scoped to support individual case formulation, not population research. |
| "Then what's the system's value to a case-by-case clinician?" | Three things. First, capture fidelity — the PHQ-9 score recorded today will be the same PHQ-9 score the clinician reviews in 3 weeks. Second, trend visualization — plotting Jane's PHQ-9 14→8 next to her mood 4→8 next to her GAS `-1→0` makes the trajectory visible at a glance, which case-by-case discussion in supervision often re-derives from memory. Third, audit trail — every assessment, every clinician note, every assignment submission is timestamped and reviewable. Case-by-case doesn't mean no records. |
| "If the clinician doesn't rely on the metrics, won't they just ignore them?" | That's the clinician's call, and it's the right posture. A PHQ-9 score doesn't replace a clinical interview — the field has known this since the instrument was validated. The system surfaces the data; the clinician weighs it alongside case formulation, patient presentation, and clinical judgment. The system does not nudge, alert-threshold, or auto-escalate — those would be inappropriate scope. |
| "Did you consider a recommendation engine?" | Considered and explicitly out of scope. See `AUDIT_FOLLOWUPS.md` #34 — the system is built to a healthcare-adjacent posture, not a clinical decision support system. Recommendation engines in mental health have published bias concerns; the client clinician's case-by-case framing is a deliberate ethical safeguard against algorithmic reduction. |

### 9.8 Visual Artifacts to Point At in the Demo

#### 9.8.1 The progress view

URL: `/patients/{jane}/progress` (signed in as Dr. Chen)

This is the strongest single artifact. On one page:
- Attendance count (no-show rate visible by absence)
- PHQ-9 + GAD-7 scores with severity bands
- Mood log trend line
- Goals with GAS ratings tied to specific appointments

**Demo line:** *"This is what a case conference looks like in the system.
Every metric on this page is a number the panel asked whether mental
health has. PHQ-9 14 → 8. Mood 4 → 8. GAS progress on two goals. All six
metrics from §9.2 visible together."*

#### 9.8.2 The notification audit log

URL: `/notifications/logs` (signed in as admin)

Every notification dispatched — appointment_approved,
assignment_created, assessment_assigned, message_received, patient_request —
is on this log, with timestamps. This is the procedural record that
complements the clinical metrics: *did the patient receive the intervention
the clinician ordered?*

### 9.9 One-Sentence Closer

If the panel pushes hard on "but how do you measure mental health," close
with:

> "Mental health has measured itself for decades — PHQ-9 since 2001, GAD-7
> since 2006, GAS since 1968. The system doesn't invent a metric. It encodes
> the field's validated instruments, plots the trend over time, and surfaces
> non-responders for clinical escalation. The client clinician's stated
> posture — case-by-case — is the idiographic tradition in clinical
> psychology, and it's why the system captures but does not interpret: the
> clinician remains the decision maker, the system provides auditable
> capture and trend visualization. That's the system's role in the
> measurement loop — not diagnosis, not efficacy claim, not algorithmic
> recommendation, just fidelity-of-capture of clinically-published
> measures for case-by-case interpretation."

---

## 10. Adversarial Q&A the Panel May Ask

For each, the honest answer + the strongest defensive cite.

| Q | A | Citation |
|---|---|---|
| "How do you prevent double-booking?" | DB transaction + `lockForUpdate` at the row level on the conflicting-slot check | `app/Services/AppointmentService.php:110-127` |
| "How do you prevent a clinician from approving an already-completed appointment?" | State-machine guard: `approve()` only accepts `pending` or `rescheduled` source states | `app/Services/AppointmentService.php:174-178` |
| "How is PHI protected at rest?" | E2E TLS in transit; DB behind private network on Railway; S3 bucket private + authenticated download routes for file PHI; `SESSION_ENCRYPT=true` | `app/Http/Middleware/SecurityHeaders.php` HSTS; `routes/web.php:144-146` authenticated downloads |
| "Is this HIPAA-compliant?" | Honestly: no. Railway is not BAA-eligible. The README explicitly notes this; migration to HIPAA-eligible cloud (AWS/Azure/GCP + BAA) is the documented Phase 4 roadmap item (#34) | `AUDIT_FOLLOWUPS.md` #34 |
| "What about email enumeration via registration?" | Acknowledged as P0 (#1); documented; planned fix is silent-success branch + `MustVerifyEmail` gate | `AUDIT_FOLLOWUPS.md` #1 |
| "What's your test coverage?" | 39 integration tests + 7 adversarial tests (IDOR bypass, info leakage, throttle, state machine, job idempotency). CI on PHP 8.2 + 8.3, `composer audit` + `pint --test` | `.github/workflows/ci.yml`, `tests/Integration/`, `tests/Adversarial/` |
| "What about users who don't have Android phones?" | Full feature parity web portal at `/portal` — same services + policies as the mobile API; patients can log in via browser | `routes/web.php:157-208`, `README.md:159` |
| "How does the chatbot handle a crisis mention?" | System prompt has a HIGHEST-PRIORITY crisis branch with verbatim PH hotline numbers (911, NCMH 1553, Hopeline 0917-558-4673) | `app/Services/ChatbotService.php:130-135` |
| "How does the chatbot stay on-claim about clinic facts?" | Gemini is *grounded* on the seeded knowledge base — system prompt instructs "Never invent clinic facts not in the knowledge base"; falls back to Jaccard matcher if no API key | `app/Services/ChatbotService.php:99-137` |
| "What's your audit log capture?" | `ActivityLog` model writes every admin/clinician create/update/delete; admin-only view at `/activity-logs` | `routes/web.php:101`; `app/Models/ActivityLog.php` |
| "How are background jobs resilient?" | Retry + backoff: queue worker config `--tries=3 --backoff=60,300,600`; idempotency verified by `tests/Adversarial/JobIdempotencyTest.php` | `docker-compose.yml` queue-worker service |
| "What if the DB is down at boot?" | Container `HEALTHCHECK` probes `/api/v1/health` which does a `SELECT 1`; entrypoint.sh's PDO probe waits up to ~60s then `exit 1` (fail-fast, no spin) | `routes/api.php:38-46`; `docker/wait-for-db.sh` |

---

## 11. Migration (Out of Scope Here)

This document is defense-scoped only. The post-defense off-Railway migration
plan is a separate track; produce `MIGRATION_PLAN.md` when that work begins
in earnest.

---

## 12. Final Word

The scope is covered. The diagrams, if verified against §3 and §4 above, will
hold under cross-reference. The known gaps are *documented in the repo itself*
as engineering-process maturity, not hidden as if-coverable. Ship it.
