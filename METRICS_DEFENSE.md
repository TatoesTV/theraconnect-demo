# Mental Health Metrics — Defense

> Companion to `DEFENSE_READINESS.md` and `DEMO_SCRIPT.md`. The panel may
> ask: "how do you prove progress or success with patients when mental
> health has no quantifiable metric?" The honest answer: it does. The
> system encodes the field's validated instruments and adds behavioral
> signals between clinic visits.

---

## 1. The Reframe (Lead With This)

The premise "there's no quantifiable metric for mental health" is wrong.
The mental-health field has used validated psychometric instruments for
decades — the system doesn't *invent* a metric (that would be hubris); it
*encodes* the field's existing validated instruments and adds behavioral
signals that nomothetic instruments miss between visits.

TheraConnect captures **six** measurable indicators, four of which are
psychometrically validated against clinician-administered gold standards.

---

## 2. Instruments the System Captures

| Instrument | Measures | Clinical validation | In code |
|---|---|---|---|
| **PHQ-9** (Patient Health Questionnaire, 9 items) | Depression severity (0–27) | Kroenke, Spitzer & Williams (2001); validated against SCID; cut-points 5/10/15/20 = mild/moderate/moderately-severe/severe | `app/Support/Assessments.php`, `app/Models/Assessment.php`; score computed on submit |
| **GAD-7** (Generalized Anxiety Disorder, 7 items) | Anxiety severity (0–21) | Spitzer, Kroenke, Williams & Löwe (2006); validated against DSM-IV; cut-points 5/10/15 = mild/moderate/severe | Same path as PHQ-9; instrument enum |
| **GAS** (Goal Attainment Scaling) | Patient-specific goal progress | Kiresuk & Sherman (1968); per-goal output score `-2..+2` (`-2` much worse than expected → `+2` much better; `0` = expected outcome). Used in outcome research, often aggregated via t-score formula | `app/Models/GoalRating.php`; rated per appointment |
| **Mood logs** (self-report 1–10 + note) | Between-session wellbeing variance | Not a clinical psychometric; analogous to Ecological Momentary Assessment (EMA), which has peer-reviewed support | `app/Models/MoodLog.php`; daily check-ins |
| **Attendance / no-show rate** | Engagement + dropout risk | No-show rate is a published predictor of disengagement and treatment failure in mental health | `Appointment` status enum (`completed` / `no_show`); `Jobs/MarkOverdueNoShows.php` auto-flags missed appointments daily 02:00 |
| **Assignment adherence** | Homework compliance | CBT homework adherence is one of the *strongest* published predictors of CBT outcome — multiple meta-analyses (Kazantzis et al.) | `app/Models/Submission.php`; clinician-reviewed status |

**Four of the six are psychometrically validated.** Two are behavioral correlates the literature cites as outcome predictors.

---

## 3. Rehearsed Panel Answer (Memorize This)

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

---

## 4. The Clinician's interpretive role (and why metrics aren't the decision)

This is the section to lean on if a panelist asks "if you have all these
metrics, why isn't the system making recommendations?" The honest answer:
**because clinical judgment can't be reduced to a score, and the system
was scoped that way on purpose.**

### 4.1 Idiographic vs nomothetic — the tradition your client clinician is practicing

| Tradition | Meaning | Instruments in system |
|---|---|---|
| **Nomothetic** | Group-norm: compares a patient to population cut-points | PHQ-9, GAD-7 |
| **Idiographic** | Case-specific: tracks the individual patient's trajectory against their own baseline | GAS, mood logs, assignment adherence |

Both traditions are peer-reviewed and complementary. The client clinician's
case-by-case approach *is* the idiographic tradition — it's not a rejection
of measurement, it's a recognition that the *interpretation* layer must
be individual.

### 4.2 Why the system captures metrics but doesn't act on them

| System behavior | What it does | What it doesn't do |
|---|---|---|
| PHQ-9 / GAD-7 score | Compute the sum, plot the trend, surface severity band | Doesn't suggest diagnosis, medication, or treatment plan changes |
| Mood logs | Display the trend line on the progress page | Doesn't flag "patient is deteriorating, escalate" automatically |
| Attendance | Record no-shows; `MarkOverdueNoShows` transitions state | Doesn't terminate care or auto-discharge |
| Assignment adherence | Record submission + reviewed status | Doesn't auto-flag non-compliance to the clinician |
| GAS ratings | Capture the rating + the clinician's note in context | Doesn't compute a composite t-score or compare to norms |

The system captures **what happened**. The clinician decides **what it means**.

### 4.3 Why this is the right scope for a thesis project

A system that *also* interpreted metrics would either:
1. Duplicate clinical judgment (out of scope, dangerous, and irremediably
   regulated), or
