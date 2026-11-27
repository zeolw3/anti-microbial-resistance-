# Antibiotic Exposure History

An awareness tool that keeps a person's antibiotic history, notes patterns worth discussing with a doctor, and produces a doctor-ready PDF report. Patients, doctors and lab technicians each get their own view of one shared record.

> **Synthetic data only. Not medical advice. Not clinically validated. This is a hackathon prototype.**

## The problem

Antibiotic resistance grows when antibiotics are overused or misused, and past antibiotic use raises a person's later risk of a resistant infection. In India, antibiotics are prescription-only (Schedule H/H1, red line on the strip), yet they are still sold without prescriptions. Patients have no single record of what they have taken, so doctors often cannot see the full history.

## What it does

1. A patient enters courses with drug autocomplete (brand or generic, typo tolerant).
2. The app maps each drug to its generic name and its WHO AWaRe group.
3. It draws a timeline of every course.
4. A rule engine notes patterns (see [Pattern rules](#pattern-rules)).
5. It generates a PDF report to show a doctor, including lab results beside earlier drug exposure.

## Three views, one data model

| Role | Can do |

| **Patient** | Add courses, see timeline and patterns, download the report, share or stop sharing with a doctor or lab |
| **Doctor** | See the history of patients who shared it, add prescriptions (dated, with diagnosis), download the report |
| **Lab technician** | Enter culture and sensitivity results (organism, S/I/R per antibiotic). Sees lab results only, not the antibiotic history |

Every entry carries a source tag (self-reported, doctor-entered, lab-entered). The report shows how much of the history is unconfirmed.

Access is consent-based: staff only see patients who have shared with them, and patients can revoke at any time. Roles are chosen with a demo switcher; there is no real login (see [Limitations](#limitations)).

## Pattern rules

All thresholds live in `config/rules.yaml`. Every number is an adjustable default chosen by the team, not a clinically validated cutoff.

| Pattern | Default rule |

| Repeated courses | 3 or more courses starting within 90 days |
| Possible old-prescription reuse | Course started more than 14 days after the prescription date, or one prescription linked to more than one course |
| Frequent higher-risk-group use | 3 or more AWaRe Watch/Reserve courses within 365 days |
| Note: no prescription on record | Informational only |

Flags say "consider discussing this pattern with your doctor". The tool never says "misuse". A report with no flags says "no patterns found in the data entered", never "low risk".

### How "broad-spectrum" is defined

We use the WHO AWaRe classification (Access / Watch / Reserve) and treat Watch and Reserve as the higher-resistance-risk groups. This is a proxy: AWaRe was designed for population-level stewardship, not to predict one person's risk, and some Access drugs (for example amoxicillin-clavulanate) are fairly broad. Fixed-dose combinations that AWaRe does not classify are marked *Not classified/FDC* and are not counted as Watch/Reserve.

## What is original

Many tools store prescriptions or send reminders. This project adds a resistance exposure history that connects patient, doctor and lab: an explainable risk-pattern engine plus a doctor-ready report that places lab-reported resistance next to earlier recorded use of the same drug.

(Part of the project overlaps Track 1, organising prescriptions. The pattern engine and lab linkage are the original part.)

## Run it

```bash
git clone <your repo url>
cd antibiotic-history-tracker
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
streamlit run src/app.py
```

The app creates and fills a local demo database on first start (`data/app.db`, ignored by git).

### Demo story

1. View as Patient **Asha** and see four courses flagged.
2. Switch to Doctor **Dr. Rao** and open Asha's shared history.
3. Switch to Lab technician to see the urine culture entered with resistance to ciprofloxacin and levofloxacin.
4. Download the report, which places those results beside Asha's earlier use of the same drugs.

**Dev** shares with nobody, so no doctor sees him. **Ravi** has spread-out, prescribed courses and no flags.

## Tests

```bash
pip install -r requirements-dev.txt
pytest
```

Tests cover the rules (including boundary cases and cases that must not flag), consent and permission checks, drug search, the scripted scenarios in `src/seed.py`, and PDF generation.

## Validation: what we did and did not do

- Rules tested on scripted synthetic scenarios, including no-flag scenarios to catch false positives. This only shows the code does what the rules say. It does not show the rules are clinically right.
- OCR of strips was not built (cut for time). Manual entry only. Future work: OCR of printed strips with a confirm screen. Handwritten prescriptions would remain out of scope.
- **Reviewer feedback on the report:** [TEAM:                    **short doc report**.]
- **Dataset check:** [TEAM: **AWaRe groups checked against WHO EML [edition] by [name] on [date]". The drafted CSV is unverified until this is filled in.]**

## Limitations

- The tool only knows what is entered. It cannot tell what was actually taken, whether a course was finished, or why it was started.
- "Old prescription reuse" is a heuristic based on dates and links. Legitimate delays or repeat prescriptions can trigger it.
- No real authentication. Production use would need proper login, audit logs, encryption and compliance (for example India's DPDP Act).
- It does not address over-the-counter sales, the root cause. It supports stewardship from the patient and clinician side. Anonymised, aggregated surveillance is future work.
- On a hosted demo, all visitors share one demo database.

## Ethics

- Synthetic data only; no real patient data.
- Awareness tool, not medical advice. Lab results are shown exactly as entered, with no interpretation beyond noting earlier use of the same drug.
- Consent is built in: patients control who sees their history.

## Project layout

```
config/rules.yaml            adjustable thresholds with sources
data/antibiotics_india.csv   drug list (brand, generic, strength, AWaRe group)
src/models.py                data model with source tags
src/db.py                    SQLite storage and consent checks
src/lookup.py                drug search, brand -> generic -> AWaRe
src/rules.py                 pattern detectors, lab matching, completeness
src/timeline.py              timeline chart
src/report.py                PDF report
src/seed.py                  synthetic demo patients
src/app.py                   Streamlit app (three views)
tests/                       pytest tests
AI_DISCLOSURE.md             how AI was used
```

## Team and timeline

[TEAM: luna]. All code built 24-30 Nov 2026. AI use is disclosed in [AI_DISCLOSURE.md](AI_DISCLOSURE.md).

## Sources

- WHO AWaRe classification of antibiotics (WHO Essential Medicines List). [TEAM:             .]
