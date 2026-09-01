# JARVIS

עוזר AI מקומי ל-macOS עם HUD בסגנון Stark Industries. מקבל פקודות קול/טקסט, מריץ skills על המחשב, ומציג מצב מערכת בזמן אמת.

הפרויקט בנוי כשכבות: **מוח** (Gemini עם failover ל-Groq), **skills** (מודולים עם `execute()`), **HUD** (Flask templates + Socket.IO), ומנועי רקע (זיכרון סמנטי, sentinel, תצפית ויזואלית).

> **שים לב:** בסנאפשוט הזה חסר קובץ השרת הראשי (למשל `jarvis.py` / `app.py`) וגם `style.css`, `safety_manager.py`, `coder_agent.py` ו-`filesystem_watcher.py`. ה-HUD וה-skills מצפים לשרת Flask-SocketIO שמעביר פקודות ל-executor. בלי הקבצים האלה אי אפשר להרים את הממשק המלא.

## מה הוא יודע לעשות

- לדבר (ElevenLabs → edge-tts → pyttsx3) ולהאזין (Google STT / Vosk)
- לפתוח אפליקציות, להריץ AppleScript/shell, לשלוט בעכבר/מקלדת/ווליום
- חיפוש רשת, מזג אוויר, חדשות, תרגום, סיכום, מחשבון והמרות
- יומן, תזכורות, הערות, clipboard, briefing יומי
- מצלמה, מסך, OCR, זיהוי פנים, visual overlay / 3D assembly על ה-HUD
- Spotify, WhatsApp/Twilio, אימייל, iMessage/Telegram/Discord
- HomeKit / Home Assistant
- מאגר ידע (RAG מקומי), כספת מוצפנת, סינתזת skill חדש

## מבנה

```
jarvis/
├── skills/                 # יכולות — כל קובץ חושף execute(params)
├── utils/
│   ├── neural_switchboard.py   # Gemini → Groq, רוטציית מפתחות
│   ├── skill_registry.py       # קטלוג skills ל-prompt ול-validation
│   ├── gemini_rotator.py       # רוטציה על 429
│   ├── audio_manager.py        # דיבור, echo-cancel, Music
│   ├── voice_verifier.py       # voice-print (MFCC)
│   └── system_discovery.py     # אפליקציות ומערכת ל-prompt
├── templates/              # HUD (/) + Holographic Lab (/lab)
├── static/                 # script.js, lab.js, lab.css
├── semantic_memory.py      # embeddings (Gemini) + semantic_index.json
├── visual_observer.py      # צילום חלון פעיל + הקשר ויזואלי
├── system_sentinel.py      # CPU/RAM/סוללה, תזמון, קבצים
├── scheduler_system.py     # משימות ברקע
├── topology_engine.py      # גרף קבצים ל-Lab (`GET /api/topology`)
├── skill_synthesis_engine.py
└── setup_*.py              # הורדת מודלי Vosk
```

כל skill ב-`skills/` מייצא `execute(params: dict)`. ה-registry ב-`utils/skill_registry.py` מתאר פרמטרים ורמת סיכון (`risk` 0/1). Skills מסוכנים יותר (shell, קבצים, vault, סינתזה) עוברים דרך `safety_manager` כשהוא זמין.

## HUD

- **`/`** — Heads-Up Display: מדדי מערכת, Neural Bridge (auto/Gemini/Groq), תמלול, קלט פקודות, מודעות ויזואלית, פאנל ויזואלי (SVG / overlay / Mermaid / 3D).
- **`/lab`** — Lab הולוגרפי: טופולוגיית הפרויקט ב-Three.js + לוגים של researcher / Coder / Sentinel.

הלקוח מדבר ב-Socket.IO (`ui_command`, `new_message`, `voice_level`, `visual_panel`, אימות פנים/קול/PIN).

## מוח (Neural Switchboard)

`utils/neural_switchboard.py`:

1. Gemini (`GEMINI_API_KEYS`, כמה מפתחות מופרדים בפסיק/רווח; רוטציה ב-429)
2. Groq (`GROQ_API_KEY`) אם Gemini נכשל

מודל ברירת מחדל בטסטים: `gemini-2.5-flash-lite`. ב-HUD אפשר לבחור ספק ידנית.

## התקנה

