

| Caption AI Product Requirements Document Architecture D+++ | v4.3 *"Static Thresholds → Adaptive, Context-Aware, Self-Calibrating System"* |
| :---: |

| Field | Value |
| ----- | ----- |
| Version | 4.3 — Architecture D+++ (Enterprise Grade) |
| Status | Approved for Implementation |
| Languages | Bengali / English / Hindi / Hinglish |
| Replaces | v4.2 Architecture D++ |
| Core Upgrade | Deterministic System → Adaptive \+ Self-Calibrating System |
| Philosophy | "System শুধু ঠিক না — পরিস্থিতি বুঝে best decision নেয়" |
| Target | Deterministic \+ Adaptive \+ Zero-Surprise Outputs at Scale |

# **1\. Product Overview**

Caption AI is a local-first, AI-powered video captioning tool that automatically generates millisecond-accurate, beautifully animated captions for videos in any language and bakes them into the final output.

## **1.1 Core Goals**

* Zero Cost — completely free, no paid subscription

* Maximum Accuracy — \~92–97%+ via adaptive pipeline \+ Groq \+ LLM \+ triple verification

* Millisecond-perfect Timing — WhisperX forced alignment with drift clamping

* Beautiful Animation — frame-by-frame professional rendering, zero visual glitches

* Any Device — automatically configures based on available RAM

* Adaptive — thresholds and models adjust to audio conditions automatically

* Deterministic — every decision has a defined, predictable outcome (from v4.2)

* Observable — structured logs \+ metrics per chunk (enterprise-grade)

## **1.2 Version Evolution — Why v4.3**

| Version | Key Change | Problem it Solved | Status |
| ----- | ----- | ----- | ----- |
| v1/v2/v3 | Dual ASR merge | Root cause of all problems — abandoned | ❌ Abandoned |
| v4.0 | Single ASR \+ free LLM | Removed merge complexity | ✅ Foundation |
| v4.1 | Semantic verification \+ alignment check | Added self-checking layers | ✅ Superseded |
| v4.2 | Dual scoring \+ deterministic fallback | Heuristic gaps → predictable decisions | ✅ Base |
| v4.3 | Adaptive thresholds \+ self-calibrating engine | Static thresholds fail on real-world audio | 🚀 CURRENT |

# **2\. Architecture D+++ — Core Philosophy**

| Version | Philosophy | What it means |
| ----- | ----- | ----- |
| v4.1 | Smart | System detects its own errors |
| v4.2 | Deterministic | System scores to decide, never guesses |
| v4.3 ★ | Adaptive \+ Deterministic | System reads audio context → calibrates thresholds → decides deterministically |

v4.3 এর মূল পার্থক্য: v4.2 জানে কী করতে হবে। v4.3 জানে পরিস্থিতি বুঝে কী করতে হবে — তারপর সেটা deterministically করে।

# **3\. v4.3 Complete Pipeline**

