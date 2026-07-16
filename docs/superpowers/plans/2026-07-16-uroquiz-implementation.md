# UroQuiz Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single self-contained HTML/CSS/JS quiz app testing EAU guideline knowledge, hosted as a Claude Artifact, with topic filtering, Study/Exam modes, results review, and localStorage-backed stats.

**Architecture:** One `index.html` file with inline `<style>` and `<script>` (no build tooling, no framework, no external requests — required for Artifact hosting). A `QUESTIONS` array holds all content, grouped by topic/subtopic. Plain JS render functions act as screens, swapped via a `state.screen` variable and a single `render()` dispatcher. `localStorage` is read/written directly for stats and missed-question tracking.

**Tech Stack:** Vanilla HTML/CSS/JavaScript (ES2020+), `localStorage` Web API. No npm, no bundler, no external libraries.

Per the spec (`docs/superpowers/specs/2026-07-16-uroquiz-design.md`), verification for this project is manual (load the Artifact, click through screens, check `localStorage`) rather than automated unit tests — there is no Node/build environment on the user's side, and the app is a single static file.

---

## File Structure

- Create: `index.html` — the entire app (the file published to the Claude Artifact)

Everything lives in this one file: `<head>` has inline `<style>`, `<body>` has a single `<div id="app">` root, and a `<script>` block contains `QUESTIONS`, state, render functions, and event wiring.

---

### Task 1: HTML/CSS shell + state skeleton

**Files:**
- Create: `index.html`

- [ ] **Step 1: Write the base HTML document with theme-aware CSS**

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>UroQuiz — EAU Guideline MCQ Bank</title>
<style>
  :root {
    --bg: #ffffff;
    --fg: #1a1a1a;
    --card-bg: #f5f5f7;
    --accent: #0a66c2;
    --correct: #1a7f37;
    --incorrect: #c0392b;
    --border: #d9d9df;
  }
  @media (prefers-color-scheme: dark) {
    :root {
      --bg: #14151a;
      --fg: #eaeaef;
      --card-bg: #1f2027;
      --accent: #5aa9f8;
      --correct: #4caf6d;
      --incorrect: #e5695f;
      --border: #33343c;
    }
  }
  :root[data-theme="light"] {
    --bg: #ffffff; --fg: #1a1a1a; --card-bg: #f5f5f7;
    --accent: #0a66c2; --correct: #1a7f37; --incorrect: #c0392b; --border: #d9d9df;
  }
  :root[data-theme="dark"] {
    --bg: #14151a; --fg: #eaeaef; --card-bg: #1f2027;
    --accent: #5aa9f8; --correct: #4caf6d; --incorrect: #e5695f; --border: #33343c;
  }
  * { box-sizing: border-box; }
  body {
    margin: 0; background: var(--bg); color: var(--fg);
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
  }
  #app { max-width: 640px; margin: 0 auto; padding: 1.25rem; }
  .card {
    background: var(--card-bg); border: 1px solid var(--border);
    border-radius: 12px; padding: 1rem; margin-bottom: 0.75rem;
  }
  button {
    font: inherit; padding: 0.6rem 1rem; border-radius: 8px;
    border: 1px solid var(--border); background: var(--accent); color: white;
    cursor: pointer;
  }
  button.secondary { background: transparent; color: var(--fg); }
  label.option {
    display: block; padding: 0.6rem 0.8rem; margin: 0.4rem 0;
    border: 1px solid var(--border); border-radius: 8px; cursor: pointer;
  }
  label.option.correct { border-color: var(--correct); background: color-mix(in srgb, var(--correct) 15%, var(--card-bg)); }
  label.option.incorrect { border-color: var(--incorrect); background: color-mix(in srgb, var(--incorrect) 15%, var(--card-bg)); }
  .progress { font-size: 0.85rem; opacity: 0.7; margin-bottom: 0.5rem; }
  .explanation { margin-top: 0.6rem; font-size: 0.9rem; opacity: 0.85; }
  .topic-row { display: flex; align-items: center; gap: 0.5rem; padding: 0.3rem 0; }
</style>
</head>
<body>
<div id="app"></div>
<script>
const STORAGE_KEY = "uroquiz-progress-v1";

const state = {
  screen: "home", // home | quiz | results | stats
  selectedTopics: [],
  mode: "study",  // study | exam
  sessionQuestions: [],
  currentIndex: 0,
  answers: [],    // { questionId, chosenIndex, correct }
};

function render() {
  const app = document.getElementById("app");
  app.innerHTML = "";
  if (state.screen === "home") app.appendChild(renderHome());
  else if (state.screen === "quiz") app.appendChild(renderQuiz());
  else if (state.screen === "results") app.appendChild(renderResults());
  else if (state.screen === "stats") app.appendChild(renderStats());
}

function renderHome() {
  const div = document.createElement("div");
  div.textContent = "Home screen placeholder";
  return div;
}
function renderQuiz() {
  const div = document.createElement("div");
  div.textContent = "Quiz screen placeholder";
  return div;
}
function renderResults() {
  const div = document.createElement("div");
  div.textContent = "Results screen placeholder";
  return div;
}
function renderStats() {
  const div = document.createElement("div");
  div.textContent = "Stats screen placeholder";
  return div;
}

render();
</script>
</body>
</html>
```

- [ ] **Step 2: Manual verification**

Open `index.html` directly in a browser (double-click or `open index.html` on macOS).
Expected: a blank page with "Home screen placeholder" text, correct dark/light background depending on system theme.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Add HTML/CSS shell and render dispatcher skeleton"
```

