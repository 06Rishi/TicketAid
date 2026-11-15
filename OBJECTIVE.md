# TicketAid — Objective and Project Rules

## What this project is
TicketAid is a support-ticket triage tool for a single (simulated) bank support team.
A customer query goes in; the app shows:
- the predicted **intent** (one of 77 Banking77 intents) with calibrated confidence and top-3 alternatives
- a **rule-based priority recommendation** (Urgent / Normal / Low) from the intent
- a **suggested reply template** for that intent
- a **needs-human-review** flag when confidence is below threshold or the query looks out of scope

A fine-tuned transformer (DistilBERT, with MiniLM as a smaller comparison) runs **entirely in the visitor's browser** (ONNX + transformers.js). The site is a static Next.js app on Vercel. There is no backend and no database.

The plan, timeline, checklists and Decision Log live in [`docs/PLAN.md`](docs/PLAN.md).

## Hardware and environment
- Local GPU: NVIDIA RTX 3050, 6 GB VRAM. Training runs locally. Google Colab or Kaggle is the fallback.
- Python 3.11 in a single uv-managed `.venv` at the repo root, defined by `pyproject.toml` + `uv.lock` (both committed). Torch comes from the CUDA 13.0 wheel index. The Python version is pinned by `requires-python`.
- Node LTS (via fnm) for `web/`, pinned by `engines.node` in `web/package.json`.
- Add Python packages with `uv add` (never plain `pip install`), so `pyproject.toml` and `uv.lock` stay in sync.

## Repo layout
```
OBJECTIVE.md   this file: scope and rules
pyproject.toml Python dependencies (uv); uv.lock pins exact versions
ml/            Python: data, baseline, training, calibration, export, evaluation
  config.yaml  single source of truth for seeds, model names, held-out draws, hyperparameters
  data/        downloaded/derived data (git-ignored except small hand-written query files)
  tests/
web/           Next.js (App Router) + TypeScript + Tailwind + shadcn/ui
  lib/worker.ts   loads the ONNX model and runs inference in a Web Worker
  lib/triage.ts   temperature, threshold, intent → priority/reply logic
  public/intent_meta.json
results/       metrics.json, deploy_config.json, parity.json and plots (committed; the website reads these)
docs/          PLAN.md and design notes
```

## Methodology rules (do not break these)
1. **Two kinds of model, two separate questions.**
   - **Closed-set model:** trained on all 77 intents. The deployed model comes from here, and it gives the headline classification metrics.
   - **Open-set models:** 5 draws, each holding out 10 intents (seeded, stratified by priority group, listed in `config.yaml`), 1 training seed per draw. Used only for the near-unknown experiment. Never deployed.
2. **Data splits.** Banking77 train is split 80/10/10, stratified, with the seed from `config.yaml`:
   - `train`: fitting
   - `val_select`: early stopping, learning-rate choice, seed choice
   - `val_calib`: temperature, review threshold, quantization drop, deploy rule
   - Final test: the Banking77 test split and `ood_test.txt`. Read **once**, by `evaluate.py` only, at the end, and reported as-is.
3. **Training protocol.** Up to 15 epochs with early stopping on `val_select` macro-F1 (patience 3), warmup 10%, weight decay 0.01, batch size 32, fp16, `max_length` 64.
   - Learning-rate grid {2e-5, 3e-5, 5e-5} on seed 0. If the top two are within 0.5 macro-F1, run seeds 1–2 for both and pick by the mean.
   - Complete the chosen learning rate to 5 seeds (earlier runs count). Report mean ± std. The deploy candidate is the seed with the best `val_select` macro-F1.
   - Open-set draws reuse the chosen closed-set learning rate.
4. **Quantize before the results lock, and deploy by a fixed rule.** Export the candidate to ONNX and quantize **once** (q8). Deploy int8 DistilBERT if its `val_calib` macro-F1 drop is ≤ 1 point. Otherwise deploy fp16 DistilBERT (if it runs on the WASM backend) if its drop is ≤ 1 point. Otherwise deploy int8 MiniLM. The single test pass reports both fp32 and the deployed precision.
5. **The deployed model's temperature and review threshold are tuned on the deployed artifact's own data:** its `val_calib` logits plus `ml/data/ood_val.txt`. The operating point flags at most 5% of in-domain `val_calib`. Never reuse settings tuned on an open-set model.
6. **Off-topic sets.** About 150 queries each. `ood_test.txt` is written blind, before any model exists, and is never used for tuning.
7. **Name metrics honestly.**
   - "Priority agreement" means how often the predicted intent maps to the same priority as the true intent. Never "priority accuracy".
   - Priorities are a rule-based recommendation, not a validated prediction.
   - Word highlighting is "input sensitivity" (leave-one-word-out), not "explanation".
   - Out-of-scope metrics carry bootstrap 95% confidence intervals. Selective prediction reports coverage, accuracy on unflagged queries, and the share of errors flagged.
8. **Every reported number comes from a script.** Scripts write results to `results/metrics.json`. No numbers typed by hand into the README or the site.
9. **Artifact integrity and parity.** The exact quantized file that is evaluated is the file uploaded to the Hugging Face Hub, and its SHA256 is recorded. The browser runs WASM + q8. The same 20 fixed queries must give the same top-1 intent and probabilities within 0.02 in Python ONNX and in the browser.

## Safety and scope rules for the app
- Suggested replies are **predefined templates** in `intent_meta.json`. No generated banking advice.
- Templates and UI must never ask for passwords, PINs, OTPs, card numbers or account numbers.
- Never claim to perform transactions or account actions.
- Always show a clear human-handoff state.
- The About page states: simulated demo, not a real bank. The training data is UK/EU-style online banking, so India-specific queries (UPI etc.) may be flagged for review.

## Data and licences
- Banking77 (PolyAI), CC BY 4.0. Attribute in the README and on the site's About page, with a link to the dataset and the licence.
- Do not commit the dataset. Load it via Hugging Face `datasets` in `ml/data.py`.
- Hand-written off-topic queries (`ood_val.txt`, `ood_test.txt`) are original to this project and are committed.
- Check the licence of any pretrained model before using it, and record it in the Decision Log.

## Commands (filled in as they come to exist)
```
# Environment (repo root)
uv sync                   # create/update .venv exactly from uv.lock
source .venv/bin/activate

# ML
cd ml
python data.py            # build splits
python baseline.py        # TF-IDF + LR
python train.py --model distilbert --setting closed
python train.py --model distilbert --setting open
python export_onnx.py     # ONNX + q8 (before evaluation)
python calibrate.py --model distilbert
python evaluate.py        # the single test pass: results/metrics.json + plots
pytest

# Web
cd web && npm install && npm run dev
npm run lint && npm run build
```

## Definition of done for any task
- Code runs from a clean checkout using the commands above.
- Any new metric or plot is produced by a script and saved under `results/`.
- Tests are added or updated where the logic is non-trivial (splits, mapping, threshold logic).
- The `docs/PLAN.md` checkbox is ticked, and the Decision Log is updated if a choice was made.