| VIDEO UPLOAD |
| :---- |
|   ↓    ↓ |
| AUDIO EXTRACTION (FFmpeg, cross-platform)  *← audio.py* |
|   ↓    ↓ |
| **★ QUALITY ESTIMATOR (SNR, speech rate, volume)**  *← NEW v4.3 — quality\_estimator.py* |
|   ↓    ↓ |
| **★ ADAPTIVE THRESHOLD ENGINE (tune all thresholds per audio)**  *← NEW v4.3* |
|   ↓    ↓ |
| OVERLAP CHUNKING (Normal: 30s+2s | Strict: 15s+3s | Config Profile-aware)  *← audio.py* |
|   ↓    ↓ |
| GROQ WHISPER large-v3 — transcription only  *← transcriber.py* |
|   ↓    ↓  retry(max=2) on failure |
| PREPROCESSOR (context-aware filler removal)  *← preprocessor.py* |
|   ↓    ↓ |
| **★ TOKEN NORMALIZATION (lowercase \+ punctuation strip \+ numeral norm)**  *← NEW v4.3 — normalizer.py* |
|   ↓    ↓ |
| LLM CORRECTION (Normal / Critical mode)  *← llm\_judge.py* |
|   ↓    ↓  retry\_llm(max=1) on failure |
| **★ LIGHTWEIGHT LM CHECK (n-gram perplexity / spell score)**  *← NEW v4.3 — lm\_check.py* |
|   ↓    ↓ |
| DUAL SCORING \[normalized text\] — final\_score \= 0.6×semantic \+ 0.4×keyword  *← dual\_scorer.py* |
|   ↓    ↓ |
| score ≥ adaptive\_dual\_threshold → accept LLM output  *← adaptive threshold* |
| score \< threshold → use Groq raw text (deterministic fallback) |
|   ↓    ↓ |
| **★ GLOBAL CONSISTENCY PASS (1-pass LLM — normalize names/terms only)**  *← NEW v4.3* |
|   ↓    ↓ |
| CHUNK MERGE (tolerance-based dedup)  *← chunk\_merger.py* |
|   ↓    ↓ |
| **★ ORDER-SAFE PARALLEL MERGE (chunk index sort)**  *← NEW v4.3* |
|   ↓    ↓ |
| **★ SENTENCE SPLITTER v2 (punctuation \+ prosody hints \+ max 10-12 words)**  *← NEW v4.3 — sentence\_splitter.py* |
|   ↓    ↓ |
| **★ ROBUST LANGUAGE DETECTION (chunk-level \+ majority vote \+ Hinglish)**  *← NEW v4.3 — lang\_detector.py* |
|   ↓    ↓  Language → Model route |
| WHISPERX FORCED ALIGNMENT (sentence-wise)  *← aligner.py* |
|   ↓    ↓  retry\_alignment(max=1) |
| ALIGNMENT SCORING (adaptive thresholds)  *← alignment\_validator.py* |
|   ↓    ↓ |
| **★ DETERMINISTIC RETRY MATRIX v2 (3-tier: same → upgrade model)**  *← NEW v4.3* |
|   ↓    ↓ |
| **★ ALIGNMENT DRIFT CLAMP (MIN\_GAP=0.02s, no-overlap enforcement)**  *← NEW v4.3 — drift\_clamp.py* |
|   ↓    ↓ |
| **CONFIDENCE GATE → ★ MICRO-REALIGN (low score → window ±1s realign)**  *← NEW v4.3* |
|   ↓    ↓ |
| **★ EMBEDDING BATCH \+ CACHE (batch=16, hash-based cache)**  *← NEW v4.3* |
|   ↓    ↓ |
| **★ STRUCTURED LOGS \+ METRICS (per-chunk JSON)**  *← NEW v4.3 — logger.py* |
|   ↓    ↓ |
| RENDERER (confidence-aware dim, zero-overlap guaranteed)  *← renderer.py* |
|   ↓    ↓ |
| **CAPTIONED VIDEO OUTPUT** |

# **4\. v4.3 Core Upgrades — Full Specification**

15 upgrades in total. Priority 1–10 are CORE (must implement). Priority 11–15 are PERFORMANCE \+ OBSERVABILITY.

| 🔁  1\. Adaptive Threshold Engine *সব threshold dynamic — audio condition অনুযায়ী auto-tune* |
| ----- |
| কেন: Static thresholds (0.80 / 0.65 / 0.50) সব audio-তে সমান ভালো কাজ করে না। Noisy audio বা fast speech-এ strict threshold false rejection বাড়ায়। **নতুন file: quality\_estimator.py** def measure\_audio\_quality(audio\_path: str) \-\> dict:     \# SNR (Signal-to-Noise Ratio) measurement     \# Speech rate (words per second) via pause detection     \# Volume RMS     return {'snr\_db': float, 'speech\_rate': float, 'volume\_rms': float} **Adaptive threshold function:** def adaptive\_thresholds(snr\_db: float, speech\_rate: float) \-\> dict:     base \= {'dual\_score': 0.80, 'align\_avg': 0.65, 'confidence': 0.50}     if snr\_db \< 10:      \# noisy audio         base\['dual\_score'\] \-= 0.05         base\['align\_avg'\]  \-= 0.05     if speech\_rate \> 3.5: \# fast speech (words/sec)         base\['align\_avg'\]  \-= 0.03     return base **Condition** **Effect** **Reason** SNR \< 10 dB (noisy) dual\_score −0.05, align\_avg −0.05 Noisy audio naturally scores lower Speech rate \> 3.5 w/s (fast) align\_avg −0.03 Fast speech harder to align precisely Normal audio Base thresholds unchanged Default v4.2 values apply  |