---

### Task 2: Question data schema + starter oncology batch

**Files:**
- Modify: `index.html` (add `QUESTIONS` array before `state`)

- [ ] **Step 1: Add the `QUESTIONS` array with oncology questions**

Insert above `const state = {`:

```html
<script>
const QUESTIONS = [
  // ---- ONCOLOGY: prostate ----
  {
    id: "onco-prostate-001",
    topic: "oncology", subtopic: "prostate",
    question: "According to EAU risk stratification for localised prostate cancer, which combination defines 'low risk' disease?",
    options: [
      "PSA <10 ng/mL AND ISUP grade 1 AND cT1-T2a",
      "PSA <20 ng/mL AND ISUP grade 2 AND cT2b",
      "PSA <10 ng/mL AND ISUP grade 3 AND cT1c",
      "PSA >20 ng/mL AND ISUP grade 1 AND cT2a"
    ],
    correctIndex: 0,
    explanation: "EAU low-risk localised prostate cancer requires all three: PSA <10 ng/mL, ISUP grade 1 (Gleason ≤6), and clinical stage T1-T2a. Any higher value on one axis moves it to intermediate or high risk."
  },
  {
    id: "onco-prostate-002",
    topic: "oncology", subtopic: "prostate",
    question: "Active surveillance is most appropriate for which prostate cancer risk group per EAU guidelines?",
    options: ["High risk", "Low risk", "Metastatic", "Locally advanced (T3-T4)"],
    correctIndex: 1,
    explanation: "Active surveillance is the preferred management option for low-risk localised prostate cancer to avoid overtreatment of indolent disease."
  },
  {
    id: "onco-prostate-003",
    topic: "oncology", subtopic: "prostate",
    question: "In multiparametric MRI reporting for prostate cancer, a PI-RADS score of 4 or 5 indicates:",
    options: [
      "No clinically significant cancer likely",
      "Clinically significant cancer is highly likely, biopsy recommended",
      "Indeterminate, repeat MRI in 6 months",
      "Only relevant for post-treatment surveillance"
    ],
    correctIndex: 1,
    explanation: "PI-RADS 4-5 indicates high probability of clinically significant prostate cancer and should prompt targeted biopsy."
  },
  // ---- ONCOLOGY: bladder ----
  {
    id: "onco-bladder-001",
    topic: "oncology", subtopic: "bladder",
    question: "For high-risk non-muscle-invasive bladder cancer (NMIBC), what is the recommended intravesical therapy per EAU guidelines?",
    options: [
      "Intravesical chemotherapy single instillation only",
      "BCG induction plus maintenance for 1-3 years",
      "Immediate radical cystectomy in all cases",
      "Surveillance only, no adjuvant therapy"
    ],
    correctIndex: 1,
    explanation: "High-risk NMIBC is treated with BCG induction (6 weekly instillations) followed by a maintenance schedule, typically for 1-3 years depending on risk substratification."
  },
  {
    id: "onco-bladder-002",
    topic: "oncology", subtopic: "bladder",
    question: "Which T stage defines muscle-invasive bladder cancer (MIBC)?",
    options: ["Ta", "T1", "T2 or higher", "Tis (carcinoma in situ)"],
    correctIndex: 2,
    explanation: "MIBC is defined as tumour invasion into or beyond the muscularis propria, i.e. ≥T2. Ta, T1, and Tis are all non-muscle-invasive."
  },
  {
    id: "onco-bladder-003",
    topic: "oncology", subtopic: "bladder",
    question: "The standard treatment for MIBC in a fit patient without metastases is:",
    options: [
      "Radical cystectomy with pelvic lymph node dissection, ideally after neoadjuvant chemotherapy",
      "BCG maintenance therapy",
      "Transurethral resection alone",
      "Active surveillance"
    ],
    correctIndex: 0,
    explanation: "Radical cystectomy with lymphadenectomy (preceded by cisplatin-based neoadjuvant chemotherapy in eligible patients) is standard of care for localised MIBC."
  },
  // ---- ONCOLOGY: renal cell carcinoma ----
  {
    id: "onco-rcc-001",
    topic: "oncology", subtopic: "rcc",
    question: "For a small renal mass (<4 cm, T1a) in a patient with normal renal function, EAU guidelines favour:",
    options: [
      "Radical nephrectomy as first-line",
      "Partial nephrectomy (nephron-sparing surgery) where technically feasible",
      "Systemic therapy without surgery",
      "Radiotherapy alone"
    ],
    correctIndex: 1,
    explanation: "Partial nephrectomy is preferred for T1a renal masses to preserve renal function, with oncological outcomes equivalent to radical nephrectomy."
  },
  {
    id: "onco-rcc-002",
    topic: "oncology", subtopic: "rcc",
    question: "Which is the most common histological subtype of renal cell carcinoma?",
    options: ["Papillary RCC", "Chromophobe RCC", "Clear cell RCC", "Collecting duct RCC"],
    correctIndex: 2,
    explanation: "Clear cell RCC accounts for roughly 70-80% of RCC cases and is the most common subtype."
  },
  // ---- ONCOLOGY: upper tract urothelial cancer ----
  {
    id: "onco-utuc-001",
    topic: "oncology", subtopic: "utuc",
    question: "The standard surgical treatment for high-risk upper tract urothelial carcinoma (UTUC) is:",
    options: [
      "Radical nephroureterectomy with bladder cuff excision",
      "Kidney-sparing endoscopic resection",
      "Partial nephrectomy",
      "Active surveillance"
    ],
    correctIndex: 0,
    explanation: "Radical nephroureterectomy with excision of the bladder cuff is the standard of care for high-risk UTUC, given the risk of tumour seeding along the ureter."
  },
  {
    id: "onco-utuc-002",
    topic: "oncology", subtopic: "utuc",
    question: "Kidney-sparing management of UTUC is generally considered for:",
    options: [
      "Any high-grade tumour regardless of size",
      "Low-risk, low-grade tumours, especially in a solitary kidney or with imperative indications",
      "Bilateral high-grade disease only",
      "Never appropriate in UTUC"
    ],
    correctIndex: 1,
    explanation: "Kidney-sparing approaches (endoscopic ablation/resection) are appropriate for low-risk disease, or when preserving renal function is imperative (e.g. solitary kidney, bilateral disease, poor renal function)."
  },
  // ---- ONCOLOGY: testicular ----
  {
    id: "onco-testis-001",
    topic: "oncology", subtopic: "testis",
    question: "The first step in management of a suspicious testicular mass is:",
    options: [
      "Percutaneous biopsy",
      "Radical inguinal orchiectomy",
      "Trans-scrotal biopsy",
      "Empirical chemotherapy"
    ],
    correctIndex: 1,
    explanation: "Radical inguinal orchiectomy is the standard first step for a suspicious testicular mass — trans-scrotal biopsy risks local seeding and disrupts lymphatic drainage planning."
  },
  {
    id: "onco-testis-002",
    topic: "oncology", subtopic: "testis",
    question: "Which tumour marker is characteristically elevated in choriocarcinoma but NOT in pure seminoma?",
    options: ["AFP", "Beta-hCG", "LDH", "PSA"],
    correctIndex: 1,
    explanation: "Beta-hCG can be mildly elevated in seminoma (from syncytiotrophoblastic cells) but marked elevation is characteristic of choriocarcinoma. AFP elevation excludes pure seminoma entirely."
  },
  // ---- ONCOLOGY: penile ----
  {
    id: "onco-penile-001",
    topic: "oncology", subtopic: "penile",
    question: "For a small, organ-confined penile carcinoma, EAU guidelines favour which approach when feasible?",
    options: [
      "Total penectomy as first-line for all lesions",
      "Organ-sparing surgery (e.g. wide local excision, glansectomy) to preserve function",
      "Radiotherapy is always contraindicated",
      "Chemotherapy alone without surgery"
    ],
    correctIndex: 1,
    explanation: "Organ-sparing surgical techniques are preferred when oncologically feasible to preserve sexual and urinary function, reserving penectomy for more extensive disease."
  }
];
</script>
```

