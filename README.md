# getmydrug

A patient-facing, chat-style demo that answers three questions before a prescription is written:

1. What does my insurance require before it will cover this medicine?
2. Do my own treatment history and migraine pattern line up with those requirements?
3. Where can I look for a cash price or a manufacturer support program?

Today a patient usually gets a prescription, goes to the pharmacy, and only then learns about the cost and whether the plan approves it. getmydrug moves that discovery to the front, so the patient and prescriber can start from the plan's actual requirements.

This is a demonstration. It is not medical advice, it does not guarantee coverage, and it is not connected to any live pricing or payer system.

## Demo scope

| Area | Included |
|---|---|
| Diagnosis | Migraine |
| Drugs | Ubrelvy (ubrogepant), Nurtec ODT (rimegepant), Qulipta (atogepant) |
| Insurers | UnitedHealthcare (all three drugs), Cigna (Nurtec ODT only), plus a "No insurance" cash path |

Cigna's Ubrelvy and Qulipta options show as "soon" because no policy for them has been loaded yet.

## What the chat does

1. The patient picks a condition, an insurer and a drug. A text box also recognizes drug names, "migraine" and "start over".
2. A prescription-label style card shows the plan's criteria for first approval and for renewal, read from the criteria file for that insurer.
3. The patient can choose **"Is this for me?"** and answer a short set of questions (use, age, migraine days, triptans and preventives tried, state, other CGRP medicines). Only the questions that insurer's rules need are asked.
4. The page returns a checklist marked likely met, not met, or confirm, plus a copyable note the patient can bring to their prescriber.
5. Links to GoodRx, Mark Cuban Cost Plus Drugs and the manufacturer's site appear for each drug. A live price estimate tile is a placeholder.

## Run it

The page loads its data with `fetch`, and browsers block that on `file://`. Serve the folder locally:

```
cd getmydrug
python3 -m http.server 8000
```

Then open http://localhost:8000. The page loads js-yaml 4.1.0 and the Google Fonts stylesheet from CDNs, so it needs internet access.

## Repository layout

```
getmydrug/
  index.html                         the whole front end (HTML, CSS, JavaScript)
  README.md
  data/
    directory.yaml                   conditions, insurers, drugs, cash and support links
    pa_criteria_uhc.yaml             UnitedHealthcare prior authorization criteria
    eligibility_rules_uhc.yaml       how to ask about and score each UnitedHealthcare criterion
    pa_criteria_cigna.yaml           Cigna prior authorization criteria (Nurtec ODT)
    eligibility_rules_cigna.yaml     how to ask about and score each Cigna criterion
```

## How the data layer works

The page contains no payer rules. Everything it shows comes from YAML files fetched at runtime, so the same files can later be served by an API.

- **`directory.yaml`** lists the choices the chat offers. Each insurer entry points to its `criteria_file`, its `rules_file`, and the `drug_ids` it has data for. Drug entries carry plain-language descriptions, GoodRx and Cost Plus links, and manufacturer support links.
- **`pa_criteria_<insurer>.yaml`** holds the policy text in a common shape: a `policy` block (payer, program number, effective date, approval length), `definitions` (such as triptan lists), per-drug `initial_authorization` and `reauthorization`, `state_and_plan_notes`, and a disclaimer. Initial authorization is either one criteria list or several `pathways` (for example acute and preventive use). Each criterion has an `id` and a `requirement`, and can carry `options`, `count_required`, and a `ref:` to a shared list.
- **`eligibility_rules_<insurer>.yaml`** turns each criterion `id` into something the "Is this for me?" step can ask and score. It holds the medicine lists shown as choices, the allowed uses per drug, and a `checks` entry per criterion id with a check `type`, a plain label and any parameters such as a minimum count.

The rules file can also shape the interview. `questions` lists which steps to ask, `mdays_for` and `cgrp_question_for` limit those steps to certain uses, and `triptan_trial_question: false` skips the "tried long enough" follow-up for policies that do not set a trial length.