| 🌐  2\. Robust Language Detection *auto → chunk-level \+ majority vote \+ Hinglish detection* |
| ----- |
| কেন: Wrong model selection \= bad alignment. Auto-detect unreliable for short or mixed-language chunks. **নতুন file: lang\_detector.py** def detect\_language(tokens: list\[str\]) \-\> str:     en \= sum(1 for t in tokens if is\_english(t))     hi \= sum(1 for t in tokens if is\_hindi(t))     bn \= sum(1 for t in tokens if is\_bengali(t))     if en \> 0 and hi \> 0: return 'hinglish'  \# token mix ratio     counts \= \[('english', en), ('hindi', hi), ('bengali', bn)\]     return max(counts, key=lambda x: x\[1\])\[0\] def detect\_language\_majority(chunks: list) \-\> str:     votes \= \[detect\_language(chunk.tokens) for chunk in chunks\]     return Counter(votes).most\_common(1)\[0\]\[0\]  \# majority vote **Method** **When Used** **Output** Token-level detection Per chunk Per-chunk language label Majority vote Across all chunks Final document language Hinglish detection en \> 0 AND hi \> 0 Route to xlsr-53 model *Update alignment\_models.py → auto-route based on detected language.* |

| 🧩  3\. Token Normalization Layer *scoring \+ alignment stable — Jaccard ভুল করে না* |
| :---- |
| কেন: Jaccard similarity punctuation এবং case differences-এ false low scores দেয়। 'Hello,' vs 'hello' আলাদা গণনা হয়। **নতুন file: normalizer.py** import re def normalize\_text(text: str) \-\> str:     t \= text.lower()     t \= re.sub(r'\[^\\w\\s\]', ' ', t)  \# strip punctuation     t \= ' '.join(t.split())            \# collapse whitespace     return t NUMERAL\_MAP \= {'0':'zero','1':'one','2':'two','3':'three','4':'four',                '5':'five','6':'six','7':'seven','8':'eight','9':'nine'} def normalize\_numerals(text: str) \-\> str:     return re.sub(r'\\b(\\d)\\b', lambda m: NUMERAL\_MAP\[m.group()\], text) normalize\_text() এবং normalize\_numerals() দুটোই apply করতে হবে dual\_scorer.py এবং alignment\_validator.py-এ — scoring করার আগে। |

| 🧠  4\. Global Consistency Pass *chunk-wise drift fix — names/terms consistent* |
| ----- |
| কেন: Chunk-wise LLM decisions একই proper noun বা technical term কে ভিন্নভাবে spell করতে পারে। যেমন একটা chunk-এ 'Dhaka' আরেকটায় 'Daka'। **Pipeline position: After chunk merge, BEFORE sentence split** CONSISTENCY\_PROMPT \= ''' You are a text consistency checker. DO NOT rewrite or rephrase any sentence. DO NOT add or remove any words. ONLY ensure consistent spelling of proper nouns, names, and technical terms across all sentences. Return the text exactly as given, with only spelling consistency fixes applied. ''' CRITICAL: Consistency pass output-এ word count diff check অবশ্যই রাখতে হবে (v4.0 থেকে আসা guard)। LLM 'no rewrite' instruction always follow করবে না। **Step** **Action** 1\. After chunk merge Send full transcript to LLM with consistency prompt 2\. LLM response Word count diff check: if diff \> 5% → reject, use original 3\. If accepted Pass normalized transcript to sentence splitter  |

| 🔄  5\. Deterministic Retry Matrix v2 *3-tier escalation — 1 retry সব case কভার করে না* |
| ----- |
| কেন: v4.2-তে retry matrix incomplete ছিল। একটাই retry, model upgrade path ছিল না। **Failure Type** **Retry 1** **Retry 2** **Final Action** Alignment fail Same model, retry Upgrade to wav2vec2-large flag\_chunk() Low confidence (avg\<0.5) Strict mode (15s+3s chunks) Strict \+ sentence split finer (8-10 words) Mark dims, continue LLM reject (dual score fail) Use Groq raw text (immediate) — (no retry) Groq raw \= final Groq API error retry(max=2) Raise error Skip chunk, log def align\_with\_escalation(text, audio, model\_size='base'):     seg \= align(text, audio, model=model\_size)     if validate\_alignment(seg): return seg     seg \= align(text, audio, model=model\_size)  \# retry 1: same     if validate\_alignment(seg): return seg     seg \= align(text, audio, model='large')     \# retry 2: upgrade     if validate\_alignment(seg): return seg     flag\_chunk(reason='alignment\_failed'); return None |