macOS. Python 3.12+ מומלץ.

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install google-genai groq python-dotenv pillow opencv-python \
  edge-tts pygame numpy psutil pyautogui SpeechRecognition \
  requests watchdog pytesseract flask flask-socketio
```

מודל דיבור עברי כלול חלקית (`vosk-model-small-he-0.15.zip`). מודלי אנגלית:

```bash
python setup_vosk.py              # small EN-US
python setup_small_vosk.py        # light Indian English
python setup_indian_vosk.py       # high-accuracy Indian English (~1GB)
```

צור `.env` בשורש (לא ב-git):

```env
# חובה למוח
GEMINI_API_KEYS=key1,key2
GEMINI_MODEL=gemini-2.5-flash-lite
GROQ_API_KEY=

# דיבור (אופציונלי — בלי זה: edge-tts)
ELEVENLABS_API_KEY=
ELEVENLABS_VOICE_ID=pNInz6obpgDQGcFmaJgB

# Skills אופציונליים
JARVIS_CITY=Tel Aviv
OPENWEATHER_API_KEY=
NEWS_API_KEY=
EMAIL_USER=
EMAIL_PASS=
TWILIO_SID=
TWILIO_AUTH_TOKEN=
TWILIO_WHATSAPP_FROM=
TELEGRAM_BOT_TOKEN=
DISCORD_WEBHOOK_URL=
HA_URL=http://homeassistant.local:8123
HA_TOKEN=
JARVIS_VAULT_MASTER=
JARVIS_VOICE_THRESHOLD=0.78
```

בדיקת חיבור בסיסית (Gemini + מצלמה + TTS):

```bash
python test_jarvis.py
python test_all_skills.py    # ייבוא + execute בכל skill
python test_switchboard.py   # Gemini / Groq
```

## Skills

| קטגוריה | Skills |
|---|---|
| ליבה | `speak`, `listen`, `hello`, `smart_action`, `open_app`, `shell_execution` |
| מערכת | `system_monitor`, `volume`, `timer`, `scheduler`, `file_watcher`, `run_script` |
| קבצים | `list_files`, `file_management`, `file_search`, `doc_reader` |
| רשת | `web_search`, `weather`, `news`, `browser`, `translate`, `summarize` |
| פרודוקטיביות | `calendar`, `reminders`, `notes`, `clipboard`, `calculator`, `convert`, `shortcuts` |
| תקשורת | `email_sender`, `send_whatsapp_message`, `messenger` |
| חושים | `vision`, `camera`, `screen_capture`, `screen_analysis`, `ocr`, `face_rec`, `mood` |
| מדיה / בית | `spotify`, `smart_home`, `notifications`, `location` |
| AI / זיכרון | `learn`, `recall_memory`, `knowledge_base`, `code_assistant`, `generate_visual`, `create_skill` |
| אישי | `daily_briefing`, `health`, `vault` |
| קלט | `mouse_control`, `keyboard_control` |

Skill חדש: קובץ `skills/<name>.py` עם `execute(params)`, ואז רשומה ב-`SKILL_REGISTRY`. סינתזה אוטומטית דרך `skill_synthesis_engine` דורשת `CoderAgent` שלא נמצא בסנאפשוט הזה.

## מנועי רקע

| מודול | תפקיד |
|---|---|
| `SemanticMemory` | שומר/מחפש חוויות לפי embedding של Gemini |
| `VisualObserver` | צילום החלון הפעיל, הקשר ל-HUD |
| `SystemSentinel` | עומס מתמשך, סוללה נמוכה, briefing ב-09:00 |
| `TaskScheduler` | משימות חד-פעמיות/חוזרות |
| `TopologyEngine` | גרף קבצים + imports ל-Lab |
| `AudioManager` | מונע echo כשהוא מדבר |
| `VoiceVerifier` | voice-print של הבעלים (`voice_prints/owner.npy`) |

## פלטפורמה והרשאות

מכוון ל-**macOS** (AppleScript, Spotlight/`mdfind`, HomeKit, Quartz לצילום חלון). חלק מה-skills יעבדו גם במקומות אחרים, אבל האוטומציה המלאה היא Mac.

בהרצה ראשונה macOS יבקש גישה למיקרופון, מצלמה, נגישות (עכבר/מקלדת), ומסך.

## רישיון

שימוש אישי / לימודי. לא לכלול מפתחות API ב-git.
