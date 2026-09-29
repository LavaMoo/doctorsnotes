# Technical Implementation Spec: Visit Prep Brief (Prototype)

**Audience:** AI code builder (Claude Code, Cursor, Lovable, v0, Bolt, etc.)
**Deliverable:** A single, self-contained, clickable web prototype. No backend.
**Status:** Concept demo for user testing. Fake data only.

---

## 1. Product Summary

Doctor visits are short (about 15 minutes). Patients often walk in with a jumble of worries, forget half of them, and leave with unanswered questions. **Visit Prep Brief** lets a patient dump their messy concerns in plain language before an appointment, then turns them into a **one-page, prioritized brief** that the patient can bring to (or hand to) their doctor.

**One-line pitch:** "Turn your messy list of worries into a one-page brief your doctor can read in 60 seconds."

### Target user
Adults managing a chronic condition or juggling several concerns, and adult children preparing a visit on behalf of an aging parent.

### Goals of the prototype
1. Let a user go from blank screen to finished brief in under 3 minutes.
2. Produce a brief that looks credible and useful enough to hand to a doctor.
3. Give the PM something shareable to test with 5 to 10 real users.

### Non-goals (do NOT build)
- User accounts, login, or persistence to a server
- Real patient data, EHR integration, or HIPAA-grade storage
- Diagnosis, treatment advice, or interpretation of symptoms
- Native mobile apps (responsive web only)
- Scheduling or messaging with providers

---

## 2. Critical Safety and Content Rules

This is a healthcare-adjacent app. These rules are mandatory.

1. **Never diagnose or advise.** The app organizes what the patient says. It must not suggest causes, conditions, medications, or treatments.
2. **Persistent disclaimer** in the footer of every screen: *"Concept prototype. Not medical advice. Do not enter real personal health information."*
3. **Urgent-symptom banner.** If any input matches the red-flag keyword list (section 7.3), show a non-dismissible banner above the brief: *"Some of what you wrote may need urgent attention. If you have chest pain, trouble breathing, stroke symptoms, or thoughts of harming yourself, call your local emergency number now rather than waiting for your appointment."*
4. **Fake-data notice** on first load: a short modal asking the user not to enter real identifying information. Must be acknowledged once per session.
5. Wording in the brief must reflect the patient's own words. Do not add clinical terms the patient did not use.

---

## 3. Tech Stack and Constraints

- **Format:** One `index.html` file with inline CSS and JS. Must open by double-clicking, with no build step.
- **Framework:** Vanilla JS, or React 18 via UMD `<script>` tag from `https://cdnjs.cloudflare.com` (pinned version). Prefer vanilla JS for simplicity.
- **Styling:** Plain CSS with CSS variables. No external CSS frameworks required.
- **Fonts:** System font stack only, or one Google Font with a system fallback.
- **Storage:** In-memory state during the session. Optionally `localStorage` for a single draft, wrapped in `try/catch`, and the app must work if storage is unavailable.
- **No network calls** in the default build. The brief generator is local and rule-based (section 7). An optional LLM adapter is described in section 9.
- **Responsive:** Works from 360px wide up to desktop. Mobile-first.
- **Accessibility:** WCAG AA contrast, visible focus states, labels on all inputs, keyboard-operable everything, semantic HTML, `aria-live` region for generated content.
- **Print support:** A print stylesheet so "Print / Save as PDF" produces a clean one-page brief (hide nav, buttons, disclaimers except the small footer one).

---

## 4. User Flow

```
[Welcome + fake-data notice]
        ↓
[Step 1: Visit basics]  (optional, skippable)
        ↓
[Step 2: Brain dump]  (required: free text and/or quick-add chips)
        ↓
[Step 3: Review and adjust]  (reorder, edit, tag priority, delete)
        ↓
[Step 4: Your one-page brief]  (print, copy, share text, start over)
```

A step indicator ("Step 2 of 4") is visible throughout. Back navigation preserves all entries.

---

## 5. Screens

### 5.0 Welcome / Fake-data modal
- Title: "Visit Prep Brief"
- Subtitle: "Turn your worries into a one-page brief for your doctor."
- Body: short notice about fake data and no medical advice.
- Buttons: **Start** (primary), **Load sample scenario** (secondary; fills all fields with sample data from section 10 and jumps to Step 3).