| 🧱  6\. Order-Safe Parallel Merge *parallel chunks → order guaranteed* |
| :---- |
| কেন: Async/parallel chunk processing-এ chunks out-of-order return করতে পারে। Subtle ordering bug যেটা reproduce করা কঠিন। \# Ensure each chunk carries its sequential index chunks \= \[Chunk(index=i, audio=...) for i, ... in enumerate(splits)\] \# After parallel processing — stable sort by index processed \= await asyncio.gather(\*\[process(c) for c in chunks\]) processed \= sorted(processed, key=lambda c: c.index) chunk.index টা pipeline শুরুতেই assign করতে হবে — processing-এর পরে নয়। |

| 🧮  7\. Alignment Drift Clamp *render safety — zero visual glitches* |
| ----- |
| কেন: WhisperX alignment কখনো edge overlap দিতে পারে। একটা word শেষ হওয়ার আগেই পরের word শুরু হলে renderer-এ visual glitch হয়। **নতুন file: drift\_clamp.py** MIN\_GAP \= 0.02  \# 20ms minimum gap between words def clamp\_alignment\_drift(words: list) \-\> list:     for i in range(1, len(words)):         if words\[i\]\['start'\] \< words\[i-1\]\['end'\] \+ MIN\_GAP:             words\[i\]\['start'\] \= words\[i-1\]\['end'\] \+ MIN\_GAP     return words **Parameter** **Value** **Meaning** MIN\_GAP 0.02s (20ms) Minimum gap between consecutive word timestamps Apply After alignment, before renderer Must run before any rendering step Effect Shift start forward if overlap Preserves end timestamps, adjusts start only  |

| 🧪  8\. Sentence Splitter v2 *prosody-aware \+ robust fallback* |
| ----- |
| কেন: Pure regex সবসময় যথেষ্ট না। Long sentences বা pause-heavy speech-এ alignment accuracy কমে। MAX\_WORDS\_V2 \= 10  \# tighter than v4.2's 13 def split\_sentences\_v2(text: str, pauses: list \= None) \-\> list\[str\]:     \# 1\. Punctuation split (multi-language)     raw \= re.split(r'\[.\!?।॥\]+', text)     sentences \= \[s.strip() for s in raw if s.strip()\]     \# 2\. Prosody hints — split at detected pauses \> 0.4s     if pauses:         sentences \= split\_at\_pauses(sentences, pauses, min\_pause=0.4)     \# 3\. Max-word fallback (10 words, tighter than v4.2)     result \= \[\]     for s in sentences:         words \= s.split()         if len(words) \<= MAX\_WORDS\_V2: result.append(s)         else:             for i in range(0, len(words), MAX\_WORDS\_V2):                 result.append(' '.join(words\[i:i+MAX\_WORDS\_V2\]))     return result **Split Strategy** **Priority** **When Applied** Punctuation (. \! ? । ॥) 1st Always Prosody pause \> 0.4s 2nd If alignment data available Max 10-12 words fallback 3rd (safety net) Always, overrides long sentences  |

| 📊  9\. Confidence → Action (Micro-Realign) *low score → local fix, not full reprocess* |
| ----- |
| কেন: v4.2-তে avg confidence \< 0.50 হলে পুরো chunk reprocess হতো। অনেক ক্ষেত্রে শুধু কয়েকটা word-এ problem, full reprocess overkill। def confidence\_action(segments: list, chunk\_audio) \-\> list:     for i, seg in enumerate(segments):         if seg.get('score', 1.0) \< 0.40:             \# Micro-realign: only ±1s window around this word             seg \= local\_realign(seg, chunk\_audio, window=1.0)             segments\[i\] \= seg         elif seg.get('score', 1.0) \< 0.60:             seg\['render\_style'\] \= 'dim'  \# visual fallback     return segments **Score Range** **Action** **Cost** \< 0.40 Micro-realign (±1s window) Low — only small window 0.40 – 0.60 Dim in renderer Zero — visual only \> 0.60 Normal render Zero  |