- [ ] **Step 2: Manual verification**

Open `index.html` in a browser, then open the browser console and run:
```js
QUESTIONS.length // expect 13
QUESTIONS.filter(q => q.topic === "oncology").length // expect 13
new Set(QUESTIONS.map(q => q.id)).size === QUESTIONS.length // expect true (no duplicate ids)
```
Expected: no console errors, all three checks pass.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Add question schema and oncology starter batch (13 questions)"
```

---

### Task 3: Non-oncology and infections/other question batches

**Files:**
- Modify: `index.html` (append to `QUESTIONS`)

- [ ] **Step 1: Append non-oncology/functional questions**

Add before the closing `];` of `QUESTIONS`:

```js
  // ---- NON-ONCOLOGY: urolithiasis ----
  {
    id: "stones-001",
    topic: "non-oncology", subtopic: "stones",
    question: "For a 6mm proximal ureteric stone causing obstruction without sepsis, first-line management per EAU guidelines is typically:",
    options: [
      "Immediate open surgery",
      "Trial of medical expulsive therapy (e.g. alpha-blocker) with observation, given reasonable spontaneous passage likelihood",
      "Emergency nephrectomy",
      "Long-term indwelling catheter"
    ],
    correctIndex: 1,
    explanation: "Stones ≤10mm without signs of sepsis or significant obstruction-related risk can be managed with observation and medical expulsive therapy, as spontaneous passage rates are reasonable at this size."
  },
  {
    id: "stones-002",
    topic: "non-oncology", subtopic: "stones",
    question: "An obstructing ureteric stone WITH signs of infection (fever, pyuria) requires:",
    options: [
      "Watchful waiting with oral antibiotics only",
      "Urgent decompression (stent or nephrostomy) plus antibiotics — this is a urological emergency",
      "Elective shockwave lithotripsy within 6 weeks",
      "Immediate stone extraction without decompression first"
    ],
    correctIndex: 1,
    explanation: "Obstruction with infection is an emergency requiring urgent drainage (ureteric stent or percutaneous nephrostomy) plus antibiotics; definitive stone treatment is deferred until sepsis is controlled."
  },
  {
    id: "stones-003",
    topic: "non-oncology", subtopic: "stones",
    question: "Which imaging modality is recommended as first-line for suspected acute renal colic per EAU guidelines?",
    options: ["Non-contrast CT (CT-KUB)", "Plain abdominal X-ray alone", "MRI", "Retrograde pyelogram"],
    correctIndex: 0,
    explanation: "Non-contrast CT (CT-KUB) is the recommended first-line imaging for suspected renal colic due to high sensitivity and specificity for stones."
  },
  // ---- NON-ONCOLOGY: BPH / male LUTS ----
  {
    id: "bph-001",
    topic: "non-oncology", subtopic: "bph",
    question: "First-line medical therapy for moderate-severe male LUTS due to benign prostatic obstruction is typically:",
    options: ["Alpha-1 blockers", "5-alpha reductase inhibitors alone", "Anticholinergics alone", "Antibiotics"],
    correctIndex: 0,
    explanation: "Alpha-1 blockers provide rapid symptom relief in male LUTS/BPO and are typically first-line; 5-ARIs are added particularly in larger prostates for long-term risk reduction."
  },
  {
    id: "bph-002",
    topic: "non-oncology", subtopic: "bph",
    question: "5-alpha reductase inhibitors are most beneficial in men with LUTS who also have:",
    options: [
      "A small prostate volume (<30mL)",
      "An enlarged prostate volume (typically >40mL) due to reducing prostate size over time",
      "Acute bacterial prostatitis",
      "Normal PSA and small gland"
    ],
    correctIndex: 1,
    explanation: "5-ARIs reduce prostate volume over months and are most beneficial in men with larger glands, reducing risk of progression, acute retention, and need for surgery."
  },
  // ---- NON-ONCOLOGY: incontinence ----
  {
    id: "incontinence-001",
    topic: "non-oncology", subtopic: "incontinence",
    question: "First-line management for uncomplicated stress urinary incontinence in women is:",
    options: [
      "Mid-urethral sling surgery immediately",
      "Pelvic floor muscle training (conservative therapy)",
      "Botulinum toxin injection",
      "Long-term catheterisation"
    ],
    correctIndex: 1,
    explanation: "Conservative management with supervised pelvic floor muscle training is first-line for stress urinary incontinence before considering surgical options."
  },
  {
    id: "incontinence-002",
    topic: "non-oncology", subtopic: "incontinence",
    question: "Antimuscarinic drugs (e.g. solifenacin) are primarily used to treat which type of incontinence?",
    options: ["Stress urinary incontinence", "Urgency urinary incontinence / overactive bladder", "Overflow incontinence", "Functional incontinence only"],
    correctIndex: 1,
    explanation: "Antimuscarinics reduce detrusor overactivity and are a mainstay of pharmacotherapy for urgency incontinence/overactive bladder, not stress incontinence."
  },
  // ---- NON-ONCOLOGY: neuro-urology ----
  {
    id: "neuro-001",
    topic: "non-oncology", subtopic: "neuro-urology",
    question: "In patients with neurogenic lower urinary tract dysfunction (e.g. spinal cord injury), the primary goal of management per EAU guidelines is:",
    options: [
      "Achieving complete continence at any cost",
      "Protecting the upper urinary tract (preserving renal function) as the top priority",
      "Immediate surgical diversion in all patients",
      "Cosmetic outcome"
    ],
    correctIndex: 1,
    explanation: "Upper tract protection (preventing high-pressure storage/voiding that damages the kidneys) is the primary goal in neurogenic bladder management, with continence and quality of life as secondary goals."
  },
  {
    id: "neuro-002",
    topic: "non-oncology", subtopic: "neuro-urology",
    question: "Clean intermittent catheterisation (CIC) is recommended in neurogenic bladder patients primarily to manage:",
    options: [
      "Detrusor overactivity with normal emptying",
      "Incomplete bladder emptying / urinary retention",
      "Stress incontinence only",
      "Erectile dysfunction"
    ],
    correctIndex: 1,
    explanation: "CIC is used when there is incomplete bladder emptying to reduce residual urine, prevent overdistension injury, and lower infection/upper tract risk."
  },
  // ---- NON-ONCOLOGY: female / functional urology ----
  {
    id: "female-001",
    topic: "non-oncology", subtopic: "female-functional",
    question: "Pelvic organ prolapse quantification (POP-Q) stage 0 indicates:",
    options: [
      "No prolapse demonstrated",
      "Prolapse extending well beyond the hymen",
      "Complete uterine procidentia",
      "Prolapse always requiring surgery"
    ],
    correctIndex: 0,
    explanation: "POP-Q stage 0 means no prolapse is demonstrated on examination; staging increases with descent relative to the hymen."
  },

  // ---- INFECTIONS & OTHER: UTI / urosepsis / Fournier's ----
  {
    id: "infect-001",
    topic: "infections-other", subtopic: "uti-urosepsis",
    question: "In a patient with suspected urosepsis, EAU guidelines emphasise which as the most time-critical intervention?",
    options: [
      "Prompt source control (e.g. decompression of an obstructed infected system) plus early appropriate antibiotics",
      "Waiting for urine culture results before any antibiotics",
      "Oral antibiotics only, outpatient management",
      "Immediate nephrectomy in all cases"
    ],
    correctIndex: 0,
    explanation: "Urosepsis from an obstructed, infected urinary tract requires urgent source control (stent/nephrostomy) alongside early broad-spectrum antibiotics — delaying decompression worsens outcomes."
  },
  {
    id: "infect-002",
    topic: "infections-other", subtopic: "uti-urosepsis",
    question: "Fournier's gangrene is best characterised as:",
    options: [
      "A mild, self-limiting cellulitis of the perineum",
      "A necrotising fasciitis of the perineal/genital region requiring emergency surgical debridement",
      "A viral infection treated with antivirals alone",
      "A chronic condition managed with oral antibiotics as outpatient"
    ],
    correctIndex: 1,
    explanation: "Fournier's gangrene is a life-threatening necrotising fasciitis of the perineum/genitalia requiring emergency, often repeated, surgical debridement plus broad-spectrum antibiotics."
  },
  {
    id: "infect-003",
    topic: "infections-other", subtopic: "uti-urosepsis",
    question: "Recurrent uncomplicated UTIs in women may be offered which non-antibiotic preventive strategy per EAU guidelines?",
    options: [
      "Cranberry products / vaginal oestrogen in postmenopausal women as adjuncts",
      "Prophylactic nephrectomy",
      "Permanent catheterisation",
      "Routine prophylactic chemotherapy"
    ],
    correctIndex: 0,
    explanation: "Non-antibiotic measures such as vaginal oestrogen in postmenopausal women, and cranberry products, are reasonable adjuncts/options for recurrent UTI prevention alongside behavioural measures."
  },
  // ---- INFECTIONS & OTHER: trauma / reconstruction ----
  {
    id: "trauma-001",
    topic: "infections-other", subtopic: "trauma-reconstruction",
    question: "Blood at the urethral meatus after pelvic trauma should raise suspicion for:",
    options: [
      "Bladder cancer",
      "Urethral injury — retrograde urethrogram should be considered before urethral catheterisation",
      "Normal finding, catheterise immediately without further workup",
      "Isolated renal contusion only"
    ],
    correctIndex: 1,
    explanation: "Blood at the meatus after trauma is a classic sign of urethral injury; blind catheterisation risks converting a partial tear into a complete disruption, so imaging (retrograde urethrogram) is considered first."
  },
  {
    id: "trauma-002",
    topic: "infections-other", subtopic: "trauma-reconstruction",
    question: "Most high-grade blunt renal traumas in haemodynamically stable patients are managed with:",
    options: [
      "Immediate nephrectomy",
      "Conservative (non-operative) management with monitoring",
      "Routine ureteric stenting for all grades",
      "Bilateral nephrectomy"
    ],
    correctIndex: 1,
    explanation: "Non-operative management is preferred for blunt renal trauma in stable patients, even for higher grades, reserving intervention for ongoing haemorrhage or specific complications."
  },
  // ---- INFECTIONS & OTHER: andrology / male infertility ----
  {
    id: "andro-001",
    topic: "infections-other", subtopic: "andrology",
    question: "The most common surgically correctable cause of male infertility is:",
    options: ["Varicocele", "Hypospadias", "Testicular torsion", "Peyronie's disease"],
    correctIndex: 0,
    explanation: "Varicocele is the most common identifiable and surgically correctable cause of male infertility, though not all varicoceles require treatment."
  },
  {
    id: "andro-002",
    topic: "infections-other", subtopic: "andrology",
    question: "First-line treatment for erectile dysfunction in most patients, per EAU guidelines, is:",
    options: [
      "Penile prosthesis implantation",
      "Oral PDE5 inhibitors",
      "Intracavernosal injections as first choice always",
      "Testosterone replacement regardless of levels"
    ],
    correctIndex: 1,
    explanation: "Oral PDE5 inhibitors are first-line therapy for erectile dysfunction in most patients due to efficacy, ease of use, and safety profile; other options are considered when PDE5i fails or is contraindicated."
  },
  // ---- INFECTIONS & OTHER: paediatric urology ----
  {
    id: "paeds-001",
    topic: "infections-other", subtopic: "paediatric",
    question: "Antenatally detected hydronephrosis in a newborn most commonly requires which initial step after birth?",
    options: [
      "Immediate surgery within 24 hours in all cases",
      "Postnatal ultrasound to reassess, with further workup guided by severity (e.g. VCUG, MAG3 if indicated)",
      "No follow-up needed regardless of severity",
      "Empirical antibiotics for life"
    ],
    correctIndex: 1,
    explanation: "Most antenatal hydronephrosis is followed with postnatal ultrasound and risk-stratified further investigation; only a minority need early surgical intervention, and unnecessary invasive workup is avoided in low-risk cases."
  },
  {
    id: "paeds-002",
    topic: "infections-other", subtopic: "paediatric",
    question: "Cryptorchidism (undescended testis) that has not resolved spontaneously should typically be surgically corrected (orchidopexy) by what age to optimise fertility/malignancy risk outcomes?",
    options: ["By 5 years of age", "By around 12-18 months of age", "Only after puberty", "Surgery is never indicated"],
    correctIndex: 1,
    explanation: "EAU/paediatric urology guidance recommends orchidopexy around 12-18 months of age if descent hasn't occurred spontaneously, to optimise fertility potential and allow surveillance for malignancy risk."
  },
  // ---- INFECTIONS & OTHER: transplantation ----
  {
    id: "transplant-001",
    topic: "infections-other", subtopic: "transplantation",
    question: "In renal transplantation, the donor kidney is typically placed:",
    options: [
      "In the native renal fossa after removing the native kidney",
      "Heterotopically in the iliac fossa, with vascular anastomosis to the iliac vessels",
      "In the thoracic cavity",
      "Always with removal of both native kidneys first"
    ],
    correctIndex: 1,
    explanation: "The transplant kidney is placed heterotopically in the iliac fossa with anastomosis to the iliac vessels and the ureter implanted into the bladder; native kidneys are usually left in situ."
  }
];
```

- [ ] **Step 2: Manual verification**

In the browser console:
```js
QUESTIONS.length // expect 34
new Set(QUESTIONS.map(q => q.id)).size === QUESTIONS.length // expect true
["oncology","non-oncology","infections-other"].every(t => QUESTIONS.some(q => q.topic === t)) // expect true
```
Expected: all checks pass, no console errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Add non-oncology and infections/other question batches (21 questions)"
```

