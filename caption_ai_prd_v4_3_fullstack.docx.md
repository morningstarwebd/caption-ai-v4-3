**Caption AI**

Product Requirements Document

**Architecture D+++ | v4.3 FULL STACK**

*"AI Pipeline \+ Backend API \+ Frontend UI — 100% Local"*

| Field | Value |
| :---- | :---- |
| Version | 4.3 — Architecture D+++ Full Stack |
| Status | Approved for Implementation |
| Languages | Bengali / English / Hindi / Hinglish |
| AI Pipeline | v4.3 (unchanged — adaptive, self-calibrating) |
| Backend | FastAPI \+ SQLite (local) |
| Frontend | React (CDN — no build step) |
| Storage | Local Filesystem only |
| Cloud/DB | NONE — 100% offline capable |
| Target | Deterministic \+ Adaptive \+ Zero-Surprise \+ Full UI |

# **1\. Product Overview**

Caption AI is a local-first, AI-powered video captioning tool that automatically generates millisecond-accurate, beautifully animated captions for videos in any language and bakes them into the final output.

## **1.1 Core Goals**

* Zero Cost — completely free, no paid subscription

* Maximum Accuracy — \~92–97%+ via adaptive pipeline \+ Groq \+ LLM \+ triple verification

* Millisecond-perfect Timing — WhisperX forced alignment with drift clamping

* Beautiful Animation — frame-by-frame professional rendering, zero visual glitches

* Any Device — automatically configures based on available RAM

* Adaptive — thresholds and models adjust to audio conditions automatically

* Deterministic — every decision has a defined, predictable outcome

* Observable — structured logs \+ metrics per chunk (enterprise-grade)

* Full Stack — web UI \+ REST API \+ local storage, no cloud required

## **1.2 Version Evolution — Why v4.3**

| Version | Key Change | Problem it Solved | Status |
| :---- | :---- | :---- | :---- |
| v1/v2/v3 | Dual ASR merge | Root cause of all problems — abandoned | ❌ Abandoned |
| v4.0 | Single ASR \+ free LLM | Removed merge complexity | ✅ Foundation |
| v4.1 | Semantic verification \+ alignment check | Added self-checking layers | ✅ Superseded |
| v4.2 | Dual scoring \+ deterministic fallback | Heuristic gaps → predictable decisions | ✅ Base |
| v4.3 | Adaptive thresholds \+ self-calibrating engine | Static thresholds fail on real-world audio | 🚀 CURRENT |
| v4.3 FS | Full Stack: Backend API \+ Frontend UI | No UI → hard to use; now web-based local app | 🆕 THIS DOC |

# **2\. Architecture D+++ — Core Philosophy**

| Version | Philosophy | What it means |
| :---- | :---- | :---- |
| v4.1 | Smart | System detects its own errors |
| v4.2 | Deterministic | System scores to decide, never guesses |
| v4.3 ★ | Adaptive \+ Deterministic | System reads audio context → calibrates thresholds → decides deterministically |
| v4.3 FS ★ | Adaptive \+ Deterministic \+ Full Stack | Above \+ web UI \+ API \+ local DB — fully usable product |

# **3\. v4.3 Complete AI Pipeline**

(Unchanged from v4.3 AI-only spec — full pipeline below)

| Complete Pipeline Flow VIDEO UPLOAD (via Web UI) ↓ AUDIO EXTRACTION (FFmpeg, cross-platform)  ← audio.py ↓ ★ QUALITY ESTIMATOR (SNR, speech rate, volume)  ← quality\_estimator.py \[NEW\] ↓ ★ ADAPTIVE THRESHOLD ENGINE  ← NEW v4.3 ↓ OVERLAP CHUNKING (Normal: 30s+2s | Strict: 15s+3s)  ← audio.py ↓ GROQ WHISPER large-v3 transcription  ← transcriber.py ↓  retry(max=2) on failure PREPROCESSOR (context-aware filler removal)  ← preprocessor.py ↓ ★ TOKEN NORMALIZATION  ← normalizer.py \[NEW\] ↓ LLM CORRECTION (Normal / Critical mode)  ← llm\_judge.py ↓  retry\_llm(max=1) on failure ★ LIGHTWEIGHT LM CHECK  ← lm\_check.py \[NEW\] ↓ DUAL SCORING \[normalized text\]  ← dual\_scorer.py ↓ ★ GLOBAL CONSISTENCY PASS  ← NEW v4.3 ↓ CHUNK MERGE  ← chunk\_merger.py ↓ ★ ORDER-SAFE PARALLEL MERGE  ← NEW v4.3 ↓ ★ SENTENCE SPLITTER v2  ← sentence\_splitter.py \[NEW\] ↓ ★ ROBUST LANGUAGE DETECTION  ← lang\_detector.py \[NEW\] ↓ WHISPERX FORCED ALIGNMENT  ← aligner.py ↓  retry\_alignment(max=1) ★ DETERMINISTIC RETRY MATRIX v2  ← NEW v4.3 ↓ ★ ALIGNMENT DRIFT CLAMP (MIN\_GAP=0.02s)  ← drift\_clamp.py \[NEW\] ↓ ★ MICRO-REALIGN (low score → window ±1s realign)  ← NEW v4.3 ↓ ★ EMBEDDING BATCH \+ CACHE  ← NEW v4.3 ↓ ★ STRUCTURED LOGS \+ METRICS  ← logger.py \[NEW\] ↓ RENDERER  ← renderer.py ↓ CAPTIONED VIDEO OUTPUT → stored locally → served via API → downloaded via UI |
| :---- |