| 🧠  10\. Lightweight LM Check *fast sanity before heavy embedding* |
| ----- |
| কেন: Sentence embedding (MiniLM) heavy। একটা cheap pre-check দিয়ে obvious errors আগে ধরলে embedding call কমে। def lightweight\_lm\_check(text: str) \-\> float:     \# Option 1: Simple spell score (pyspellchecker)     words \= text.split()     correct \= sum(1 for w in words if spell.correction(w) \== w)     spell\_score \= correct / len(words) if words else 1.0     \# Option 2: Bigram perplexity (fast, no GPU)     \# Returns score 0.0–1.0 (higher \= more natural text)     return spell\_score  \# or combine with perplexity LM check score \< 0.50 → skip embedding, directly use Groq raw text. Score ≥ 0.50 → proceed to dual scoring (embedding). **Check** **Tool** **Speed** **Catches** Spell score pyspellchecker Fast (CPU) Garbled words, obvious errors N-gram perplexity Custom bigram model Fast (CPU) Unnatural word sequences  |

## **4.2 Performance & Scale Upgrades**

| 🧵  11\. Embedding Batching \+ Caching *8-16 sentences per batch \+ hash-based cache* |
| :---- |
| কেন: Per-sentence embedding call \= N × model forward pass। Batch করলে throughput বাড়ে। \# Batch encoding def batch\_encode(texts: list\[str\], batch\_size: int \= 16\) \-\> list:     return embed\_model.encode(texts, batch\_size=batch\_size,                              convert\_to\_tensor=True) \# Hash-based cache import hashlib, json, os EMBED\_CACHE \= {} def get\_embedding(text: str):     key \= hashlib.md5(text.encode()).hexdigest()     if key in EMBED\_CACHE: return EMBED\_CACHE\[key\]     emb \= embed\_model.encode(text, convert\_to\_tensor=True)     EMBED\_CACHE\[key\] \= emb     return emb |

| 🗃️  12\. Smart Cache Invalidation *config change হলে cache auto-invalidate* |
| :---- |
| কেন: v4.2 cache key শুধু audio hash \+ pipeline version। Config change (threshold tune) হলে old cache serve হতো। import hashlib, json PIPELINE\_VERSION \= 'v4.3' CONFIG\_HASH \= hashlib.md5(     json.dumps(CONFIG, sort\_keys=True).encode() ).hexdigest()\[:8\] def get\_cache\_key(audio\_path: str) \-\> str:     audio\_hash \= hashlib.md5(open(audio\_path,'rb').read()).hexdigest()     return f'{audio\_hash}\_{PIPELINE\_VERSION}\_{CONFIG\_HASH}' |

| 🧰  13\. Config Profiles *fast | balanced | accurate* |
| ----- |
| কেন: Different use cases need different speed/accuracy tradeoffs। Developer testing vs production vs high-stakes content। **Profile** **Disables** **Enables** **Use Case** fast Embedding check, double-pass — Development, testing balanced Double-pass Embedding check Standard production (default) accurate — Double-pass, strict thresholds High-stakes, broadcast PROFILE \= 'balanced'  \# 'fast' | 'balanced' | 'accurate' if PROFILE \== 'fast':     DISABLE\_EMBEDDING\_CHECK \= True     LLM\_MODE \= 'normal' elif PROFILE \== 'accurate':     LLM\_MODE \= 'critical'     SENTENCE\_MAX\_WORDS \= 8     DUAL\_SCORE\_THRESHOLD \= 0.85  \# stricter |

## **4.3 Observability Upgrades**

| 📝  14\. Structured Logs (per-chunk JSON) *enterprise-grade observability* |
| :---- |
| কী: প্রতিটা chunk-এর জন্য structured JSON log — scores, thresholds, decisions সব। \# Per-chunk structured log format {   'chunk': 3,   'audio\_snr': 12.4,   'speech\_rate': 2.8,   'adaptive\_thresholds': {'dual\_score': 0.80, 'align\_avg': 0.65},   'dual\_score': 0.78,   'action': 'fallback\_groq',   'align\_avg': 0.71,   'align\_low\_ratio': 0.12,   'align\_pass': true,   'confidence\_avg': 0.68,   'retry\_count': 0,   'processing\_ms': 1240 } |

| 📈  15\. Metrics Dashboard (aggregate) *acceptance rate, retry rate, avg scores* |
| :---- |
| কী: Session শেষে aggregate metrics — debugging এবং optimization এর জন্য। class PipelineMetrics:     def \_\_init\_\_(self):         self.total\_chunks \= 0         self.llm\_accepted \= 0         self.llm\_rejected \= 0         self.align\_scores \= \[\]         self.retry\_counts \= \[\]         self.flagged\_chunks \= \[\]     def summary(self) \-\> dict:         return {             'llm\_acceptance\_rate': self.llm\_accepted / self.total\_chunks,             'avg\_alignment\_score': mean(self.align\_scores),             'avg\_retry\_rate': mean(self.retry\_counts),             'flagged\_count': len(self.flagged\_chunks)         } |