2. Reduce patients to algorithmic outputs (ethically fraught — see the
   2021 epstein/of Optum's algorithm flagging bias concerns)

The honest scoping is: **capture fideliy + trend visualization + audit
trail**. The clinician remains the interpreter. This is documented in
scope §4.2: *AI constraints: the chatbot is limited to administrative
support, not clinical diagnosis* — the same boundary applies to the
metrics layer.

### 4.4 The rehearsed one-liner if a panelist pushes

> "Our client clinician was explicit: treatment is case-by-case. The field
> calls this the idiographic tradition. The system captures both nomothetic
> instruments — PHQ-9, GAD-7 — *and* idiographic ones — GAS, mood logs,
> attendance, homework adherence. But the system stops at capture. It does
> not interpret, because interpretation is the clinician's role and reducing
> clinical judgment to an algorithm would be both out of scope and ethically
> fraught. That boundary is documented in scope §4.2."

---

## 5. Scope-Honest Disclaimers (Pre-empt the Gotchas)

These are the things the system **doesn't** claim. Saying them proactively
shows maturity.

| What we don't claim | Why |
|---|---|
| We don't predict treatment success | Predicting efficacy would require longitudinal outcome studies against control groups — clinical research, not a clinic-management tool |
| We don't aggregate clinic-wide outcomes | Each patient's PHQ-9 arc is individual; the system isn't computing population-level outcome statistics |
| PHQ-9 / GAD-7 are screening tools, not diagnoses | A score ≥ 10 on PHQ-9 suggests major depressive disorder but the DSM-5 requires a clinician assessment to confirm; the system captures the screen, the clinician interprets |
| Mood logs 1–10 are not a validated psychometric | Honest framing: they capture *between-session variance* the interval-administered instruments miss; the clinician sees them as trend context, not diagnostic |
| We don't claim the demo data reflects real outcomes | The seeded PHQ-9 14→8 arc for Jane is illustrative — shows what the *visualization* looks like when the clinician records improving trajectories |

---

## 6. Demo Data Cheat Sheet — Use This in the Panel

The seeded data tells a coherent clinical story. Use it as visual evidence
the system captures trajectories, not just snapshots.

### 5.1 Jane Doe — Recovery arc (the success case)

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

### 5.2 Emily Watson — Non-responder case (the escalation case)

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

### 5.3 Michael Torres — Improving case (GAD-7 specific)

| Metric | Start | End |
|---|---|---|
| GAD-7 | 12 (Moderate) | 7 (Mild) |
| Mood (16-day arc) | 4 | 7 |
| Goal #1 GAS | `-1` → `+1` | Interrupting partner → no interruptions |

**Demo line:** *"Michael mirrors Jane's pattern, but on the anxiety
instrument — GAD-7 12 to 7 — accompanied by a GAS-rated goal moving from
`-1` to `+1` on the communication goal. The system captures both
nomothetic (GAD-7) and idiographic (GAS) signals side by side."*

---

## 7. Adversarial Q&A Bank

### 6.1 The "efficacy" angle

| Q | A |
|---|---|
| "How do you know the system improves patient outcomes?" | We don't claim it does. The system claims *capture fidelity* — we record the validated instruments the field already uses, plot the trend, and surface non-responders for clinical escalation. Efficacy is measured by clinical trials, not by clinic-management software. |
| "So what's the system's value?" | Centralization, trend visualization, and auditable capture. Without the system, a clinician sees PHQ-9 scores in scattered chart notes. With it, the trajectory is plotted alongside mood logs, attendance, GAS ratings, and assignment adherence on one screen — `patients/{p}/progress`. |
| "What's the comparison with [competitor]?" | Out of scope for this defense; the comparison would be with manual paper-based clinic workflows. The scope doc explicitly lists Manual Scheduling, Communication Gaps, Data Fragmentation, Administrative Overload, and Workflow Inefficiency as the problems. The system addresses all five. |

### 6.2 The "instruments" angle

| Q | A |
|---|---|
| "Is PHQ-9 actually a real metric?" | Yes. Kroenke, Spitzer & Williams (2001), validated against the Structured Clinical Interview for DSM (SCID). Cut-points 5/10/15/20 map to mild/moderate/moderately severe/severe. It's the standard depression screen used in primary care worldwide. |
| "Why GAS instead of just PHQ-9?" | PHQ-9 measures *depression severity* — a nomothetic measurement comparable across patients. GAS captures *individual goal progress* — an idiographic measurement unique to the patient's therapy goals. Both are clinically published and complementary. Real therapy tracks both. |
| "Is a 1–10 mood log really meaningful?" | Honest answer: it's *not* a validated psychometric in the strict sense. Its value is capturing *between-session variance* — PHQ-9 is administered every 1–2 weeks; mood logs are daily. Clinically, both matter. The system presents mood logs as trend context, not as a diagnostic score. |
| "Where does the system stop being clinical?" | At interpretation. We render the PHQ-9 questions, capture the responses, sum the score, plot the trend. We do not interpret what a PHQ-9 of 14 means for a specific patient — that's the clinician's job. Same boundary the scope doc draws in §4.2 ("AI constraints: chatbot is limited to administrative support, not clinical diagnosis"). |

### 6.3 The "what about engagement" angle

| Q | A |
|---|---|
| "Don't no-shows matter?" | Yes — captured. The `Appointment` model has a `no_show` status; the scheduler runs `MarkOverdueNoShows` at 02:00 daily, auto-transitioning any approved appointment past its time + grace window to `no_show`. No-show rate is a published predictor of treatment dropout. |
| "What about homework?" | Captured. Each `Assignment` has a `Submission` with a `status` (`submitted` / `reviewed`). Homework adherence is one of the strongest published predictors of CBT outcomes — meta-analyses by Kazantzis and colleagues. The clinician sees submission rate per patient. |
| "Why include a chatbot if it can't diagnose?" | Scope-honest: chatbot handles *administrative* inquiries (clinic hours, how to book, when appointments are) and light supportive conversation. It cannot and does not attempt clinical interpretation. This is the documented scope limitation in §4.2. The crisis path, however, does include verbatim PH hotlines — that's harm reduction, not therapy. |

### 6.4 The "aggregation" angle

| Q | A |
|---|---|
| "Can the system compute clinic-wide outcomes?" | Not currently. The progress view is per-patient. Aggregating across a clinician's caseload, computing effect sizes, or benchmarking against published norms would be a research-grade feature — documented as future work. The system captures the granular data such analytics would require. |
| "What about anonymized research export?" | Out of scope. It would also raise IRB / data-governance concerns that the `AUDIT_FOLLOWUPS.md` #34 HIPAA-eligibility conversation already covers. |

### 6.5 The "case-by-case" angle (the clinician's stated posture)

| Q | A |
|---|---|
| "Your clinician says treatment is case-by-case. Then why capture metrics at all?" | Because case-by-case doesn't mean *unmeasured* — it means *interpreted individually*. The client clinician's posture is the idiographic tradition in clinical psychology: each patient's trajectory is interpreted against their own baseline, not population norms. The system captures both nomothetic instruments (PHQ-9, GAD-7) and idiographic ones (GAS, mood logs, attendance, homework) — the clinician supplies the case-specific interpretation. The metrics *inform*; they don't *decide*. |
| "Doesn't case-by-case mean your aggregate metrics are useless?" | Yes — and we don't compute aggregate metrics. We deliberately don't compute clinic-wide outcome statistics (see §6.4). The progress view is per-patient, not per-clinic. The system is scoped to support individual case formulation, not population research. |
| "Then what's the system's value to a case-by-case clinician?" | Three things. First, capture fidelity — the PHQ-9 score recorded today will be the same PHQ-9 score the clinician reviews in 3 weeks. Second, trend visualization — plotting Jane's PHQ-9 14→8 next to her mood 4→8 next to her GAS `-1→0` makes the trajectory visible at a glance, which case-by-case discussion in supervision often re-derives from memory. Third, audit trail — every assessment, every clinician note, every assignment submission is timestamped and reviewable. Case-by-case doesn't mean no records. |
| "If the clinician doesn't rely on the metrics, won't they just ignore them?" | That's the clinician's call, and it's the right posture. A PHQ-9 score doesn't replace a clinical interview — the field has known this since the instrument was validated. The system surfaces the data; the clinician weighs it alongside case formulation, patient presentation, and clinical judgment. The system does not nudge, alert-threshold, or auto-escalate — those would be inappropriate scope. |
| "Did you consider a recommendation engine?" | Considered and explicitly out of scope. See `AUDIT_FOLLOWUPS.md` #34 — the system is built to a healthcare-adjacent posture, not a clinical decision support system. Recommendation engines in mental health have published bias concerns; the client clinician's case-by-case framing is a deliberate ethical safeguard against algorithmic reduction. |

---

## 8. Two Visual Artifacts to Point At in the Demo

During the panel, point at real screens:

### 7.1 The progress view

URL: `/patients/{jane}/progress` (signed in as Dr. Chen)

This is the strongest single artifact. On one page:
- Attendance count (no-show rate visible by absence)
- PHQ-9 + GAD-7 scores with severity bands
- Mood log trend line
- Goals with GAS ratings tied to specific appointments

**Demo line:** *"This is what a case conference looks like in the system.
Every metric on this page is a number the panel asked whether mental
health has. PHQ-9 14 → 8. Mood 4 → 8. GAS progress on two goals. All six
metrics from §2 visible together."*

### 7.2 The notification audit log

URL: `/notifications/logs` (signed in as admin)

Every notification dispatched — appointment_approved,
assignment_created, assessment_assigned, message_received, patient_request —
is on this log, with timestamps. This is the procedural record that
complements the clinical metrics: *did the patient receive the intervention
the clinician ordered?*

---

## 9. The One-Sentence Closer

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

## 10. See Also

- `DEFENSE_READINESS.md` §5 — rehearsed answer for the "real-time" claim
- `DEMO_SCRIPT.md` §4.6 — assessment submit flow; §3.9 — progress view walkthrough
- `AUDIT_FOLLOWUPS.md` #34 — HIPAA-eligibility boundary (relevant if panel
  asks about research-grade aggregation that would require IRB governance)