# **4\. v4.3 AI Core Upgrades — Full Specification**

15 upgrades in total. Priority 1–10 are CORE. Priority 11–15 are PERFORMANCE \+ OBSERVABILITY. (Full spec unchanged from v4.3 AI PRD — implementation details below.)

## **4.1 Adaptive Threshold Engine (Priority 1\)**

| 🔁 1\. Adaptive Threshold Engine — quality\_estimator.py \[NEW\] কেন: Static thresholds (0.80/0.65/0.50) সব audio-তে সমান ভালো কাজ করে না। def measure\_audio\_quality(audio\_path: str) \-\> dict:     \# SNR (Signal-to-Noise Ratio) measurement     \# Speech rate (words per second) via pause detection     \# Volume RMS     return {'snr\_db': float, 'speech\_rate': float, 'volume\_rms': float} def adaptive\_thresholds(snr\_db: float, speech\_rate: float) \-\> dict:     base \= {'dual\_score': 0.80, 'align\_avg': 0.65, 'confidence': 0.50}     if snr\_db \< 10:  \# noisy         base\['dual\_score'\] \-= 0.05         base\['align\_avg'\] \-= 0.05     if speech\_rate \> 3.5:  \# fast speech         base\['align\_avg'\] \-= 0.03     return base |
| :---- |

| Condition | Effect | Reason |
| :---- | :---- | :---- |
| SNR \< 10 dB (noisy) | dual\_score −0.05, align\_avg −0.05 | Noisy audio naturally scores lower |
| Speech rate \> 3.5 w/s (fast) | align\_avg −0.03 | Fast speech harder to align precisely |
| Normal audio | Base thresholds unchanged | Default v4.2 values apply |

## **4.2 Robust Language Detection (Priority 2\)**

| 🌐 2\. Robust Language Detection — lang\_detector.py \[NEW\] কেন: Wrong model selection \= bad alignment. Auto-detect unreliable for short/mixed-language chunks. def detect\_language(tokens: list\[str\]) \-\> str:     en \= sum(1 for t in tokens if is\_english(t))     hi \= sum(1 for t in tokens if is\_hindi(t))     bn \= sum(1 for t in tokens if is\_bengali(t))     if en \> 0 and hi \> 0: return 'hinglish'     counts \= \[('english', en), ('hindi', hi), ('bengali', bn)\]     return max(counts, key=lambda x: x\[1\])\[0\] def detect\_language\_majority(chunks: list) \-\> str:     votes \= \[detect\_language(chunk.tokens) for chunk in chunks\]     return Counter(votes).most\_common(1)\[0\]\[0\] |
| :---- |

## **4.3 Token Normalization (Priority 3\)**

| 🧩 3\. Token Normalization — normalizer.py \[NEW\] কেন: Jaccard similarity punctuation এবং case differences-এ false low scores দেয়। def normalize\_text(text: str) \-\> str:     t \= text.lower()     t \= re.sub(r'\[^\\w\\s\]', ' ', t)     return ' '.join(t.split()) NUMERAL\_MAP \= {'0':'zero','1':'one','2':'two','3':'three','4':'four',                '5':'five','6':'six','7':'seven','8':'eight','9':'nine'} Apply normalize\_text() \+ normalize\_numerals() in dual\_scorer.py and alignment\_validator.py before scoring. |
| :---- |

## **4.4 Global Consistency Pass (Priority 4\)**

| 🧠 4\. Global Consistency Pass — after chunk merge, before sentence split কেন: Chunk-wise LLM decisions একই proper noun বা technical term কে ভিন্নভাবে spell করতে পারে। CONSISTENCY\_PROMPT \= ''' You are a text consistency checker. DO NOT rewrite or rephrase any sentence. ONLY ensure consistent spelling of proper nouns, names, technical terms. Return the text exactly as given, with only spelling consistency fixes. ''' CRITICAL: Word count diff check — if diff \> 5% → reject, use original. |
| :---- |

## **4.5 Deterministic Retry Matrix v2 (Priority 5\)**

| Failure Type | Retry 1 | Retry 2 | Final Action |
| :---- | :---- | :---- | :---- |
| Alignment fail | Same model, retry | Upgrade to wav2vec2-large | flag\_chunk() |
| Low confidence (avg\<0.5) | Strict mode (15s+3s chunks) | Strict \+ sentence split finer (8-10 words) | Mark dims, continue |
| LLM reject (dual score fail) | Use Groq raw text (immediate) | — (no retry) | Groq raw \= final |
| Groq API error | retry(max=2) | Raise error | Skip chunk, log |

## **4.6 Remaining Core Upgrades (Priority 6–10)**

| \# | Upgrade | File | Key Logic |
| :---- | :---- | :---- | :---- |
| 6 | Order-Safe Parallel Merge | chunk\_merger.py | sorted(processed, key=lambda c: c.index) after asyncio.gather() |
| 7 | Alignment Drift Clamp | drift\_clamp.py | MIN\_GAP=0.02s — shift word start forward if overlap detected |
| 8 | Sentence Splitter v2 | sentence\_splitter.py | Punct split → prosody pause \>0.4s → max 10 words fallback |
| 9 | Confidence → Micro-Realign | confidence.py | Score\<0.40 → ±1s local realign; 0.40–0.60 → dim render |
| 10 | Lightweight LM Check | lm\_check.py | Spell score \<0.50 → skip embedding, use Groq raw directly |