# **5\. Deterministic Decision Flow (v4.3)**

v4.3-এ প্রতিটা decision point-এ adaptive threshold ব্যবহার হয় — static নয়। বাকি সব deterministic logic v4.2 থেকে inherit করা।

| Decision Point | Threshold (v4.3) | Pass Action | Fail Action |
| ----- | ----- | ----- | ----- |
| Audio quality measure | Always runs | Set adaptive thresholds | Use base thresholds |
| Lightweight LM check | score ≥ 0.50 | Proceed to dual score | Use Groq raw (skip embedding) |
| Dual score check | adaptive dual\_score threshold | Use LLM output | Use Groq raw text |
| Global consistency pass | word\_diff ≤ 5% | Use consistent transcript | Use pre-consistency transcript |
| Alignment validation | adaptive align\_avg \+ low\<30% | Continue | Retry matrix v2 (3-tier) |
| Confidence gate (word) | score \< 0.40 | Micro-realign ±1s | Dim render |
| Confidence gate (chunk) | avg \< adaptive confidence | Reprocess strict (once) | Mark dims, continue |
| Order-safe merge | Always | Sorted by chunk.index | — |
| Drift clamp | Always | MIN\_GAP=0.02s enforced | — |

# **6\. Project File Structure (v4.3)**

| File | Status | Purpose |
| ----- | ----- | ----- |
| quality\_estimator.py | 🆕 NEW v4.3 | SNR, speech rate, volume measurement |
| lang\_detector.py | 🆕 NEW v4.3 | Chunk-level language detect \+ majority vote \+ Hinglish |
| normalizer.py | 🆕 NEW v4.3 | Token normalization: lowercase \+ punct strip \+ numeral norm |
| drift\_clamp.py | 🆕 NEW v4.3 | Alignment drift clamp — MIN\_GAP enforcement |
| lm\_check.py | 🆕 NEW v4.3 | Lightweight LM check (spell score / n-gram perplexity) |
| logger.py | 🆕 NEW v4.3 | Structured per-chunk JSON logging \+ aggregate metrics |
| sentence\_splitter.py | 🔄 UPGRADED v4.3 | v2: prosody hints \+ tighter max-words (10) |
| dual\_scorer.py | 🔄 UPGRADED v4.3 | Now uses normalized text before scoring |
| alignment\_validator.py | 🔄 UPGRADED v4.3 | Now uses adaptive thresholds |
| config.py | 🔄 UPGRADED v4.3 | Profiles (fast/balanced/accurate) \+ adaptive params \+ v4.3 |
| chunk\_merger.py | 🔄 UPGRADED v4.3 | Order-safe merge with chunk.index sort |
| confidence.py | 🔄 UPGRADED v4.3 | Micro-realign instead of full reprocess for word-level |
| alignment\_models.py | 🔄 UPGRADED v4.3 | Auto-route via lang\_detector output |
| main.py | ✏️ MODIFIED | Global consistency pass step added after merge |
| audio.py | ✏️ MODIFIED | Profile-aware chunking params |
| dual\_scorer.py | ✅ FROM v4.2 | Semantic \+ keyword — now with normalization |
| hallucination\_guard.py | ✅ FROM v4.2 | Word count diff (unchanged) |
| aligner.py | ✅ FROM v4.2 | WhisperX sentence-wise (unchanged) |
| renderer.py | ✅ FROM v4.2 | Confidence-aware dim (unchanged) |
| retry.py | ✅ FROM v4.2 | Pipeline-wide retry wrapper (unchanged) |
| cache.py | 🔄 UPGRADED v4.3 | New key \= audio\_hash \+ PIPELINE\_VERSION \+ CONFIG\_HASH |

# **7\. Central Config (config.py v4.3)**

\# config.py — Caption AI v4.3 Central Configuration

PIPELINE\_VERSION \= 'v4.3'

\# ── Config Profile ────────────────────────────────────────────

PROFILE \= 'balanced'  \# 'fast' | 'balanced' | 'accurate'

\# ── Base Thresholds (overridden by adaptive engine) ───────────

DUAL\_SCORE\_THRESHOLD     \= 0.80

ALIGN\_AVG\_THRESHOLD      \= 0.65

