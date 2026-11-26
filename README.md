# anti-microbial-resistance- luna
basically an app that notifies the patients of the time to take meds and a digital folder which only needs login access and their data will be stored there itself and they can be accessed by doctors 
Adherence and resistance data, side by side, under the patient's control.

When a fungal skin infection won't clear, a doctor has to guess: did the patient miss doses, or is the fungus resistant? Luna AMR is the first step toward answering that, by putting both kinds of evidence in one consent-controlled place.

> "Not a medical device." Luna AMR is a research and education prototype. It gives no diagnosis or treatment advice. All demo patients, sequencing reads and results are synthetic.

 **Problem**: Recurrent fungal skin infections (e.g. tinea/ringworm) are increasingly hard to clear in India. Treatment runs for weeks, patients stop early or miss doses, and they move between clinics with no shared history. Resistant strains add a second, invisible cause of failure. 
 
 **Insight**: Doctors rarely see adherence and resistance data together, so they can't tell non-adherence from true resistance.
 **Solution**: One app with three views: a "patient app" that captures a prescription and logs doses, a "doctor dashboard" with adherence and red flags, and a "researcher view" with de-identified data and a "genomics pipeline" that detects known resistance mutations.
 **Key design choice**: **Consent is enforced on the server.** Doctors and researchers only ever receive data from patients who opted in, and patients can withdraw at any time.
 **What works today**: End-to-end demo: prescription photo to reminders to dose logging to consent to doctor and researcher views to pipeline run and resistance report. 
 **What is simulated**: All patient data and sequencing reads. The demo resistance marker is fake. Real-data steps are documented below. 

## Why it's different

1. **Joins two worlds.** Most adherence apps stop at reminders, and most genomics tools stop at the lab. Luna AMR connects them.
2. **Consent-first architecture.** Visibility is decided by the backend on every request, not by the interface.
3. **Honest about uncertainty.** Patients confirm every scanned prescription. Adherence is labelled self-reported. A resistance marker is shown as supporting evidence, never as proof.
4. **Reproducible.** A synthetic benchmark (spiked mutations in a fake genome) lets anyone check that the pipeline recovers what it should.

## Live demo walkthrough (3 minutes)

All demo passwords are `demo`.

| Login | Role | Notes |
|---|---|---|
| `patient1` | patient | Empty account, no consent yet |
| `asha` | patient | Sample history, consent already given |
| `doctor1` | doctor | Sees consenting patients only |
| `researcher1` | researcher | Sees anonymized cohort, can run the pipeline |

1. **Patient:** log in as `patient1`, scan a prescription, review and confirm the extracted medicines. Note the steroid warning on the combination cream and the daily reminders.
2. **Log a dose:** tap "Taken".
3. **Consent gate (the key moment):** log in as `doctor1`. `patient1` is **not listed**. `asha` is, with red flags.
4. **Opt in:** as `patient1`, tick consent. Refresh the doctor view and `patient1` now appears.
5. **Researcher:** log in as `researcher1`. Patients show as anonymous IDs. Click **Run pipeline** and open the resistance report.
6. **The point:** adherence plus genomics lets a doctor separate "didn't take it" from "likely resistant".

## How it works

```
 Patient                    Backend (FastAPI)                 Doctor / Researcher
 -------                    -----------------                 -------------------
 photo of prescription ---> /scan  (vision AI -> JSON)
 confirms and corrects ---> /meds  (saved to SQLite)
 taps "Taken"          ---> /taken
 ticks consent         ---> /consent  ---------------------->  /patients  (doctor, consented only)
                                                                /cohort    (researcher, anonymized)
                            FASTQ reads --> pipeline.sh ----->  /report    (resistance markers)
```

### Genomics pipeline

| Step | Tool | Purpose |
|---|---|---|
| 1 | fastp | Trim low-quality bases, remove junk reads |
| 2 | bwa + samtools (optional) | Remove human reads from swab data (also a privacy safeguard) |
| 3 | SPAdes | Assemble reads into contigs |
| 4 | bwa + samtools + bcftools | Align to a reference and call mutations |
| 5 | BLAST (optional) | Species identification against an ITS database |
| 6 | `make_report.py` | Match mutations to known resistance markers and write `report.json` |

## Features

**Patient**
- Prescription photo to medicines, doses and timings via a vision model
- Review and correct before anything is saved
- Steroid flag from an editable keyword list
- Reminders, one-tap logging, 14-day adherence figure
- Consent switch, changeable at any time

**Doctor**
- Sees consenting patients only
- Adherence bar and plain-language red flags (low adherence, steroid-containing cream in history)

**Researcher**
- Cohort of consenting patients with anonymous IDs
- Run the sequencing pipeline and view the resistance report

## Privacy and consent

- Consent is stored per patient and checked **on the server** for every doctor and researcher request.
- Researchers get anonymous IDs, never names.
- Withdrawing consent removes the patient from other views immediately.
- Human reads are discarded in the pipeline when host removal is enabled.

**Prototype-level security, stated plainly:** passwords use SHA-256 and sessions live in memory. A real deployment would need salted hashing (bcrypt or argon2), HTTPS, encrypted storage, audit logs, verified clinician identity and compliance with India's DPDP Act.

## Honest limitations