### Check types the engine understands

`use`, `age18`, `tried_triptans`, `tried_preventives`, `prevention_status`, `no_other_gepant`, `no_cgrp_mab_or_gepant`, `episodic_ge4`, `dx_migraine`, `qulipta_frequency`. A criterion with no matching check shows as "confirm with your prescriber".

## Adding an insurer or drug

1. Write `data/pa_criteria_<insurer>.yaml` in the shape above, with a unique `id` on every criterion.
2. Write `data/eligibility_rules_<insurer>.yaml` with a `checks` entry for every criterion id. Reuse existing check types where the policy language fits.
3. Add the insurer to `data/directory.yaml` with `criteria_file`, `rules_file` and `drug_ids`. Add any new drug to `drugs` with its links.
4. If the policy needs a rule the engine lacks, add a new `case` in `runCheck` in `index.html`.

A quick check that every criterion id has a rule: collect each `id` that sits next to a `requirement` in the criteria file and confirm it appears under `checks` in the rules file.

## Privacy

Answers to the "Is this for me?" questions live in a JavaScript variable in the open tab. The page does not use cookies, local storage or any logging, and its only network requests are for the static YAML files, fonts and the YAML library. The patient's answers are never sent anywhere. Keep that property if you add a backend, or decide deliberately how to handle protected health information first.

## Data sources and accuracy

| Insurer | Document | Effective |
|---|---|---|
| UnitedHealthcare | Clinical Pharmacy Programs, Prior Authorization/Medical Necessity, program 2026 P 2189-15 | 6/1/2026 |
| Cigna | Drug Coverage Policy, Migraine - Nurtec ODT, IP0147 | 4/15/2026 |

The YAML was distilled from the policy PDFs with AI assistance and has not been independently verified clause by clause. Confirm it against the source documents before any real-world use. Zavzpret appears in the UnitedHealthcare policy but is intentionally left out of the demo.

## Known limits

- Only first-approval criteria are scored. Renewal criteria are displayed, not evaluated.
- "Tried long enough" is taken at face value from the patient, and "not sure" answers become "confirm" items.
- For Cigna's preventive use, headache days (all types) under 15 stands in for the policy's "migraine headache days under 15".
- The UnitedHealthcare state rules cover California and Connecticut, Kentucky and Mississippi. Other state mandates are noted but not modeled.
- Cash price, coupon and manufacturer program links are pointers to external sites. They have not been checked for current validity, and Cost Plus may not stock brand-name gepants.
- Free-text routing is simple keyword matching, not a language model.

## Pilot: can a patient just ask an AI?

A small pilot ran six patient-style questions through Haiku, with and without web search, and scored the answers against the YAML.

- **With search on**, the model found the real payer policy PDFs and summarized them mostly accurately. The assumption that an assistant cannot find the criteria did not hold.
- **The search answers were incomplete.** The two UnitedHealthcare answers skipped criteria that decide approval (preventive-medicine status for Ubrelvy; migraine frequency, diagnosis and no-combination rules for Qulipta) without saying so.
- **With search off**, answers were generic. Two were wrong in a way that matters: it said Cigna may require trying other preventives first for Nurtec, which the policy no longer does, and it missed UnitedHealthcare's California waiver.
- **Cash price** was the weakest area. The model found a retail price but no usable coupon price.

This is a single run per prompt on one model, scored against data written for this project, so treat it as a pilot rather than a result. A fuller test would add more prompts, repeated runs, other assistants and harder cases.

## Roadmap

- Load Cigna's Ubrelvy (IP0148) and Qulipta policies, and more insurers.
- Backend and database for criteria, rules and links, with versioning by policy effective date.
- Live cash price lookup and verified manufacturer support program links.
- Deep links from ads that open the chat with a drug already chosen.
- More conditions, and a broader, repeated test against general-purpose assistants.