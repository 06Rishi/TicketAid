# TicketAid — Plan and Timeline

**Start:** Fri 13 Nov 2026 **v1.0 target:** Sun 13 Dec 2026 **Buffer:** 14–20 Dec 2026

**Budget:** about 1.5 h/day, 6 days/week, so about 9 h/week. The Must tasks total about 36 h across 4 weeks plus setup. If a week slips, cut from the bottom of that week (Stretch first, then Should) before touching Must. If end-semester exams fall in this window, push the whole schedule back by the exam weeks. Don't compress it.

Task tags: **[M]** Must, needed for v1.0 · **[S]** Should, strong but cuttable · **[X]** Stretch, buffer week only.

## Milestones

| Date | Milestone | Proof it's done |
|---|---|---|
| Sun 15 Nov | Setup done | GPU visible in PyTorch; blank Next.js site live on Vercel |
| Sun 22 Nov | First model trained | DistilBERT closed-set val accuracy logged; browser spike runs a stock model |
| **Sun 29 Nov** | **ML results locked** | `results/metrics.json` complete for the test sets; git tag `v0.1-results` |
| **Sun 6 Dec** | **Live MVP** | Triage page on Vercel runs your quantized model; parity check passes |
| **Sun 13 Dec** | **v1.0** | Results and About pages, README with GIF, CI green |
| 14–20 Dec | Buffer | Slipped work, then Stretch items |

---

## Week 0: Setup (Fri 13 – Sun 15 Nov) · ~3 h

- [ ] [M] Create GitHub repo `ticketaid`. Add `CLAUDE.md`, `docs/PLAN.md`, Python + Node `.gitignore`, and an MIT licence for your code. (0.5 h)
- [ ] [M] `ml/.venv` with Python 3.11. Install PyTorch with CUDA (use the pytorch.org selector), then `transformers datasets evaluate accelerate scikit-learn pyyaml matplotlib pytest ruff`. Check `torch.cuda.is_available()` and the device name. (1 h)
- [ ] [M] `web/`: `create-next-app` (TypeScript, Tailwind, App Router) plus shadcn/ui. Connect the repo to Vercel and deploy the blank page. (1 h)
- [ ] [M] Hugging Face account and write token, saved in `.env` (git-ignored). (0.25 h)

## Week 1: Data, baseline, first fine-tune (Mon 16 – Sun 22 Nov) · ~8.5 h

- [ ] [M] `ml/config.yaml`: seed, model names, hyperparameters (`fp16`, `max_length: 64`, batch 32, epochs 3–5, lr), and the **10 held-out intents**, picked across different priority groups. Record why in the Decision Log. (0.5 h)
- [ ] [M] `data.py`: load Banking77; stratified 10% validation split from train; open-set split (drop the held-out intents from train and val, and tag their test examples as unknown). Tests: no overlap between splits, held-out intents absent from open-set train. (1.5 h)
- [ ] [M] `web/public/intent_meta.json`: for all 77 intents, a display name, priority (Urgent/Normal/Low) and reply template. Apply the safety rules in `CLAUDE.md`. Review every template yourself. (2 h)
- [ ] [M] `baseline.py`: TF-IDF + logistic regression on the closed set. Write validation accuracy and macro-F1 to `results/`. (1 h)
- [ ] [M] `train.py`: fine-tune DistilBERT on the closed set on the 3050. Log validation metrics and save the checkpoint (git-ignored). Note the run time. (2 h)
- [ ] [M] **Browser spike (risk reduction):** load any stock transformers.js text-classification model inside a Next.js Web Worker and print a prediction. Only proves the pipeline works. (1.5 h)

## Week 2: Trust layer and evaluation (Mon 23 – Sun 29 Nov) · ~7.5 h + 1 h Should

- [ ] [M] Write `ml/data/ood_val.txt` and `ml/data/ood_test.txt`, about 75 queries each. Mix clearly off-topic queries ("what's the weather") with banking-adjacent ones (loans, crypto, UPI, credit score), which are the hard cases. Never tune on `ood_test.txt`. (1.5 h)
- [ ] [M] `train.py --setting open`: DistilBERT on 67 intents. (0.5 h)
- [ ] [M] `calibrate.py`: fit temperature T on closed-set validation logits. Report ECE and a reliability diagram, before and after. (1.5 h)
- [ ] [M] Threshold for the **deployed (closed-set) model**: choose it on in-domain validation plus `ood_val.txt`. Pick an explicit operating point, e.g. "flag at most 5% of in-domain queries". Save T and the threshold to `results/deploy_config.json`. (1 h)
- [ ] [M] `evaluate.py`, run **once** on the test sets, writing `results/metrics.json` and plots. (2.5 h)
  - Closed-set: accuracy, macro-F1, top confused intent pairs, priority agreement, ECE after scaling
  - Near-unknown (open-set model): AUROC for detecting the 10 held-out intents
  - Far-unknown (deployed model): AUROC on `ood_test.txt`, and the share of `ood_test` flagged at the chosen threshold
  - Baseline row for comparison
