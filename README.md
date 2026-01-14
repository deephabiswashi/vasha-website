### Vasha Website – Multilingual ASR, MT & TTS Web App

This project is the **web application** for Vasha AI: an end‑to‑end pipeline that converts **speech → text → translation → speech** with authentication and user-facing docs. It pairs a **React + Vite + TypeScript** frontend with a **FastAPI + Python** backend for ASR, machine translation, text‑to‑speech, and secure user management.

### 1. Architecture Overview

- **Frontend (`frontend/`)**
  - Vite + React + TypeScript, Tailwind CSS, shadcn-ui components
  - Main pages in `src/pages/`:
    - `Index` (landing/hero), `Chat` (ASR + chat), `MT` (machine translation), `TTS` (text‑to‑speech)
    - `UserDocs` (user manual), `DevDocs` (developer profiles/resources), `NotFound`
  - Service layer in `src/services/`:
    - `asrService.ts` → calls `/languages`, `/asr/models`, `/asr/upload`, `/asr/youtube`, `/asr/microphone`
    - `mtService.ts` → calls `/mt/translate`
    - `ttsService.ts` → calls `/tts/generate` and `/tts/audio/{filename}`
  - Auth UI (`src/components/auth/`) + `AuthContext` and Firebase config for OTP / captcha flows

- **Backend (`backend/`)**
  - `main.py` FastAPI application with:
    - User auth (signup + email OTP, SMS OTP, captcha login, Firebase phone verification, JWT tokens)
    - Chat history persistence with MongoDB
    - ASR, MT, and TTS REST endpoints
  - ASR / LID:
    - `asr_pipeline.py` (not shown here, used via `run_asr_with_fallback`)
    - `lid.py` for **language identification** based on Whisper or MMS LID (`TARGET_LANGS` covers 20+ Indic + global languages)
  - Machine Translation:
    - `mt.py` with Google Translate, IndicTrans2, and Meta NLLB‑200 + automatic fallback logic
  - Text‑to‑Speech:
    - `tts_handler.ts` + `indic_tts.py`, `tts_gtts.py`, `xtts.py` to combine **Indic Parler‑TTS**, **Coqui XTTS**, and **gTTS** with auto‑fallback
  - Additional docs in this folder (`ASR_README.md`, `AUTHENTICATION_FLOW.md`, cloud deployment guides)

### 2. Core User Features

- **ASR Chat (Speech → Text)**
  - Input options in `Chat` page:
    - Microphone recording via `AudioRecorder` (uploads `.webm`, backend converts to WAV)
    - File upload (`.wav`, `.mp3`, `.mp4`, `.mkv`, `.mov`, `.avi`, `.webm`)
    - YouTube URL download + processing
  - Automatic language detection (LID) using Whisper / MMS, restricted to supported `TARGET_LANGS`
  - Model controls:
    - ASR models: `whisper`, `faster_whisper`, `ai4bharat` (Indic Conformer with fallback to Whisper)
    - Whisper size and decoding strategy configuration
  - UI shows:
    - Detected language, model used, progress bars, errors, and a **post‑ASR “Continue to MT” flow**
  - Optional chat history persistence per user via `/chats` endpoints

- **Machine Translation (Text → Text)**
  - `MT` page takes ASR transcription (plus source audio if present)
  - Language selectors for **source** and **target**; model selector for:
    - `indictrans` (default), `google`, `nllb`
  - `mt.py`:
    - Sentence and chunk splitting for long text
    - Language‑code normalization between ISO, IndicTrans tags, and FLORES codes
    - Primary model + **automatic fallback** (e.g. IndicTrans → Google → IndicTrans)
  - Result can be copied and is passed directly to the TTS page.

- **Text‑to‑Speech (Text → Speech)**
  - `TTS` page receives translated text (and optionally original text) from MT
  - Model choices:
    - **Auto** (chooses Indic or XTTS or gTTS based on language)
    - **XTTS (Coqui)** for multilingual, voice‑cloned speech
    - **Indic Parler‑TTS** for high‑quality Indic voices
    - **gTTS** as universal fallback
  - `tts_handler.py`:
    - Safe token‑based text splitting for XTTS
    - Voice cloning support via reference WAV (e.g. `samples/female_clip.wav`)
    - Output written to `tts_output/` and served by `/tts/audio/{filename}`
  - Frontend provides audio playback and download.

- **Authentication & User Management**
  - Signup (`/signup`) → email OTP (`/complete-signup`) → welcome email
  - SMS OTP APIs: `/send-otp`, `/verify-otp`, `/resend-otp`
  - Captcha‑protected login: `/login-with-captcha`, `/verify-captcha`
  - Optional Firebase phone‑auth verification: `/verify-firebase-phone`
  - Authenticated `/me` and `/chats` endpoints for profile and history
  - Frontend components:
    - `AuthDialog`, `SignupForm`, `LoginForm`, `OtpVerificationModal`, `PhoneMfaModal`, `CaptchaField`

### 3. Running the Web App (Dev)

- **Backend**
  - `cd backend`
  - Create venv + install: `python -m venv .venv` → activate → `pip install -r requirements.txt` or `python setup_asr.py`
  - Install external tools: **FFmpeg**, spaCy model (`python -m spacy download en_core_web_lg`), CUDA (optional)
  - Start: `uvicorn main:app --reload --host 0.0.0.0 --port 8000`

- **Frontend**
  - `cd frontend`
  - Install: `npm install` (or `bun install`)
  - Run dev: `npm run dev` (default `http://localhost:5173`)
  - Ensure `API_BASE_URL` in `src/config/api.ts` points to the backend (e.g. `http://localhost:8000`)

For deeper API details, environment variables, and deployment (AWS / DigitalOcean / Vercel / Firebase), refer to the markdown guides in this directory and the backend folder.