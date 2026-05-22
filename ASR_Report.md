# ASR Shootout — Findings & Recommendation

*Benchmarking ASR for Indian conversational speech on a blue-collar hiring platform*

---

## TL;DR

For the platform's core job — capturing where a candidate says they live — **Deepgram nova-3 is the strongest choice today** (0.85 locality entity accuracy, robust across every condition and language). **Sarvam Saarika is the best value alternative**: nearly matching accuracy (0.75) at less than half the latency, and the best WER on clean Hindi. The single most important finding is a warning: **clean-benchmark WER does not predict real-world entity capture.** On open datasets, Sarvam and IndicConformer beat Deepgram on WER, yet on the actual locality task Deepgram wins. A team selecting a model from published WER numbers would pick the wrong one for this use case.

---

## 1. Setup

**Primary dataset — 20 self-recorded clips.** Bangalore locality names in natural sentences, varied across five acoustic conditions (quiet, noisy, phone, rushed, whispered) and four language framings (Hindi, Hinglish, Kannada, English). Four long tongue-twisters (Byatarayanapura, Rajarajeshwarinagar, Kadugondanahalli, Yelahanka) were included to stress entity capture.

**Supplementary datasets.** 20 samples each from Common Voice Hindi and Kathbath Hindi (AI4Bharat), to test whether clean-data rankings transfer to the real task.

**Models — five distinct approaches.**

| Model | Type | Rationale |
|---|---|---|
| Deepgram nova-3 | Commercial, global | Required baseline; production stack Vahan already runs |
| Sarvam Saarika v2.5 | Commercial, India-specific | Direct competitor built for Indian telephony |
| Whisper large-v3 | Open, multilingual | The open-source reference; second half of Vahan's real ASR stack |
| IndicConformer-600M | Open, India-specific, 22 langs | Conformer architecture; tests "did India-specific training help?" |
| IndicWav2Vec (Vakyansh) | Open, India-specific, Hindi-only | Lightweight wav2vec2 baseline; on-device speed test |

This spans commercial-vs-open, global-vs-India-specific, and three architectures (Whisper/transformer, Conformer, wav2vec2). Each clip's true language was routed to every model that accepts a language hint, so Kannada was decoded as Kannada, not forced to Hindi.

**Metrics.** Primary: **entity accuracy** — did the locality survive transcription? (fuzzy match, threshold 80). Because Hindi/Kannada models output native script, every transcript is romanized to a common Roman script before matching (a production normalization layer). We also report **WER**, **CER**, **latency / Real-Time Factor (RTF)**, and **mean entity distance on misses** (how badly a model fails when it fails). For the open datasets, which are general sentences, WER/CER is the meaningful metric.

---

## 2. Results

### 2.1 Your recordings — locality entity accuracy (primary)

| Model | Entity Acc ↑ | WER ↓ | CER ↓ | Latency p50 (s) | RTF ↓ |
|---|---|---|---|---|---|
| **Deepgram nova-3** | **0.85** | 0.74 | 0.21 | 1.49 | 0.31 |
| Whisper large-v3 | 0.80 | 0.87 | 0.30 | 1.16 | 0.25 |
| Sarvam Saarika | 0.75 | 0.83 | 0.29 | **0.63** | 0.14 |
| IndicConformer | 0.70 | 0.99 | 0.39 | 1.27 | 0.26 |
| IndicWav2Vec | 0.35 | 1.10 | 0.46 | 0.03 | **0.006** |

Deepgram leads, but the top four are within 15 points — and Sarvam delivers 0.75 at roughly half Deepgram's latency. IndicWav2Vec is ~50x faster than Deepgram but far less accurate: a stark speed/accuracy trade-off.

### 2.2 Open-source data — WER (lower is better)

| Model | Common Voice WER | Kathbath WER |
|---|---|---|
| Sarvam Saarika | **0.14** | 0.089 |
| IndicConformer | 0.197 | **0.055** |
| Deepgram nova-3 | 0.192 | 0.21 |
| IndicWav2Vec | 0.211 | 0.12 |
| Whisper large-v3 | 0.26 | 0.169 |

**The ranking inverts.** On clean benchmark audio, Sarvam and IndicConformer lead and Deepgram is mid-pack. On Kathbath, IndicConformer (0.055) is nearly **4x better** than Deepgram (0.21) — the opposite of the entity-accuracy ranking on real recordings.

---

## 3. The Headline Insight: benchmark WER misranks models for this task

A team picking an ASR from published Hindi WER would rank IndicConformer and Sarvam at the top and Deepgram in the middle. On the platform's *actual* job — extracting a locality from short, noisy, code-mixed phone speech — that ranking is **wrong**: Deepgram wins by 5–15 points of entity accuracy. Clean read-speech benchmarks reward fluent full-sentence transcription; the real use case rewards robust capture of a single entity under adverse conditions. **Evaluation must mirror the production task, not borrow academic leaderboards.**