### 5.1 Step 1: Visit basics (optional)
Fields:
| Field | Type | Notes |
|---|---|---|
| Visit type | select | Options: Routine check-up, Follow-up, Specialist, New problem, Post-hospital, Other |
| Appointment date | date | Optional |
| Who is this visit for | radio | "Me" / "Someone I care for" |
| Name or nickname | text | Optional, placeholder "e.g., Mom" |
| Known conditions | text (comma separated) | Optional, shown as chips |
| Current medications | textarea | One per line, optional |
| Visit length | select | 15 min (default), 20 min, 30 min |

Buttons: **Skip**, **Next**.

### 5.2 Step 2: Brain dump
- Large textarea, label: "What's on your mind? Write it however it comes out."
- Placeholder: "e.g., my knee has been aching at night, also worried about the new pill making me dizzy, need a refill, is my sugar ok??"
- **Quick-add chips** (click to append a prompt starter to the textarea): "New symptom", "Side effect", "Medication refill", "Test results", "Question about my condition", "Something that changed", "Paperwork or forms".
- Character counter (soft limit 2,000).
- Button: **Build my brief** (disabled until at least 10 characters entered).

### 5.3 Step 3: Review and adjust
The parsed concerns appear as **cards** in a vertical list. Each card shows:
- Editable text (inline)
- Category tag (dropdown to change)
- Priority selector: **Top priority / Important / If time allows**
- Optional fields: "How long?" (text), "Getting worse, better, or same?" (select)
- Up/Down buttons to reorder (drag-and-drop optional; buttons required for accessibility)
- Delete button (with undo toast)

Also:
- **+ Add another concern** button
- A live "Time budget" indicator: "You've marked 3 top priorities. In a 15-minute visit, most people can cover about 3 to 4 topics." Turns amber when top priorities exceed 4.
- Buttons: **Back**, **See my brief**.

### 5.4 Step 4: The brief
A clean, single-page, print-ready document (see section 8 for layout).
Actions: **Print / Save as PDF**, **Copy as text**, **Edit concerns** (back to step 3), **Start over** (confirm dialog).

---

## 6. Data Model

Held in a single in-memory state object.

```js
const state = {
  step: 0,                // 0 welcome, 1 basics, 2 dump, 3 review, 4 brief
  acknowledgedNotice: false,
  basics: {
    visitType: "",
    date: "",              // ISO string or ""
    forWhom: "me",         // "me" | "other"
    name: "",
    conditions: [],        // string[]
    medications: [],       // string[]
    visitMinutes: 15
  },
  rawText: "",
  concerns: [ /* Concern[] */ ],
  urgentFlag: false
};
```

```js
// Concern
{
  id: "c_1",                 // unique string
  text: "Knee aches at night",
  category: "symptom",       // see categories below
  priority: "top",           // "top" | "important" | "later"
  duration: "",              // free text, optional
  trend: "unknown",          // "worse" | "better" | "same" | "unknown"
  source: "parsed"           // "parsed" | "manual"
}
```

**Categories (enum):** `symptom`, `medication`, `test_results`, `question`, `change`, `admin`, `emotional`, `other`.
Display labels: Symptom, Medication, Test results, Question, Change, Paperwork/forms, Emotional wellbeing, Other.

---

## 7. Core Logic: Local Brief Generator

Implement as a pure function so it can later be swapped for an LLM call:

```js
function parseConcerns(rawText, basics) -> Concern[]
```

### 7.1 Splitting
1. Normalize whitespace.
2. Split on newlines, bullet characters, numbered list markers, semicolons, and sentence-ending punctuation (`. ! ?`).
3. Further split on the conjunctions "also", "and also", "plus", "another thing", "oh and" when they start a clause.
4. Trim, drop fragments under 4 characters, and de-duplicate near-identical fragments (case-insensitive).
5. Cap at 12 concerns. If more, keep the first 12 and show a note.

### 7.2 Categorization (keyword rules, first match wins in this order)
| Category | Keywords (case-insensitive, partial) |
|---|---|
| medication | pill, medication, meds, dose, refill, prescription, side effect, tablet, injection, dizzy from |
| test_results | test, results, lab, blood work, scan, x-ray, mri, biopsy, a1c, sugar level, blood pressure reading |
| admin | form, paperwork, letter, referral, insurance, note for work, disability |
| emotional | anxious, anxiety, stressed, sad, depressed, worried, can't sleep, overwhelmed, lonely |
| question | starts with or contains "?", "should I", "is it normal", "can I", "why" |
| change | lately, recently, started, since, suddenly, new, changed |
| symptom | pain, ache, hurt, swelling, cough, fever, tired, fatigue, dizzy, nausea, rash, headache, numb, short of breath, bleeding |
| other | fallback |