*(Note: this brings the total to 34 questions — a working foundation. Reaching the ~100-120 target from the spec happens via Task 8 below, which adds further batches using the same schema.)*

---

### Task 4: Home screen — topic picker + mode selector

**Files:**
- Modify: `index.html` (replace `renderHome` placeholder)

- [ ] **Step 1: Implement `renderHome`**

Replace the placeholder `renderHome` function with:

```js
function getTopics() {
  return [...new Set(QUESTIONS.map(q => q.topic))];
}

function renderHome() {
  const container = document.createElement("div");

  const title = document.createElement("h1");
  title.textContent = "UroQuiz";
  container.appendChild(title);

  const topicCard = document.createElement("div");
  topicCard.className = "card";
  const topicTitle = document.createElement("h3");
  topicTitle.textContent = "Topics";
  topicCard.appendChild(topicTitle);

  getTopics().forEach(topic => {
    const row = document.createElement("div");
    row.className = "topic-row";
    const checkbox = document.createElement("input");
    checkbox.type = "checkbox";
    checkbox.id = "topic-" + topic;
    checkbox.checked = state.selectedTopics.includes(topic);
    checkbox.addEventListener("change", () => {
      if (checkbox.checked) state.selectedTopics.push(topic);
      else state.selectedTopics = state.selectedTopics.filter(t => t !== topic);
    });
    const label = document.createElement("label");
    label.htmlFor = checkbox.id;
    label.textContent = topic;
    row.appendChild(checkbox);
    row.appendChild(label);
    topicCard.appendChild(row);
  });
  container.appendChild(topicCard);

  const modeCard = document.createElement("div");
  modeCard.className = "card";
  const modeTitle = document.createElement("h3");
  modeTitle.textContent = "Mode";
  modeCard.appendChild(modeTitle);

  ["study", "exam"].forEach(mode => {
    const row = document.createElement("div");
    row.className = "topic-row";
    const radio = document.createElement("input");
    radio.type = "radio";
    radio.name = "mode";
    radio.id = "mode-" + mode;
    radio.checked = state.mode === mode;
    radio.addEventListener("change", () => { state.mode = mode; });
    const label = document.createElement("label");
    label.htmlFor = radio.id;
    label.textContent = mode === "study" ? "Study Mode (immediate feedback)" : "Exam Mode (review at end)";
    row.appendChild(radio);
    row.appendChild(label);
    modeCard.appendChild(row);
  });
  container.appendChild(modeCard);

  const startBtn = document.createElement("button");
  startBtn.textContent = "Start Quiz";
  startBtn.addEventListener("click", startQuiz);
  container.appendChild(startBtn);

  const statsBtn = document.createElement("button");
  statsBtn.className = "secondary";
  statsBtn.textContent = "View Stats";
  statsBtn.style.marginLeft = "0.5rem";
  statsBtn.addEventListener("click", () => { state.screen = "stats"; render(); });
  container.appendChild(statsBtn);

  return container;
}

function shuffle(array) {
  const a = array.slice();
  for (let i = a.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [a[i], a[j]] = [a[j], a[i]];
  }
  return a;
}

function startQuiz() {
  const topics = state.selectedTopics.length ? state.selectedTopics : getTopics();
  const pool = QUESTIONS.filter(q => topics.includes(q.topic));
  state.sessionQuestions = shuffle(pool);
  state.currentIndex = 0;
  state.answers = [];
  state.screen = "quiz";
  render();
}
```