## **4.7 Performance \+ Observability (Priority 11–15)**

| \# | Upgrade | File | Key Detail |
| :---- | :---- | :---- | :---- |
| 11 | Embedding Batch \+ Cache | dual\_scorer.py, cache.py | batch\_size=16, hash-based in-memory cache |
| 12 | Smart Cache Invalidation | cache.py | key \= audio\_hash \+ PIPELINE\_VERSION \+ CONFIG\_HASH |
| 13 | Config Profiles | config.py | fast | balanced | accurate |
| 14 | Structured Logs per-chunk | logger.py | JSON: chunk, snr, thresholds, scores, action, ms |
| 15 | Metrics Dashboard | logger.py | llm\_acceptance\_rate, avg\_alignment\_score, flagged\_count |

# **5\. Deterministic Decision Flow (v4.3)**

| Decision Point | Threshold (v4.3) | Pass Action | Fail Action |
| :---- | :---- | :---- | :---- |
| Audio quality measure | Always runs | Set adaptive thresholds | Use base thresholds |
| Lightweight LM check | score ≥ 0.50 | Proceed to dual score | Use Groq raw (skip embedding) |
| Dual score check | adaptive dual\_score threshold | Use LLM output | Use Groq raw text |
| Global consistency pass | word\_diff ≤ 5% | Use consistent transcript | Use pre-consistency transcript |
| Alignment validation | adaptive align\_avg \+ low\<30% | Continue | Retry matrix v2 (3-tier) |
| Confidence gate (word) | score \< 0.40 | Micro-realign ±1s | Dim render |
| Confidence gate (chunk) | avg \< adaptive confidence | Reprocess strict (once) | Mark dims, continue |
| Order-safe merge | Always | Sorted by chunk.index | — |
| Drift clamp | Always | MIN\_GAP=0.02s enforced | — |

# **6\. Backend Architecture — FastAPI \+ SQLite (Local)**

The backend exposes a REST API \+ WebSocket server. Everything runs locally. No cloud. No Supabase. No Firebase. Storage is local filesystem. Database is SQLite (single file, zero setup).

## **6.1 Tech Stack**

| Layer | Technology | Why |
| :---- | :---- | :---- |
| Web Framework | FastAPI (Python) | Same language as AI pipeline — zero context switch |
| Database | SQLite via aiosqlite | Zero setup, single file, 100% local |
| File Storage | Local filesystem | No cloud — files saved to storage/ folder |
| Real-time Progress | WebSocket (FastAPI built-in) | Live progress updates to frontend without polling |
| Background Jobs | asyncio \+ BackgroundTasks | Non-blocking pipeline execution |
| CORS | FastAPI CORSMiddleware | Allow frontend on different port (dev) |
| Env Config | .env file (existing) | API keys, FFMPEG path, Groq key etc. |

## **6.2 REST API Endpoints**

| Method | Endpoint | Description | Response |
| :---- | :---- | :---- | :---- |
| POST | /api/jobs/upload | Upload video file, create job entry | { job\_id, status } |
| GET | /api/jobs/{job\_id} | Get job status \+ progress % | { job\_id, status, progress, error } |
| GET | /api/jobs/{job\_id}/download | Download captioned video file | File stream (mp4) |
| GET | /api/jobs | List all past jobs (from SQLite) | \[ { job\_id, filename, status, created\_at } \] |
| DELETE | /api/jobs/{job\_id} | Delete job \+ files from disk | { deleted: true } |
| GET | /api/health | Server health check | { status: 'ok', version: 'v4.3' } |

## **6.3 WebSocket — Real-time Progress**

| WebSocket: ws://localhost:8000/ws/{job\_id} Frontend connects to this WebSocket after uploading a video. Backend sends progress messages as the pipeline advances. Message format (JSON): {   'type': 'progress',   'job\_id': 'abc123',   'step': 'WHISPERX\_ALIGNMENT',   'percent': 72,   'message': 'Aligning chunk 6 of 8...' } On completion: { 'type': 'done', 'job\_id': 'abc123', 'output\_url': '/api/jobs/abc123/download' } On error: { 'type': 'error', 'job\_id': 'abc123', 'message': 'Groq API timeout on chunk 3' } |
| :---- |

## **6.4 SQLite Database Schema**

| Database: storage/caption\_ai.db  (auto-created on first run) Table: jobs CREATE TABLE IF NOT EXISTS jobs (   id          TEXT PRIMARY KEY,          \-- UUID   filename    TEXT NOT NULL,             \-- original video filename   status      TEXT NOT NULL DEFAULT 'pending',               \-- pending | processing | done | failed   progress    INTEGER DEFAULT 0,         \-- 0–100   current\_step TEXT,                    \-- e.g. 'WHISPERX\_ALIGNMENT'   input\_path  TEXT,                     \-- storage/uploads/{id}/input.mp4   output\_path TEXT,                     \-- storage/outputs/{id}/output.mp4   error\_msg   TEXT,                     \-- error message if failed   config\_profile TEXT DEFAULT 'balanced',   language    TEXT DEFAULT 'auto',   created\_at  TEXT DEFAULT (datetime('now')),   updated\_at  TEXT DEFAULT (datetime('now')) ); |
| :---- |

## **6.5 Backend File Structure**