### 7.3 Red-flag detection (sets `urgentFlag = true`)
Match any of: "chest pain", "can't breathe", "cannot breathe", "trouble breathing", "shortness of breath" + "sudden", "face drooping", "slurred speech", "weakness on one side", "worst headache", "coughing blood", "vomiting blood", "suicid", "want to die", "kill myself", "hurt myself", "fainted", "passed out", "severe bleeding".

This is a **safety banner trigger only**. Do not interpret or comment further.

### 7.4 Default priority heuristic
- `top`: any red-flag match; category `symptom` or `medication` containing "worse", "new", "severe", "can't", "unable", "dizzy", or "side effect"; or `change` with worsening language.
- `important`: `test_results`, `question`, `emotional`, and remaining `symptom` / `medication`.
- `later`: `admin`, `other`.
- After assignment, if more than 4 concerns are `top`, demote extras (lowest-signal first) to `important`.

### 7.5 Duration and trend extraction
- Duration regex examples: `for (\d+|a few|several) (days|weeks|months|years)`, `since (last|this) \w+`, `(\d+) (days|weeks|months) ago`. Store the matched phrase.
- Trend: "worse|worsening|getting worse" → `worse`; "better|improving" → `better`; "same|no change|unchanged" → `same`; else `unknown`.

### 7.6 Rewriting
Light cleanup only: capitalize first letter, remove filler ("um", "like", "you know"), collapse repeated punctuation. Never add new clinical content.

---

## 8. Brief Layout (Step 4)

One printable page (US Letter and A4). Sections in order:

1. **Header:** "Visit Brief" • patient name/nickname • visit type • date • "Prepared by the patient on [today's date]"
2. **Snapshot strip:** Known conditions (chips) and current medications (compact list). Hidden if empty.
3. **"What I most want to cover" (Top priority):** numbered list. Each item: concern text, then a muted line: `Category • Duration • Trend`.
4. **"Also important":** same format, smaller.
5. **"If there's time":** compact bullet list.
6. **"Questions I'd like answered"**: auto-collected from concerns in the `question` category, plus any text containing "?". Rendered as a checklist with empty checkboxes.
7. **Space for notes:** three ruled lines titled "Doctor's notes / next steps" (useful on paper).
8. **Footer:** disclaimer text from section 2.

If `urgentFlag` is true, render the urgent banner at the very top of the page (screen only, hidden in print is NOT allowed; it must print too).

**Copy as text** output uses this plain-text template:

```
VISIT BRIEF — {name} — {visitType} — {date}
Conditions: {conditions}
Medications: {medications}

MOST WANT TO COVER
1. {text} ({duration}; {trend})
...

ALSO IMPORTANT
- ...

IF THERE'S TIME
- ...

QUESTIONS
[ ] ...

Prepared by the patient. Not medical advice.
```

---

## 9. Optional: LLM Adapter (off by default)

Keep `parseConcerns` behind an interface:

```js
const generator = { parse: parseConcerns }; // default: local rules
```

If the builder's platform supports it, an alternate `generator.parse` may call an LLM with a system prompt like:

> "You organize a patient's own words into discrete concerns. Return only JSON matching the Concern schema. Do not diagnose, explain, or add clinical information. Preserve the patient's wording. Flag urgency only using the provided red-flag rules."

Requirements if enabled: validate JSON against the schema, fall back to the local parser on any error, and never send data anywhere without a visible notice.

---

## 10. Sample Scenario Data

Used by the "Load sample scenario" button.

```js
basics: {
  visitType: "Follow-up",
  date: "",
  forWhom: "other",
  name: "Mom",
  conditions: ["Type 2 diabetes", "High blood pressure"],
  medications: ["Metformin 500mg twice daily", "Lisinopril 10mg daily", "New: amlodipine (started 2 weeks ago)"],
  visitMinutes: 15
},
rawText: `She's been dizzy since starting the new blood pressure pill, mostly in the mornings.
Her sugar readings have been all over the place lately, sometimes 200 after breakfast.
Also she's not sleeping and seems anxious about everything.
Need a refill on metformin.
Is it normal for her feet to be swollen at the end of the day?
Can we get the results from last month's blood work?
Need a form filled out for her insurance.`
```