- [ ] **Step 2: Manual verification**

Open `index.html`. Expected: page shows "UroQuiz" title, a "Topics" card with 3 checkboxes (oncology, non-oncology, infections-other), a "Mode" card with Study/Exam radios (Study checked by default), a "Start Quiz" button, and a "View Stats" button. Checking/unchecking topics and clicking "Start Quiz" should navigate away (quiz screen will show its placeholder until Task 5).

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Implement home screen with topic picker and mode selector"
```

---

### Task 5: Quiz screen — Study and Exam modes

**Files:**
- Modify: `index.html` (replace `renderQuiz` placeholder)

- [ ] **Step 1: Implement `renderQuiz`**

```js
function renderQuiz() {
  const container = document.createElement("div");
  const q = state.sessionQuestions[state.currentIndex];

  if (!q) {
    state.screen = "results";
    render();
    return container;
  }

  const progress = document.createElement("div");
  progress.className = "progress";
  progress.textContent = `Question ${state.currentIndex + 1} of ${state.sessionQuestions.length} — ${q.subtopic}`;
  container.appendChild(progress);

  const card = document.createElement("div");
  card.className = "card";
  const questionText = document.createElement("p");
  questionText.textContent = q.question;
  card.appendChild(questionText);

  const existingAnswer = state.answers.find(a => a.questionId === q.id);
  const alreadyAnswered = !!existingAnswer;

  q.options.forEach((optionText, idx) => {
    const label = document.createElement("label");
    label.className = "option";
    label.textContent = optionText;

    if (state.mode === "study" && alreadyAnswered) {
      if (idx === q.correctIndex) label.classList.add("correct");
      else if (idx === existingAnswer.chosenIndex) label.classList.add("incorrect");
    }

    label.addEventListener("click", () => {
      if (state.mode === "study" && alreadyAnswered) return;
      recordAnswer(q, idx);
      render();
    });
    card.appendChild(label);
  });

  if (state.mode === "study" && alreadyAnswered) {
    const explanation = document.createElement("p");
    explanation.className = "explanation";
    explanation.textContent = existingAnswer.correct
      ? "Correct. " + q.explanation
      : "Incorrect. " + q.explanation;
    card.appendChild(explanation);

    const nextBtn = document.createElement("button");
    nextBtn.textContent = state.currentIndex + 1 < state.sessionQuestions.length ? "Next Question" : "See Results";
    nextBtn.addEventListener("click", () => {
      state.currentIndex++;
      render();
    });
    card.appendChild(nextBtn);
  }

  container.appendChild(card);
  return container;
}