CONFIDENCE\_REPROCESS     \= 0.50

CONFIDENCE\_DIM           \= 0.60

CONFIDENCE\_MICRO\_REALIGN \= 0.40  \# NEW v4.3

\# ── Adaptive Engine Deltas ────────────────────────────────────

NOISY\_SNR\_THRESHOLD      \= 10    \# dB — below this \= noisy

NOISY\_DUAL\_DELTA         \= \-0.05

NOISY\_ALIGN\_DELTA        \= \-0.05

FAST\_SPEECH\_THRESHOLD    \= 3.5   \# words/sec

FAST\_SPEECH\_ALIGN\_DELTA  \= \-0.03

\# ── Scoring Weights ───────────────────────────────────────────

SEMANTIC\_WEIGHT \= 0.60

KEYWORD\_WEIGHT  \= 0.40

ALIGN\_LOW\_RATIO\_MAX      \= 0.30

ALIGN\_WORD\_THRESHOLD     \= 0.50

\# ── Chunking ──────────────────────────────────────────────────

CHUNK\_SIZE\_NORMAL   \= 30

CHUNK\_OVERLAP\_NORMAL \= 2

CHUNK\_SIZE\_STRICT   \= 15

CHUNK\_OVERLAP\_STRICT \= 3

\# ── Sentence Split ────────────────────────────────────────────

SENTENCE\_MAX\_WORDS      \= 10   \# v4.3: tighter than v4.2's 13

SENTENCE\_MAX\_WORDS\_STRICT \= 8

PROSODY\_PAUSE\_THRESHOLD \= 0.4  \# seconds

\# ── Retry ─────────────────────────────────────────────────────

RETRY\_GROQ  \= 2

RETRY\_LLM   \= 1

RETRY\_ALIGN \= 2  \# v4.3: 2 retries (was 1 in v4.2)

\# ── Alignment ─────────────────────────────────────────────────

OVERLAP\_TOLERANCE \= 0.10

DRIFT\_MIN\_GAP     \= 0.02  \# NEW v4.3 — 20ms

MICRO\_REALIGN\_WINDOW \= 1.0  \# NEW v4.3 — seconds

\# ── Embedding ─────────────────────────────────────────────────

EMBED\_BATCH\_SIZE \= 16  \# NEW v4.3

\# ── LLM Modes ─────────────────────────────────────────────────

LLM\_MODE \= 'normal'  \# 'normal' | 'critical'

\# ── LM Check ──────────────────────────────────────────────────

LM\_CHECK\_MIN\_SCORE \= 0.50  \# NEW v4.3

\# ── Consistency Pass ──────────────────────────────────────────

CONSISTENCY\_WORD\_DIFF\_MAX \= 0.05  \# 5% NEW v4.3

\# ── Cache ─────────────────────────────────────────────────────

CACHE\_DIR \= 'storage/cache'

\# ── Observability ─────────────────────────────────────────────

LOG\_STRUCTURED \= True   \# NEW v4.3

LOG\_DIR \= 'storage/logs'  \# NEW v4.3

# **8\. v4.2 vs v4.3 — Full Comparison**

| Feature | v4.2 | v4.3 |
| ----- | ----- | ----- |
| Thresholds | Static (0.80/0.65/0.50) | ★ Adaptive — audio-aware auto-tune |
| Language detection | Basic auto-detect | ★ Chunk-level \+ majority vote \+ Hinglish |
| Token normalization | None (raw text) | ★ lowercase \+ punct strip \+ numeral norm |
| Global consistency | None | ★ 1-pass LLM consistency pass (names/terms) |
| Retry matrix | 1-tier (retry same) | ★ 3-tier (same → upgrade model → flag) |
| Parallel merge order | Not guaranteed | ★ Order-safe with chunk.index sort |
| Drift clamping | None | ★ MIN\_GAP=0.02s enforced |
| Sentence splitter | Regex \+ max 13 words | ★ v2: prosody hints \+ max 10 words |
| Confidence action | Full reprocess or dim | ★ Micro-realign ±1s for word-level |
| LM pre-check | None | ★ Lightweight spell/perplexity before embedding |
| Embedding | Per-sentence | ★ Batch (16/call) \+ hash cache |
| Cache key | audio \+ version | ★ audio \+ version \+ CONFIG\_HASH |
| Config profiles | None | ★ fast / balanced / accurate |
| Logging | Warning logs only | ★ Structured JSON per-chunk |
| Metrics | None | ★ Acceptance rate, retry rate, avg scores |
| System type | Deterministic | ★ Deterministic \+ Adaptive \+ Observable |