| server/ — New backend directory server/ ├── main.py              \# FastAPI app entry point ├── database.py          \# SQLite connection \+ table init (aiosqlite) ├── models.py            \# Pydantic request/response schemas ├── pipeline\_runner.py   \# Async wrapper that runs the v4.3 AI pipeline ├── progress.py          \# WebSocket connection manager └── api/     ├── jobs.py          \# /api/jobs endpoints     └── health.py        \# /api/health endpoint |
| :---- |

## **6.6 server/main.py — Full Structure**

| server/main.py from fastapi import FastAPI from fastapi.middleware.cors import CORSMiddleware from fastapi.staticfiles import StaticFiles from server.api import jobs, health from server.database import init\_db app \= FastAPI(title='Caption AI', version='v4.3') app.add\_middleware(CORSMiddleware,     allow\_origins=\['\*'\], allow\_methods=\['\*'\], allow\_headers=\['\*'\]) @app.on\_event('startup') async def startup(): await init\_db() app.include\_router(jobs.router, prefix='/api') app.include\_router(health.router, prefix='/api') \# Serve frontend static files app.mount('/', StaticFiles(directory='frontend/dist', html=True)) \# Run: uvicorn server.main:app \--reload \--port 8000 |
| :---- |

## **6.7 server/pipeline\_runner.py — Async Pipeline Wrapper**

| pipeline\_runner.py — connects AI pipeline to backend import asyncio, uuid from server.database import update\_job\_status from server.progress import notify PIPELINE\_STEPS \= \[     ('AUDIO\_EXTRACT', 5),   ('QUALITY\_ESTIMATE', 10),     ('CHUNKING', 15),        ('GROQ\_TRANSCRIBE', 35),     ('LLM\_CORRECTION', 55),  ('ALIGNMENT', 75),     ('DRIFT\_CLAMP', 80),     ('RENDER', 95),     ('DONE', 100\) \] async def run\_pipeline(job\_id: str, input\_path: str, config: dict):     try:         for step\_name, percent in PIPELINE\_STEPS:             await update\_job\_status(job\_id, 'processing', percent, step\_name)             await notify(job\_id, step\_name, percent)             \# Call actual pipeline function for this step             await asyncio.to\_thread(execute\_step, step\_name, job\_id, input\_path)         await update\_job\_status(job\_id, 'done', 100, 'DONE')         await notify(job\_id, 'DONE', 100, done=True)     except Exception as e:         await update\_job\_status(job\_id, 'failed', error\_msg=str(e))         await notify(job\_id, 'ERROR', 0, error=str(e)) |
| :---- |

## **6.8 server/progress.py — WebSocket Manager**

| progress.py — manages WebSocket connections per job from fastapi import WebSocket import json class ConnectionManager:     def \_\_init\_\_(self):         self.connections: dict\[str, list\[WebSocket\]\] \= {}     async def connect(self, job\_id: str, ws: WebSocket):         await ws.accept()         self.connections.setdefault(job\_id, \[\]).append(ws)     async def disconnect(self, job\_id: str, ws: WebSocket):         self.connections.get(job\_id, \[\]).remove(ws)     async def broadcast(self, job\_id: str, message: dict):         for ws in self.connections.get(job\_id, \[\]):             await ws.send\_text(json.dumps(message)) manager \= ConnectionManager() async def notify(job\_id, step, percent, done=False, error=None):     msg \= {'type': 'done' if done else ('error' if error else 'progress'),            'job\_id': job\_id, 'step': step, 'percent': percent}     if error: msg\['message'\] \= error     await manager.broadcast(job\_id, msg) |
| :---- |

# **7\. Frontend Architecture — React (CDN, No Build Step)**

Simple, single-page React app served directly by FastAPI. No npm build process needed — React is loaded via CDN. User uploads a video, sees live progress, then downloads the captioned output.

## **7.1 Frontend Tech Stack**

| Layer | Technology | Why |
| :---- | :---- | :---- |
| UI Framework | React 18 (via CDN) | No build step — just open in browser |
| Styling | TailwindCSS (CDN) | Fast, utility-first — no CSS files to manage |
| HTTP Client | fetch() built-in | Zero dependency — browser native |
| Real-time | WebSocket (browser built-in) | Live progress from backend |
| Icons | Heroicons (SVG inline) | No package needed |
| Served by | FastAPI StaticFiles | Single server — no separate dev server |

## **7.2 Frontend Pages / Views**

| View | Route | What User Sees |
| :---- | :---- | :---- |
| Home / Upload | / | Drag & drop video upload \+ language selector \+ profile selector |
| Processing | /jobs/{id} | Live progress bar \+ step name \+ animated spinner |
| Done | /jobs/{id} (done) | Success message \+ Download button \+ processing stats |
| History | /history | List of all past jobs with status \+ re-download option |

## **7.3 Frontend File Structure**

| frontend/ — served as static files by FastAPI frontend/ ├── index.html           \# Main HTML shell — loads React \+ Tailwind from CDN ├── app.jsx              \# Root React component (type='text/babel') ├── components/ │   ├── UploadCard.jsx   \# Drag & drop video upload component │   ├── ProgressCard.jsx \# Live progress bar \+ step display │   ├── ResultCard.jsx   \# Download button \+ stats after done │   └── HistoryList.jsx  \# Past jobs table └── api.js               \# fetch() wrappers for all API calls |
| :---- |

## **7.4 frontend/index.html — Full Structure**