function recordAnswer(question, chosenIndex) {
  const correct = chosenIndex === question.correctIndex;
  state.answers.push({ questionId: question.id, chosenIndex, correct });

  if (state.mode === "exam") {
    state.currentIndex++;
  }
}
```

- [ ] **Step 2: Manual verification**

Open `index.html`, select Study Mode, click "Start Quiz". Expected: first question shows with options; clicking an option immediately highlights correct (green) and incorrect (red, if wrong) plus an explanation and a "Next Question"/"See Results" button. Reload, switch to Exam Mode, start quiz: expected clicking an option immediately advances to the next question with no highlight/explanation shown, until questions run out and it moves to Results (placeholder).

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Implement quiz screen for Study and Exam modes"
```

---

### Task 6: Results screen + retry-missed

**Files:**
- Modify: `index.html` (replace `renderResults` placeholder)

- [ ] **Step 1: Implement `renderResults`**

```js
function renderResults() {
  const container = document.createElement("div");
  const total = state.answers.length;
  const correctCount = state.answers.filter(a => a.correct).length;

  saveSessionToStorage();

  const summary = document.createElement("div");
  summary.className = "card";
  const scoreText = document.createElement("h2");
  scoreText.textContent = `Score: ${correctCount} / ${total}`;
  summary.appendChild(scoreText);
  container.appendChild(summary);

  const missed = state.answers.filter(a => !a.correct);
  if (missed.length) {
    const missedTitle = document.createElement("h3");
    missedTitle.textContent = "Missed Questions";
    container.appendChild(missedTitle);

    missed.forEach(answer => {
      const q = state.sessionQuestions.find(q => q.id === answer.questionId);
      const card = document.createElement("div");
      card.className = "card";
      const qText = document.createElement("p");
      qText.textContent = q.question;
      card.appendChild(qText);
      const correctText = document.createElement("p");
      correctText.textContent = "Correct answer: " + q.options[q.correctIndex];
      card.appendChild(correctText);
      const explanation = document.createElement("p");
      explanation.className = "explanation";
      explanation.textContent = q.explanation;
      card.appendChild(explanation);
      container.appendChild(card);
    });

    const retryBtn = document.createElement("button");
    retryBtn.textContent = "Retry Missed Only";
    retryBtn.addEventListener("click", () => {
      state.sessionQuestions = shuffle(missed.map(a => state.sessionQuestions.find(q => q.id === a.questionId)));
      state.currentIndex = 0;
      state.answers = [];
      state.screen = "quiz";
      render();
    });
    container.appendChild(retryBtn);
  }

  const homeBtn = document.createElement("button");
  homeBtn.className = "secondary";
  homeBtn.textContent = "Back to Home";
  homeBtn.style.marginLeft = "0.5rem";
  homeBtn.addEventListener("click", () => { state.screen = "home"; render(); });
  container.appendChild(homeBtn);

  return container;
}
```