# **9\. Expected Accuracy (v4.3)**

| Content Type | v4.2 | v4.3 | Key Improvement |
| ----- | ----- | ----- | ----- |
| Pure Bengali | 94–97%+ | 95–97%+ | Normalization \+ better lang detection |
| Pure English | 95–98%+ | 95–98%+ | Minimal change (already strong) |
| Pure Hindi | 91–95%+ | 92–96%+ | Adaptive threshold for Hindi speech rate |
| Hinglish (mixed) | 88–93%+ | 90–94%+ | ★ Biggest gain — reliable Hinglish detect \+ xlsr-53 |
| Noisy Audio | 83–90%+ | 86–92%+ | ★ Biggest gain — adaptive threshold for low SNR |
| Fast Speech | 85–91%+ | 87–93%+ | Adaptive align threshold for high speech rate |

v4.3-এর সবচেয়ে বড় gain: Hinglish এবং Noisy Audio। Static threshold এই দুটো case-এ v4.2 তে false reject/accept বেশি করতো।

# **10\. Implementation Plan (v4.3) — Priority Order**

| Priority | Upgrade | File(s) | Why First |
| ----- | ----- | ----- | ----- |
| 1 | Adaptive Threshold Engine | quality\_estimator.py, config.py | Foundation for all other improvements |
| 2 | Language Detection \+ Routing | lang\_detector.py, alignment\_models.py | Wrong model \= wrong everything |
| 3 | Token Normalization | normalizer.py, dual\_scorer.py | Fixes scoring before any other tune |
| 4 | Retry Matrix v2 | aligner.py, main.py | Alignment is most common failure point |
| 5 | Order-Safe Merge | chunk\_merger.py | Silent bug — must fix before scale |
| 6 | Drift Clamp | drift\_clamp.py, renderer.py | Visual quality — user-visible immediately |
| 7 | Embedding Batch \+ Cache | dual\_scorer.py, cache.py | Performance — noticeable on long videos |
| 8 | Global Consistency Pass | main.py, llm\_judge.py | Polish — important for professional output |
| 9 | Confidence → Micro-Realign | confidence.py | Targeted fix, reduces full reprocess |
| 10 | Observability (Logs \+ Metrics) | logger.py, main.py | Needed for production debugging |
| 11 | Sentence Splitter v2 | sentence\_splitter.py | Robustness improvement |
| 12 | Lightweight LM Check | lm\_check.py | Speed optimization |
| 13 | Config Profiles | config.py | Developer experience |
| 14 | Smart Cache Invalidation | cache.py | Production safety |
| 15 | Metrics Dashboard | logger.py | Post-launch analysis |

# **11\. Final Summary**

| v4.3 \= v4.2 (deterministic foundation) \+ Adaptive threshold engine (audio-aware) \+ Robust language detection (Hinglish-safe) \+ Token normalization (stable scoring) \+ Global consistency pass (no term drift) \+ 3-tier retry matrix (full escalation path) \+ Alignment drift clamp (zero visual glitch) \+ Micro-realign (targeted fix, not full reprocess) \+ Embedding batch \+ smart cache \+ Config profiles (fast/balanced/accurate) \+ Structured logs \+ metrics (enterprise-grade) *"v4.3 এ system শুধু ঠিক না — নিজে নিজে পরিস্থিতি বুঝে best decision নেয়।"* |
| :---: |

| Goal | v4.3 Answer |
| ----- | ----- |
| Adaptive to audio | SNR \+ speech rate → auto-tune all thresholds |
| Accurate Hinglish | Chunk-level detect \+ majority vote \+ xlsr-53 routing |
| Stable scoring | Token normalization before every score computation |
| No term drift | Global consistency pass after chunk merge |
| No silent failures | 3-tier retry \+ order-safe merge \+ flag\_chunk |
| Zero visual glitch | Drift clamp MIN\_GAP=0.02s always enforced |
| Fast at scale | Embedding batch=16 \+ hash cache \+ config profiles |
| Production debuggable | Structured JSON logs \+ aggregate metrics per session |

**Caption AI — PRD v4.3 — Architecture D+++**  
*Adaptive • Self-Calibrating • Deterministic • Zero-Surprise • Enterprise-Grade*