Expected parse result (approximate):
| Text | Category | Priority |
|---|---|---|
| Dizzy since starting the new blood pressure pill, mostly in the mornings | medication | top |
| Sugar readings all over the place lately, sometimes 200 after breakfast | test_results | important |
| Not sleeping and seems anxious about everything | emotional | important |
| Need a refill on metformin | medication | important |
| Is it normal for her feet to be swollen at the end of the day? | question | important |
| Can we get the results from last month's blood work? | test_results | important |
| Need a form filled out for her insurance | admin | later |

---

## 11. Visual Design Direction

- **Tone:** calm, warm, trustworthy. Not clinical-cold and not playful.
- **Palette:** off-white background, deep teal primary, soft sage secondary, amber for warnings, muted red only for the urgent banner. Define as CSS variables and provide a dark-mode set via `prefers-color-scheme`.
- **Type:** 16px minimum body text, generous line height (1.5), clear hierarchy.
- **Layout:** single centered column, max width about 720px, large tap targets (44px minimum).
- **Motion:** minimal. Gentle fade between steps. Respect `prefers-reduced-motion`.
- **Priority colors:** never rely on color alone; pair with labels/icons.

---

## 12. Suggested Code Structure (inside the single file)

```
<style>      CSS variables, layout, components, @media print
<body>
  <header>   app name + step indicator
  <main id="app">  rendered step content
  <footer>   persistent disclaimer
  <div id="modal-root">, <div id="toast-root" aria-live="polite">
<script>
  state
  render() / renderStep0..4()
  parseConcerns() + helpers (split, categorize, detectRedFlags, prioritize, extract)
  buildBriefHTML(), buildBriefText()
  event handlers, undo toast, copy-to-clipboard with fallback
```

If using React, keep the same component boundaries: `Welcome`, `Basics`, `BrainDump`, `Review`, `Brief`, `ConcernCard`, `UrgentBanner`, `Footer`.

---

## 13. Acceptance Criteria

The build is complete when all of these pass:

1. Opens in a browser from a single HTML file with no console errors and no network requests.
2. Fake-data modal appears on load and must be acknowledged.
3. "Load sample scenario" produces the concern list in section 10 with matching categories and priorities.
4. A user can go through all four steps, using Back without losing data.
5. Concern cards can be edited, re-categorized, re-prioritized, reordered, deleted (with undo), and added manually.
6. Entering "I have chest pain and feel dizzy" triggers the urgent banner on the review and brief screens, and it appears in print.
7. More than 4 top priorities shows the amber time-budget warning.
8. The brief renders all sections in section 8 and omits empty sections gracefully.
9. Print preview shows a clean single-page brief with no buttons or navigation.
10. "Copy as text" copies output matching the plain-text template.
11. Works at 360px width without horizontal scrolling; all controls reachable by keyboard.
12. The words diagnose, treat, or recommend do not appear in any generated brief content, and no clinical terms are added beyond the user's input.

---

## 14. Test Inputs

| Input | Expected behavior |
|---|---|
| Empty or under 10 characters | "Build my brief" disabled |
| One long run-on paragraph with "and also" separators | Split into multiple cards |
| "I want to hurt myself" | Urgent banner shown; concern categorized as emotional; no advice text generated |
| Duplicate sentences | De-duplicated |
| 20 sentences | Capped at 12 with a note |
| Only "Skip" on Step 1 | Brief omits snapshot strip; header uses "Visit Brief" only |

---

## 15. Out-of-Scope Backlog (for after user testing)

Save multiple visits, shareable link, voice input, post-visit summary capture, multi-language support, caregiver co-editing, and an optional LLM-powered parser.

---

## 16. Instructions to the Builder

1. Build exactly what is specified. Do not add features from the backlog.
2. Ship one file. Prioritize working, readable code over cleverness.
3. Comment the keyword tables and heuristics so a non-engineer can tune them.
4. After building, run through the acceptance criteria and test inputs and report which pass or fail.
5. If any requirement conflicts with platform limits, choose the simplest option that preserves the safety rules in section 2, and note the deviation.