- [ ] **Step 2: Manual verification**

Complete a short quiz (select just one topic to keep it quick) in Exam Mode, deliberately answering some wrong. Expected: results screen shows "Score: X / Y", a "Missed Questions" section listing each missed question with its correct answer and explanation, a "Retry Missed Only" button, and "Back to Home". Clicking "Retry Missed Only" should start a new quiz containing only the previously missed questions.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Implement results screen with missed-question review and retry"
```

---

### Task 7: localStorage stats + stats dashboard

**Files:**
- Modify: `index.html` (add storage functions, replace `renderStats` placeholder)

- [ ] **Step 1: Add storage read/write functions**

Add near the top of the script, after `STORAGE_KEY`:

```js
function loadProgress() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (!raw) return { topicStats: {}, missedIds: [] };
    return JSON.parse(raw);
  } catch (e) {
    return { topicStats: {}, missedIds: [] };
  }
}

function saveProgress(progress) {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(progress));
}

function saveSessionToStorage() {
  const progress = loadProgress();

  state.answers.forEach(answer => {
    const q = state.sessionQuestions.find(q => q.id === answer.questionId);
    const topic = q.topic;
    if (!progress.topicStats[topic]) progress.topicStats[topic] = { attempted: 0, correct: 0 };
    progress.topicStats[topic].attempted++;
    if (answer.correct) progress.topicStats[topic].correct++;

    const missedSet = new Set(progress.missedIds);
    if (answer.correct) missedSet.delete(answer.questionId);
    else missedSet.add(answer.questionId);
    progress.missedIds = [...missedSet];
  });

  saveProgress(progress);
}
```

- [ ] **Step 2: Implement `renderStats`**

```js
function renderStats() {
  const container = document.createElement("div");
  const title = document.createElement("h1");
  title.textContent = "Stats";
  container.appendChild(title);

  const progress = loadProgress();

  const topics = Object.keys(progress.topicStats);
  if (!topics.length) {
    const empty = document.createElement("p");
    empty.textContent = "No quiz attempts recorded yet.";
    container.appendChild(empty);
  } else {
    topics.forEach(topic => {
      const s = progress.topicStats[topic];
      const card = document.createElement("div");
      card.className = "card";
      const pct = s.attempted ? Math.round((s.correct / s.attempted) * 100) : 0;
      card.textContent = `${topic}: ${s.correct}/${s.attempted} correct (${pct}%)`;
      container.appendChild(card);
    });
  }

  if (progress.missedIds.length) {
    const missedTitle = document.createElement("h3");
    missedTitle.textContent = `Missed Question Log (${progress.missedIds.length})`;
    container.appendChild(missedTitle);

    const retryBtn = document.createElement("button");
    retryBtn.textContent = "Quiz Missed Questions";
    retryBtn.addEventListener("click", () => {
      const missedQuestions = QUESTIONS.filter(q => progress.missedIds.includes(q.id));
      state.sessionQuestions = shuffle(missedQuestions);
      state.currentIndex = 0;
      state.answers = [];
      state.screen = "quiz";
      render();
    });
    container.appendChild(retryBtn);
  }

  const homeBtn = document.createElement("button");
  homeBtn.className = "secondary";
  homeBtn.textContent = "Back to Home";
  homeBtn.style.marginLeft = "0.5rem";
  homeBtn.addEventListener("click", () => { state.screen = "home"; render(); });
  container.appendChild(homeBtn);

  return container;
}
```

- [ ] **Step 3: Manual verification**

Complete a full quiz, then click through to "View Stats" from Home. Expected: per-topic accuracy shown, a missed-question count if any were missed, and a "Quiz Missed Questions" button that starts a quiz using only globally-missed questions (persisted across reloads — verify by reloading the page and checking stats still show).

In the browser console, confirm persistence directly:
```js
JSON.parse(localStorage.getItem("uroquiz-progress-v1"))
```
Expected: an object with `topicStats` and `missedIds` populated matching what was just answered.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Add localStorage-backed stats tracking and stats dashboard"
```