- Adherence is self-reported: a tick means the patient said they took the dose.
- Prescription reading can err, especially on handwriting, which is why patients confirm the result.
- The steroid warning uses a short keyword list and misses many brand names.
- The pipeline finds **known** markers at **specific positions** only. A marker suggests possible resistance and does not prove treatment failure.
- Variant calling assumes a haploid genome and simple conditions. Real analysis needs QC, coverage thresholds and expert review.
- Reminders are browser alerts while the page is open, not phone notifications.
- Single-user SQLite, not built for concurrent use.

## Roadmap

- WhatsApp/SMS reminders and push notifications
- PostgreSQL or Supabase with row-level security and proper authentication
- Doctor-to-researcher requests for lab comparison
- Curated ITS database for species ID, plus phylogenetic trees of isolates
- Regional resistance map from anonymized, aggregated data
- Regional-language and handwriting accuracy testing
- Verified doctor and researcher registration

---

## Run it yourself

### Requirements
- Linux, macOS, or Windows with **WSL2** (Ubuntu)
- [Miniforge](https://github.com/conda-forge/miniforge) (conda)
- 8 to 16 GB RAM for real fungal genomes (the synthetic demo needs far less)
- Optional: a [Gemini API key](https://aistudio.google.com) for real prescription scanning. Without one, the app returns demo medicines.

### Install

```bash
git clone <your-repo-url>
cd luna-amr

conda create -n derma -c conda-forge -c bioconda \
    python=3.11 spades fastp bwa samtools bcftools blast pip
conda activate derma
pip install fastapi uvicorn python-multipart google-genai pillow
```

### Start the web app

```bash
conda activate derma
export GEMINI_API_KEY="your_key"   # optional
uvicorn main:app --reload
```

Open **http://localhost:8000**. Start the server from inside the `derma` environment, or the "Run pipeline" button won't find the sequencing tools.

### Run the pipeline on synthetic data

```bash
python simulate_reads.py
SKIP_HOST=1 bash pipeline.sh sample_R1.fastq sample_R2.fastq ref.fa
cat report.json
```

`simulate_reads.py` builds a fake 50 kb genome with one spiked mutation at position 10600. A working pipeline reports that position. To benchmark, spike several mutations and measure how many are recovered and how many false positives appear.

| Variable | Meaning |
|---|---|
| `SKIP_HOST=1` | Skip human read removal (needed unless you have a human genome index) |
| `THREADS=8` | CPU threads (default 4) |
| `GEMINI_API_KEY` | Enables real prescription scanning |

### Moving to real data

1. **Reference genome:** download a dermatophyte reference (e.g. *Trichophyton*) from NCBI as `ref.fa` and cite the accession.
2. **Reads:** use public sequencing runs from NCBI SRA as `sample_R1/R2.fastq`.
3. **Markers:** replace `known_markers.tsv` with published positions (e.g. SQLE for terbinafine, ERG11/CYP51 for azoles). Coordinates must match your reference. The demo entry is fake.
4. **Human read removal:** download GRCh38 as `human.fa`, run `bwa index human.fa` once, then run the pipeline without `SKIP_HOST`.
5. **Species ID:** build a BLAST database of fungal ITS sequences and use the `blastn` line in `pipeline.sh`.

Always label simulated or public data as such.

## API reference

All endpoints except `/login` and `/` need the header `Authorization: <token>` from `/login`. Interactive docs: **http://localhost:8000/docs**.

| Method | Path | Role | Description |
|---|---|---|---|
| POST | `/login` | any | Body `{name, password}`, returns a token |
| POST | `/scan` | patient | Upload a prescription image, returns extracted medicines (demo data without API key) |
| POST | `/meds` | patient | Save confirmed medicines |
| GET | `/today` | patient | Today's reminders and 14-day adherence |
| POST | `/taken` | patient | Toggle a dose as taken |
| POST | `/consent` | patient | Body `{value: true/false}` |
| GET | `/patients` | doctor | Consenting patients with flags |
| GET | `/cohort` | researcher | Anonymized consenting patients |
| POST | `/pipeline` | researcher | Start the pipeline in the background |
| GET | `/report` | researcher | Pipeline status and `report.json` |

## Project structure

```
luna-amr/
├── main.py                 # FastAPI backend (API, database, roles, consent)
├── index.html              # Single-page frontend, served by the backend
├── pipeline.sh             # Genomics pipeline
├── make_report.py          # Turns variants.vcf into report.json
├── known_markers.tsv       # Known resistance positions (replace with published data)
├── simulate_reads.py       # Synthetic reference and reads for testing
├── read_prescription.py    # Standalone prescription reader (CLI)
├── environment.yml         # Optional conda environment
└── README.md
```

Created at runtime and not committed: `app.db`, `out/`, `report.json`, `variants.vcf`, `ref.fa`, `sample_*.fastq`. Keep API keys in environment variables or a git-ignored `.env`.

## Troubleshooting

| Problem | Fix |
|---|---|
| `command not found: fastp` or `spades.py` | Run `conda activate derma` first |
| `ModuleNotFoundError: fastapi` | `pip install fastapi uvicorn python-multipart` inside `derma` |
| Pipeline button shows `failed` | Check `sample_R1.fastq`, `sample_R2.fastq` and `ref.fa` exist (run `simulate_reads.py`) and the server was started inside `derma` |
| Scan always returns the same medicines | No `GEMINI_API_KEY` set, so demo data is used |
| Port 8000 in use | `uvicorn main:app --port 8001` |
| Files appear as folders in VS Code | Create them with **New File**, not New Folder |

## Team

Built for a hackathon by **Team Luna**.
## Disclaimer
This software is for research and education. It does not diagnose or treat disease. Consult a qualified doctor for any medical decision.