- [ ] [S] MiniLM closed-set run through the same pipeline, for the size-vs-accuracy row. (1 h)
- [ ] [M] Lock results: commit `results/` and tag `v0.1-results`. Update the Decision Log. (0.5 h)

> **Gate:** don't start Week 3 UI work until `metrics.json` exists. ML results drive the app, not the other way round.

## Week 3: Export and live MVP (Mon 30 Nov – Sun 6 Dec) · ~9.5 h

- [ ] [M] `export_onnx.py`: export the closed-set DistilBERT to ONNX and quantize to int8. Report accuracy before vs after and file size in `metrics.json`. Push to the HF Hub in the layout transformers.js expects (follow its conversion guide). (2 h)
- [ ] [M] Parity fixtures: 20 fixed queries, with Python ONNX top-1 and probabilities saved to `results/parity.json`. (0.5 h)
- [ ] [M] `web/lib/worker.ts`: load the model with a progress callback, run `AutoModelForSequenceClassification` for raw logits, apply logits ÷ T, softmax, top 3. (2 h)
- [ ] [M] `web/lib/triage.ts`: threshold check, then priority, template and review flag. Unit tests for the edge cases (exactly at threshold, unknown intent key). (1.5 h)
- [ ] [M] Triage page (`/`): query box, sample buttons (including one off-topic), top-3 confidence bars, priority badge, suggested reply, a clear "Needs human review" state with its reason, and a first-load progress bar. (2.5 h)
- [ ] [M] Browser parity check against `parity.json`. All 20 top-1 intents must match. (0.5 h)
- [ ] [M] Deploy, so the **MVP is live**. (0.5 h)

## Week 4: Results, docs, polish (Mon 7 – Sun 13 Dec) · ~7.5 h + 3 h Should

- [ ] [M] Results page (`/results`), reading `results/metrics.json`: model comparison table, top confused pairs, reliability diagram, near- and far-unknown AUROC, model size, quantization drop. (2.5 h)
- [ ] [M] Measure in-browser latency: median over 50 queries after warm-up. Record your device in the README. (0.5 h)
- [ ] [M] About page (`/about`): what the data is, its limits (UK/EU banking, no UPI), simulated-demo notice, safety rules, Banking77 CC BY 4.0 attribution. (1 h)
- [ ] [S] Queue page (`/queue`): session-only list of submitted tickets, sorted and filterable by priority, with a small top-intents chart. No database. (1.5 h)
- [ ] [S] Input sensitivity: leave-one-word-out confidence drop, shown as highlights and labelled as sensitivity, not explanation. (1.5 h)
- [ ] [M] CI (GitHub Actions): pytest and ruff for `ml/`, lint and build for `web/`. (1 h)
- [ ] [M] README: problem, live link, demo GIF, architecture diagram, results table (from `metrics.json`), design decisions (why an encoder over an LLM, why browser inference, how the threshold was chosen), limitations, reproduction steps, attribution. (2 h)
- [ ] [M] Record the demo GIF or a 60–90 s video. (0.5 h)

## Buffer: (Mon 14 – Sun 20 Dec)
Finish anything that slipped first. Then optionally:
- [ ] [X] Energy-score unknown detection vs maximum softmax probability
- [ ] [X] Zero-shot LLM comparison row (accuracy, latency, cost per 1,000 queries)
- [ ] [X] Mobile layout polish and accessibility pass

---

## Risks

| Risk | Early signal | Response |
|---|---|---|
| CUDA/driver setup eats time | Week 0 runs past 3 h | Train on Colab or Kaggle and fix local later |
| transformers.js loading friction | Browser spike fails in Week 1 | You have 2 weeks of slack. Check the conversion guide. Fallback is FastAPI on Render |
| Python vs browser mismatch | Parity check fails | Check the tokenizer files, the quantized vs fp32 model, and that you're using raw logits, not pipeline softmax |
| Off-topic sets too easy | Far-unknown AUROC ≈ 1.0 | Add more banking-adjacent queries to the **val** set only. Keep the test set as written |
| Week runs short | Under 6 h logged by Thursday | Drop [S] items for that week. Don't skip evaluation |
| Exams overlap | — | Shift the schedule. The buffer week absorbs one slipped week |

## Decision log
Record every methodology or design choice here, with the date and the reason.

- 2026-11-12: Project chosen: TicketAid, single-team banking ticket triage on Banking77 (CC BY 4.0).
- 2026-11-12: Deployed model is the closed-set (77-intent) DistilBERT. The open-set (67-intent) model is evaluation only.
- 2026-11-12: Temperature and threshold for the deployed model are tuned on its own validation data plus `ood_val`. Test sets are used once.
- 2026-11-12: Priority is a rule-based recommendation. The metric is "priority agreement".
- 2026-11-12: Inference runs in the browser (ONNX int8 + transformers.js), so there's no backend, no API cost, and queries stay on the device.
- 2026-11-12: Suggested replies are fixed templates. They never ask for PINs, OTPs, passwords or account details.