---

## 4. Failure Analysis

40 total failures across all models. Patterns:

**Failure mode 1 — Code-switching (Hinglish) is the universal weak spot.**
Hinglish is the worst language category for every open/commercial model except Deepgram: Sarvam and Whisper both drop to 0.50 on Hinglish vs 0.75–1.0 elsewhere. Mid-sentence English terms get transcribed phonetically in Devanagari — e.g. "Whitefield" becomes व्हाइट फील्ड, "Electronic City" becomes इलेक्ट्रॉनिक सिटी — which survives romanization only approximately. For a platform where candidates constantly mix Hindi and English, this is the biggest accuracy risk.

**Failure mode 2 — Phone-quality audio.**
The production channel is the hardest. On "Hebbal" over phone audio, models produced "Hibble", "Hippal", "Hibal" — all phonetically close but below threshold. Deepgram and Sarvam (both telephony-tuned) held up best; the open models degraded most. Phone accounts for a disproportionate share of failures.

**Failure mode 3 — Script + transliteration loss on English terms.**
Hindi-trained models output Devanagari. We romanize all output uniformly, but English locality terms take a lossy round-trip (English → Devanagari → Roman-phonetic), so "Electronic City" → "ilektronik siti" scores below an exact match. This penalizes WER more than entity accuracy — which is exactly why entity accuracy (resolved against the known locality list) is the right primary metric.

**Failure mode 4 — IndicWav2Vec is Hindi-only by design.**
Its 0.35 score reflects that it cannot handle English-framed clips and degrades on phone audio (0.00 on phone). Its value is purely its 0.006 RTF — interesting only for ultra-low-latency edge use.

**Graceful vs catastrophic failure.** Mean entity distance on misses is tightly clustered (0.31–0.37) across all models — meaning failures are near-misses, not wild hallucinations. This is good news: most misses are recoverable by fuzzy-matching against the known list of localities.

---

## 5. Beyond accuracy — production considerations

| Factor | Deepgram | Sarvam | Whisper-L | IndicConformer | IndicWav2Vec |
|---|---|---|---|---|---|
| Entity accuracy | 0.85 | 0.75 | 0.80 | 0.70 | 0.35 |
| Latency p50 (s) | 1.49 | **0.63** | 1.16 | 1.27 | **0.03** |
| RTF | 0.31 | 0.14 | 0.25 | 0.26 | 0.006 |
| Deployment | API key | API key | Self-host GPU | Self-host GPU | Self-host GPU |
| Telephony-tuned | Yes | Yes | No | No | No |
| Languages | Multi | 11 Indic | Multilingual | 22 Indic | Hindi only |

All models run faster than real-time (RTF < 1), so all are viable for live calls. Sarvam offers the best latency among the accurate models — a strong fit for a live phone product where sub-second response matters.

---

## 6. Recommendation

1. **Ship today: Deepgram nova-3.** Highest entity accuracy (0.85), robust across all conditions and languages including Kannada, zero deployment effort. It is also Vahan's existing production ASR — these results validate that choice.

2. **Evaluate as the value alternative: Sarvam Saarika.** 0.75 entity accuracy at less than half Deepgram's latency, best clean-Hindi WER, and India-telephony-tuned. Worth a production A/B test — if its real-world entity accuracy closes the gap, the latency advantage makes it the better long-term pick.

3. **On-prem / cost-controlled path: IndicConformer.** Best clean-Hindi accuracy and no per-minute cost, but needs tuning for code-switched (Hinglish) input before it is production-ready.

4. **Regardless of model, add two layers:** (a) a script-normalization / transliteration step (already prototyped here) so native-script output matches the Roman locality DB; (b) a phonetic fuzzy-match against the *known* list of localities — the platform already has the candidate set, so exploit it to recover the near-misses that dominate the failure cases.

---

## 7. Limitations (honest caveats)

- **N=20, single speaker.** Per-language cells are 4–8 samples; treat rankings as directional, not precise. A multi-speaker held-out set (e.g. IndicVoices) is the necessary next step.
- **"Phone" condition was simulated** (speakerphone re-recording), not true VoIP-codec audio; real telephony degradation may be worse.
- **Latency is not strictly apples-to-apples** — API latencies include network time; local-model latencies are GPU compute only (Colab T4).
- **Transliteration of code-mixed English is lossy**, which depresses WER for Hindi-script models; entity accuracy is more robust because it resolves against the known locality list.
- **Open datasets are read/prompted speech**, not spontaneous phone speech — which is precisely why their WER ranking diverges from the real-world entity ranking. That divergence is itself a core finding.
- **No streaming benchmark.** All measurements are batch-mode; live-call first-token latency would refine the production picture.

---

*Code, all 20 recordings, and full results CSVs accompany this report. Every model was benchmarked against Deepgram as the baseline.*