---

### Task 8: Expand question bank toward ~100-120 total

**Files:**
- Modify: `index.html` (append further question batches to `QUESTIONS`)

This task is content-authoring, not logic. Follow the exact schema established in Tasks 2-3 (`id`, `topic`, `subtopic`, `question`, `options`, `correctIndex`, `explanation`). Author additional MCQs to bring each subtopic up to roughly 7-10 questions, covering these subtopics (already seeded, now expanded) — targeting a final total between 100 and 120:

- `oncology`: prostate, bladder, rcc, utuc, testis, penile
- `non-oncology`: stones, bph, incontinence, neuro-urology, female-functional
- `infections-other`: uti-urosepsis, trauma-reconstruction, andrology, paediatric, transplantation

For each new question: give 4 plausible options with exactly one correct, and a 1-3 sentence explanation that ties the answer back to the guideline concept (e.g. risk stratification criteria, first-line vs second-line therapy, emergency vs elective management, imaging/staging thresholds) so it can be spot-checked against the current EAU guideline document.

- [ ] **Step 1: Author and append ~20-25 additional oncology questions** (across prostate, bladder, rcc, utuc, testis, penile) following the schema above.

- [ ] **Step 2: Manual verification**

```js
QUESTIONS.filter(q => q.topic === "oncology").length // expect roughly 33-38
new Set(QUESTIONS.map(q => q.id)).size === QUESTIONS.length // expect true, still no duplicate ids
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Expand oncology question batch toward target bank size"
```

- [ ] **Step 4: Author and append ~20-25 additional non-oncology questions** (across stones, bph, incontinence, neuro-urology, female-functional).

- [ ] **Step 5: Manual verification and commit** (same pattern as Steps 2-3, updated counts and commit message "Expand non-oncology question batch toward target bank size").

- [ ] **Step 6: Author and append ~20-25 additional infections-other questions** (across uti-urosepsis, trauma-reconstruction, andrology, paediatric, transplantation).

- [ ] **Step 7: Manual verification and commit** (same pattern, commit message "Expand infections/other question batch toward target bank size").

- [ ] **Step 8: Final count check**

```js
QUESTIONS.length // expect between 100 and 120
```

---

### Task 9: Publish as Claude Artifact and final walkthrough

**Files:** none (deployment step)

- [ ] **Step 1: Publish `index.html` via the Artifact tool**

Use the Artifact tool with `file_path` set to the project's `index.html`, a title (e.g. "UroQuiz"), a one-sentence description, and a favicon emoji (e.g. 🩺). This produces a private URL.

- [ ] **Step 2: Full manual walkthrough on the published URL**

On both a desktop-sized and mobile-sized browser window (or the phone itself):
- Home → select a topic subset → Study Mode → answer several questions → confirm immediate feedback and explanations appear.
- Home → select "all topics" → Exam Mode → complete the block → confirm no feedback until Results.
- Results → confirm missed questions list with explanations, "Retry Missed Only" works.
- Stats → confirm per-topic accuracy and missed-question log are present and match what was just answered.
- Reload the page → confirm Stats screen still shows prior data (localStorage persisted).

- [ ] **Step 3: Note the caveat to the user**

Remind that questions are AI-generated from training knowledge (per the spec) and should be spot-checked against the current EAU guideline edition before being relied on for final exam prep.