| frontend/index.html \<\!DOCTYPE html\> \<html lang='en'\> \<head\>   \<meta charset='UTF-8'\>   \<meta name='viewport' content='width=device-width, initial-scale=1.0'\>   \<title\>Caption AI v4.3\</title\>   \<\!-- Tailwind CSS CDN \--\>   \<script src='https://cdn.tailwindcss.com'\>\</script\>   \<\!-- React \+ ReactDOM CDN \--\>   \<script crossorigin src='https://unpkg.com/react@18/umd/react.production.min.js'\>\</script\>   \<script crossorigin src='https://unpkg.com/react-dom@18/umd/react-dom.production.min.js'\>\</script\>   \<\!-- Babel for JSX (no build step) \--\>   \<script src='https://unpkg.com/@babel/standalone/babel.min.js'\>\</script\> \</head\> \<body class='bg-gray-900 text-white min-h-screen'\>   \<div id='root'\>\</div\>   \<script type='text/babel' src='app.jsx'\>\</script\> \</body\> \</html\> |
| :---- |

## **7.5 frontend/app.jsx — Main App Component**

| app.jsx — root component with routing logic const { useState, useEffect, useRef } \= React; function App() {   const \[view, setView\] \= useState('upload');  // 'upload' | 'processing' | 'done' | 'history'   const \[jobId, setJobId\] \= useState(null);   const \[progress, setProgress\] \= useState(0);   const \[step, setStep\] \= useState('');   const \[outputUrl, setOutputUrl\] \= useState(null);   async function handleUpload(file, language, profile) {     const formData \= new FormData();     formData.append('file', file);     formData.append('language', language);     formData.append('profile', profile);     const res \= await fetch('/api/jobs/upload', { method: 'POST', body: formData });     const { job\_id } \= await res.json();     setJobId(job\_id);     setView('processing');     connectWebSocket(job\_id);   }   function connectWebSocket(id) {     const ws \= new WebSocket(\`ws://localhost:8000/ws/${id}\`);     ws.onmessage \= (e) \=\> {       const msg \= JSON.parse(e.data);       if (msg.type \=== 'progress') { setProgress(msg.percent); setStep(msg.step); }       if (msg.type \=== 'done') { setOutputUrl(msg.output\_url); setView('done'); ws.close(); }       if (msg.type \=== 'error') { setView('error'); ws.close(); }     };   }   return (     \<div className='max-w-2xl mx-auto p-6'\>       \<Header /\>       {view \=== 'upload' && \<UploadCard onUpload={handleUpload} onHistory={() \=\> setView('history')} /\>}       {view \=== 'processing' && \<ProgressCard progress={progress} step={step} /\>}       {view \=== 'done' && \<ResultCard outputUrl={outputUrl} onNew={() \=\> setView('upload')} /\>}       {view \=== 'history' && \<HistoryList onBack={() \=\> setView('upload')} /\>}     \</div\>   ); } ReactDOM.createRoot(document.getElementById('root')).render(\<App /\>); |
| :---- |

## **7.6 frontend/components/UploadCard.jsx**

| UploadCard.jsx — drag & drop upload \+ settings function UploadCard({ onUpload, onHistory }) {   const \[file, setFile\] \= useState(null);   const \[language, setLanguage\] \= useState('auto');   const \[profile, setProfile\] \= useState('balanced');   const \[dragging, setDragging\] \= useState(false);   const LANGUAGES \= \['auto','bengali','english','hindi','hinglish'\];   const PROFILES  \= \['fast','balanced','accurate'\];   return (     \<div className='bg-gray-800 rounded-2xl p-8 space-y-6'\>       {/\* Drag & Drop Zone \*/}       \<div onDrop={e \=\> { e.preventDefault(); setFile(e.dataTransfer.files\[0\]); }}            onDragOver={e \=\> { e.preventDefault(); setDragging(true); }}            className={\`border-2 border-dashed rounded-xl p-12 text-center cursor-pointer                        ${dragging ? 'border-blue-400 bg-blue-900/20' : 'border-gray-600'}\`}\>         \<p className='text-gray-300'\>Drag & drop video here, or click to browse\</p\>         {file && \<p className='text-green-400 mt-2'\>✓ {file.name}\</p\>}       \</div\>       {/\* Language \+ Profile selectors \*/}       \<div className='grid grid-cols-2 gap-4'\>         \<select value={language} onChange={e \=\> setLanguage(e.target.value)}                 className='bg-gray-700 rounded-lg p-3'\>           {LANGUAGES.map(l \=\> \<option key={l}\>{l}\</option\>)}         \</select\>         \<select value={profile} onChange={e \=\> setProfile(e.target.value)}                 className='bg-gray-700 rounded-lg p-3'\>           {PROFILES.map(p \=\> \<option key={p}\>{p}\</option\>)}         \</select\>       \</div\>       {/\* Upload button \*/}       \<button disabled={\!file} onClick={() \=\> onUpload(file, language, profile)}               className='w-full bg-blue-600 hover:bg-blue-500 disabled:opacity-40                          rounded-xl py-4 font-semibold text-lg transition'\>         Generate Captions       \</button\>       \<button onClick={onHistory} className='w-full text-gray-400 hover:text-white'\>         View History       \</button\>     \</div\>   ); } |
| :---- |

## **7.7 frontend/components/ProgressCard.jsx**

| ProgressCard.jsx — live progress bar function ProgressCard({ progress, step }) {   const STEP\_LABELS \= {     'AUDIO\_EXTRACT': 'Extracting audio...',     'QUALITY\_ESTIMATE': 'Measuring audio quality (SNR, speech rate)...',     'CHUNKING': 'Splitting audio into chunks...',     'GROQ\_TRANSCRIBE': 'Transcribing with Groq Whisper...',     'LLM\_CORRECTION': 'Correcting with LLM...',     'ALIGNMENT': 'Aligning timestamps with WhisperX...',     'DRIFT\_CLAMP': 'Clamping alignment drift...',     'RENDER': 'Rendering captions onto video...',     'DONE': 'Complete\!'   };   return (     \<div className='bg-gray-800 rounded-2xl p-8 space-y-6'\>       \<h2 className='text-xl font-semibold'\>Processing your video\</h2\>       {/\* Progress bar \*/}       \<div className='w-full bg-gray-700 rounded-full h-3'\>         \<div className='bg-blue-500 h-3 rounded-full transition-all duration-500'              style={{ width: \`${progress}%\` }} /\>       \</div\>       \<div className='flex justify-between text-sm text-gray-400'\>         \<span\>{STEP\_LABELS\[step\] || step}\</span\>         \<span\>{progress}%\</span\>       \</div\>       {/\* Spinner \*/}       \<div className='flex justify-center'\>         \<div className='w-10 h-10 border-4 border-blue-500 border-t-transparent                        rounded-full animate-spin' /\>       \</div\>     \</div\>   ); } |
| :---- |

## **7.8 frontend/components/ResultCard.jsx**

| ResultCard.jsx — download \+ new job function ResultCard({ outputUrl, onNew }) {   return (     \<div className='bg-gray-800 rounded-2xl p-8 space-y-6 text-center'\>       \<div className='text-6xl'\>✅\</div\>       \<h2 className='text-2xl font-bold text-green-400'\>Captions Ready\!\</h2\>       \<p className='text-gray-400'\>Your captioned video has been generated successfully.\</p\>       \<a href={outputUrl} download          className='block w-full bg-green-600 hover:bg-green-500 rounded-xl py-4                     font-semibold text-lg transition'\>         Download Captioned Video       \</a\>       \<button onClick={onNew}               className='w-full text-gray-400 hover:text-white transition'\>         Caption Another Video       \</button\>     \</div\>   ); } |
| :---- |

# **8\. Complete Project File Structure (v4.3 Full Stack)**

| caption\_ai/ — root project directory caption\_ai/ │ ├── .env                     \# API keys, FFMPEG path etc. (DO NOT share) │ ├── server/                  \# 🆕 Backend (NEW — Full Stack) │   ├── main.py              \# FastAPI app \+ startup │   ├── database.py          \# SQLite init \+ queries (aiosqlite) │   ├── models.py            \# Pydantic schemas │   ├── pipeline\_runner.py   \# Async AI pipeline wrapper │   ├── progress.py          \# WebSocket connection manager │   └── api/ │       ├── jobs.py          \# /api/jobs routes │       └── health.py        \# /api/health │ ├── frontend/                \# 🆕 Frontend (NEW — Full Stack) │   ├── index.html           \# HTML shell \+ CDN imports │   ├── app.jsx              \# Root React component │   ├── api.js               \# fetch() helpers │   └── components/ │       ├── UploadCard.jsx │       ├── ProgressCard.jsx │       ├── ResultCard.jsx │       └── HistoryList.jsx │ ├── ai\_pipeline/             \# 🔄 v4.3 AI Pipeline (existing \+ upgraded) │   ├── quality\_estimator.py  \# 🆕 NEW v4.3 │   ├── lang\_detector.py      \# 🆕 NEW v4.3 │   ├── normalizer.py         \# 🆕 NEW v4.3 │   ├── drift\_clamp.py        \# 🆕 NEW v4.3 │   ├── lm\_check.py           \# 🆕 NEW v4.3 │   ├── logger.py             \# 🆕 NEW v4.3 │   ├── sentence\_splitter.py  \# 🔄 UPGRADED v4.3 │   ├── dual\_scorer.py        \# 🔄 UPGRADED v4.3 │   ├── alignment\_validator.py\# 🔄 UPGRADED v4.3 │   ├── config.py             \# 🔄 UPGRADED v4.3 │   ├── chunk\_merger.py       \# 🔄 UPGRADED v4.3 │   ├── confidence.py         \# 🔄 UPGRADED v4.3 │   ├── alignment\_models.py   \# 🔄 UPGRADED v4.3 │   ├── audio.py              \# ✏️ MODIFIED │   ├── transcriber.py        \# ✅ FROM v4.2 │   ├── llm\_judge.py          \# ✅ FROM v4.2 │   ├── preprocessor.py       \# ✅ FROM v4.2 │   ├── aligner.py            \# ✅ FROM v4.2 │   ├── renderer.py           \# ✅ FROM v4.2 │   ├── cache.py              \# 🔄 UPGRADED v4.3 │   └── hallucination\_guard.py\# ✅ FROM v4.2 │ └── storage/                 \# 🆕 Local file storage (auto-created)     ├── caption\_ai.db        \# SQLite database     ├── uploads/             \# Input videos (one folder per job UUID)     ├── outputs/             \# Captioned output videos     └── logs/                \# Structured JSON logs (per chunk) |
| :---- |

# **9\. Central Config (config.py v4.3)**

| config.py — full configuration (AI pipeline \+ server) \# config.py — Caption AI v4.3 Full Stack PIPELINE\_VERSION \= 'v4.3' \# ── Server ──────────────────────────────────────────────────── SERVER\_HOST \= '127.0.0.1' SERVER\_PORT \= 8000 STORAGE\_DIR \= 'storage' UPLOAD\_DIR  \= 'storage/uploads' OUTPUT\_DIR  \= 'storage/outputs' DB\_PATH     \= 'storage/caption\_ai.db' MAX\_UPLOAD\_SIZE\_MB \= 500 \# ── Config Profile ───────────────────────────────────────────── PROFILE \= 'balanced'  \# 'fast' | 'balanced' | 'accurate' \# ── Base Thresholds (overridden by adaptive engine) ──────────── DUAL\_SCORE\_THRESHOLD   \= 0.80 ALIGN\_AVG\_THRESHOLD    \= 0.65 CONFIDENCE\_REPROCESS   \= 0.50 CONFIDENCE\_DIM         \= 0.60 CONFIDENCE\_MICRO\_REALIGN \= 0.40 \# ── Adaptive Engine Deltas ───────────────────────────────────── NOISY\_SNR\_THRESHOLD      \= 10 NOISY\_DUAL\_DELTA         \= \-0.05 NOISY\_ALIGN\_DELTA        \= \-0.05 FAST\_SPEECH\_THRESHOLD    \= 3.5 FAST\_SPEECH\_ALIGN\_DELTA  \= \-0.03 \# ── Scoring Weights ──────────────────────────────────────────── SEMANTIC\_WEIGHT          \= 0.60 KEYWORD\_WEIGHT           \= 0.40 ALIGN\_LOW\_RATIO\_MAX      \= 0.30 ALIGN\_WORD\_THRESHOLD     \= 0.50 \# ── Chunking ─────────────────────────────────────────────────── CHUNK\_SIZE\_NORMAL        \= 30 CHUNK\_OVERLAP\_NORMAL     \= 2 CHUNK\_SIZE\_STRICT        \= 15 CHUNK\_OVERLAP\_STRICT     \= 3 \# ── Sentence Split ───────────────────────────────────────────── SENTENCE\_MAX\_WORDS       \= 10 SENTENCE\_MAX\_WORDS\_STRICT \= 8 PROSODY\_PAUSE\_THRESHOLD  \= 0.4 \# ── Retry ────────────────────────────────────────────────────── RETRY\_GROQ  \= 2 RETRY\_LLM   \= 1 RETRY\_ALIGN \= 2 \# ── Alignment ────────────────────────────────────────────────── OVERLAP\_TOLERANCE        \= 0.10 DRIFT\_MIN\_GAP            \= 0.02 MICRO\_REALIGN\_WINDOW     \= 1.0 \# ── Embedding ────────────────────────────────────────────────── EMBED\_BATCH\_SIZE         \= 16 \# ── LLM ──────────────────────────────────────────────────────── LLM\_MODE                 \= 'normal' \# ── LM Check ─────────────────────────────────────────────────── LM\_CHECK\_MIN\_SCORE       \= 0.50 \# ── Consistency Pass ─────────────────────────────────────────── CONSISTENCY\_WORD\_DIFF\_MAX \= 0.05 \# ── Cache ────────────────────────────────────────────────────── CACHE\_DIR                \= 'storage/cache' \# ── Observability ────────────────────────────────────────────── LOG\_STRUCTURED           \= True LOG\_DIR                  \= 'storage/logs' |
| :---- |

# **10\. Dependencies & Installation**

## **10.1 requirements.txt (updated for Full Stack)**

| requirements.txt \# \=== AI Pipeline (v4.3) \=== whisperx faster-whisper groq sentence-transformers pyspellchecker librosa soundfile numpy langdetect \# \=== Backend (Full Stack — NEW) \=== fastapi uvicorn\[standard\] aiosqlite python-multipart     \# for file upload aiofiles             \# for async file I/O python-dotenv |
| :---- |

## **10.2 Setup Steps**

1. Clone/open project folder: cd caption\_ai

2. Create virtual environment: python \-m venv venv && source venv/bin/activate

3. Install dependencies: pip install \-r requirements.txt

4. Copy .env file (preserve existing — contains Groq API key, FFMPEG path etc.)

5. Run server: uvicorn server.main:app \--reload \--host 127.0.0.1 \--port 8000

6. Open browser: http://localhost:8000

## **10.3 First Run — Auto-Created Directories**

| On first startup, server auto-creates: storage/ storage/uploads/      ← uploaded videos go here storage/outputs/      ← captioned videos saved here storage/logs/         ← per-chunk JSON logs storage/cache/        ← embedding cache storage/caption\_ai.db ← SQLite database (auto-initialized) |
| :---- |

# **11\. Implementation Plan (Full Stack) — Priority Order**

| Priority | Task | Files | Notes |
| :---- | :---- | :---- | :---- |
| 1 | Backend setup | server/main.py, server/database.py | FastAPI app \+ SQLite init \+ storage dirs |
| 2 | Upload endpoint | server/api/jobs.py | POST /api/jobs/upload — save file, create DB record |
| 3 | Pipeline runner | server/pipeline\_runner.py | Async wrapper that calls AI pipeline steps |
| 4 | WebSocket progress | server/progress.py, server/api/jobs.py | ws://{job\_id} — live updates |
| 5 | Download endpoint | server/api/jobs.py | GET /api/jobs/{id}/download — stream output file |
| 6 | Frontend shell | frontend/index.html | HTML \+ CDN links (React, Tailwind, Babel) |
| 7 | Upload UI | frontend/components/UploadCard.jsx | Drag & drop \+ language \+ profile selector |
| 8 | Progress UI | frontend/components/ProgressCard.jsx | Progress bar \+ step label \+ WebSocket connect |
| 9 | Result UI | frontend/components/ResultCard.jsx | Download button \+ new job button |
| 10 | History UI | frontend/components/HistoryList.jsx | Past jobs list from GET /api/jobs |
| 11 | AI: Adaptive Threshold | quality\_estimator.py, config.py | Foundation for all other AI upgrades |
| 12 | AI: Language Detection | lang\_detector.py, alignment\_models.py | Wrong model \= wrong everything |
| 13 | AI: Token Normalization | normalizer.py, dual\_scorer.py | Fixes scoring accuracy |
| 14 | AI: Retry Matrix v2 | aligner.py, main.py | 3-tier escalation |
| 15 | AI: Drift Clamp | drift\_clamp.py, renderer.py | Zero visual glitches |
| 16 | AI: Remaining upgrades | See Section 4.6–4.7 | Order-safe merge, LM check, logs, cache etc. |

# **12\. v4.2 vs v4.3 Full Stack — Complete Comparison**

| Feature | v4.2 (AI only) | v4.3 Full Stack |
| :---- | :---- | :---- |
| Thresholds | Static (0.80/0.65/0.50) | ★ Adaptive — audio-aware auto-tune |
| Language detection | Basic auto-detect | ★ Chunk-level \+ majority vote \+ Hinglish |
| Token normalization | None | ★ lowercase \+ punct strip \+ numeral norm |
| Global consistency | None | ★ 1-pass LLM consistency pass |
| Retry matrix | 1-tier | ★ 3-tier (same → upgrade → flag) |
| Parallel merge order | Not guaranteed | ★ Order-safe with chunk.index sort |
| Drift clamping | None | ★ MIN\_GAP=0.02s enforced |
| Sentence splitter | Regex \+ max 13 words | ★ v2: prosody hints \+ max 10 words |
| Confidence action | Full reprocess or dim | ★ Micro-realign ±1s for word-level |
| LM pre-check | None | ★ Lightweight spell/perplexity |
| Embedding | Per-sentence | ★ Batch (16/call) \+ hash cache |
| Cache key | audio \+ version | ★ audio \+ version \+ CONFIG\_HASH |
| Config profiles | None | ★ fast / balanced / accurate |
| Logging | Warning logs only | ★ Structured JSON per-chunk |
| Metrics | None | ★ Acceptance rate, retry rate, scores |
| User Interface | NONE — terminal only | ★★ Web UI — upload, progress, download |
| API | NONE | ★★ REST API \+ WebSocket |
| Job History | NONE | ★★ SQLite — all past jobs browsable |
| Storage | Manual file paths | ★★ Organized local storage (uploads/outputs) |
| Database | NONE | ★★ SQLite — zero setup, 100% local |
| System type | Deterministic | ★★ Adaptive \+ Observable \+ Full Stack |

# **13\. Final Summary**

| v4.3 Full Stack \= v4.3 AI Pipeline \+ Adaptive threshold engine (audio-aware) \+ Robust language detection (Hinglish-safe) \+ Token normalization (stable scoring) \+ Global consistency pass (no term drift) \+ 3-tier retry matrix (full escalation path) \+ Alignment drift clamp (zero visual glitch) \+ Micro-realign (targeted fix, not full reprocess) \+ Embedding batch \+ smart cache \+ Config profiles (fast/balanced/accurate) \+ Structured logs \+ metrics (enterprise-grade) \+ FastAPI backend — REST API \+ WebSocket \+ SQLite local database — zero cloud dependency \+ React frontend — drag & drop upload \+ live progress \+ download \+ Local filesystem storage — organized uploads/outputs \+ Single command startup: uvicorn server.main:app \--reload "v4.3 Full Stack-এ system শুধু ঠিক না — নিজে নিজে পরিস্থিতি বুঝে best decision  নেয় — এবং এখন সেটা যেকোনো ব্যবহারকারী browser থেকে use করতে পারে।" |
| :---- |

| Goal | v4.3 Full Stack Answer |
| :---- | :---- |
| Adaptive to audio | SNR \+ speech rate → auto-tune all thresholds |
| Accurate Hinglish | Chunk-level detect \+ majority vote \+ xlsr-53 routing |
| Stable scoring | Token normalization before every score computation |
| No term drift | Global consistency pass after chunk merge |
| No silent failures | 3-tier retry \+ order-safe merge \+ flag\_chunk |
| Zero visual glitch | Drift clamp MIN\_GAP=0.02s always enforced |
| Fast at scale | Embedding batch=16 \+ hash cache \+ config profiles |
| Production debuggable | Structured JSON logs \+ aggregate metrics per session |
| Easy to use | Web UI — drag & drop video → live progress → download |
| 100% local | No cloud, no Supabase, no Firebase — SQLite \+ local filesystem |
| Single startup cmd | uvicorn server.main:app \--reload \--port 8000 |

**Caption AI — PRD v4.3 Full Stack — Architecture D+++**

*Adaptive • Self-Calibrating • Deterministic • Zero-Surprise • Full Stack • 100% Local*