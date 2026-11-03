# IoT-Based Digital Voting and Public Bill Archive System

A fingerprint-authenticated digital voting terminal built around an **ESP32**, an **AS608 (DY50) fingerprint sensor** and a **3.5" ILI9488 touch TFT**, connected over Wi-Fi to a central server, an administrative website and a public bill archive.

This is an **academic prototype**. A real parliamentary deployment would need a much higher level of security, independent testing, accessibility review and formal operational authority.

---

## 📌 Project Overview

The complete system has four parts:

```
Administrative Website  (Main Host + Assistant Host, desktop browser)
        │  Wi-Fi / local network / HTTPS API
        ▼
Central Server + Database  (users, fingerprint reference IDs, bills, sessions, votes, results, audit records)
        │  Wi-Fi / API
        ▼
ESP32 Voting Terminal  (ESP32 + AS608 + 3.5" TFT touch + buttons + LEDs + buzzer)
        │  published finalized records
        ▼
Public Website and Bill Archive  (final results, proposal PDFs, bill details, historical terms)
```

Planned features:

- Members authenticate with a **fingerprint** and vote on the ESP32 terminal
- Bills have a Bangla/English title, a detailed voting question, voting options, a session duration and a full proposal PDF
- **Main Host** approves or rejects sessions; **Assistant Host** manages members, fingerprint enrollment, terms and bill preparation
- The **server** is the authority for deadline, eligibility, vote acceptance and final totals
- Members may revise their vote only while the session is active (if enabled)
- Final results and PDFs are published to a searchable public archive

---

## 🚦 Current Status (end of Day 3)

The ESP32 terminal hardware is working on its own. Nothing is connected to Wi-Fi or a server yet.

| Item | Status |
|---|---|
| ESP32 upload and Serial Monitor | ✅ Working |
| 3.5" ILI9488 TFT display | ✅ Working |
| XPT2046 touch panel (calibrated, saved to flash) | ✅ Working |
| AS608 fingerprint sensor (enroll / test / delete / erase all) | ✅ Working through the touch menu |
| Buttons, LEDs, buzzer | ⬜ Not wired yet |
| Wi-Fi + API connection | ⬜ Not started |
| Backend, database, admin site, public site | ⬜ Not started |

---

## 🧰 Hardware Used

| Component | Purpose |
|---|---|
| ESP32 DevKit V1 (30-pin, Micro-USB) | Main microcontroller |
| DY50 fingerprint module (AS608 chip) | Fingerprint enrollment and matching |
| 3.5" SPI TFT, 480x320, with resistive touch (XPT2046) | Display and touch menu |
| Push buttons, LEDs, buzzer | Extra feedback (planned) |
| Breadboard and jumper wires | Prototyping, shared 3.3 V and GND rails |
| Digital multimeter | Voltage and continuity checks |
| Micro-USB **data** cable | Programming and power |

---

## 🔌 Pin Configuration

### ESP32 pin map (labels as printed on the board)

| ESP32 pin | GPIO | Connects to | Function |
|---|---:|---|---|
| VIN | 5 V | TFT VCC | Display power (jumper J1 open) |
| 3V3 | 3.3 V | TFT LED, DY50 3V3, DY50 T-3V3 | Backlight, sensor power, touch-ring power |
| GND | 0 V | TFT GND, DY50 GND | Common ground |
| D5 | 5 | TFT CS | Display chip select |
| D18 | 18 | TFT SCK + T_CLK | Shared SPI clock |
| D19 | 19 | TFT T_DO **only** | SPI MISO (touch data out) |
| D23 | 23 | TFT SDI (MOSI) + T_DIN | Shared SPI MOSI |
| D26 | 26 | TFT RESET | Display reset |
| D27 | 27 | TFT DC/RS | Data / command |
| D25 | 25 | TFT T_CS | Touch chip select |
| D33 | 33 | TFT T_IRQ | Touch interrupt (optional, unused in code) |
| RX2 | 16 | DY50 **TX** | Fingerprint UART receive |
| TX2 | 17 | DY50 **RX** | Fingerprint UART transmit |
| D34 | 34 | DY50 T-OUT | Finger-present signal (optional, input-only pin) |

Pins for the buttons, LEDs and buzzer are **not assigned yet**. They will be chosen from the free GPIOs when that hardware is added (avoid TX0/RX0, and be careful with strapping pins such as GPIO 0, 2, 12 and 15).

### TFT header (left to right)

`T_IRQ` · `T_DO` · `T_DIN` · `T_CS` · `T_CLK` · `SDO(MISO)` · `LED` · `SCK` · `SDI(MOSI)` · `DC/RS` · `RESET` · `CS` · `GND` · `VCC`

> ⚠️ **Leave the TFT `SDO(MISO)` pin unconnected.** The ILI9488 does not release the MISO line when its chip select is high, so if it shares D19 with the touch `T_DO` it corrupts the touch data. The display only needs to receive data, so nothing is lost.

### DY50 / AS608 (6-pin connector)

| DY50 pin | ESP32 |
|---|---|
| GND | GND |
| RX | TX2 (GPIO 17) |
| TX | RX2 (GPIO 16) |
| 3V3 | 3V3 |
| T-OUT | D34 (optional) |
| T-3V3 | 3V3 |

TX/RX are named from the module's point of view, so they **cross over**.

### Power rules

- TFT VCC takes 5 V from **VIN**, with solder jumper **J1 open**. (Alternative: bridge J1 and feed VCC from 3V3. Never mix the two.)
- The fingerprint sensor stays on **3.3 V only**.
- The ESP32 has only one 3V3 pin, so use a breadboard 3.3 V rail for the three 3.3 V wires.
- All grounds must be joined.
- Never connect 5 V to any ESP32 GPIO pin.

---

## ⚙️ Software Configuration

### Arduino IDE (hardware tests)

1. Install the Arduino IDE and the **ESP32 by Espressif Systems** board package.
2. Select **Tools → Board → ESP32 Dev Module** and the correct port.
3. Install the libraries **TFT_eSPI** (Bodmer) and **Adafruit Fingerprint Sensor Library**. `Preferences` is built in.
4. Replace the contents of `Arduino/libraries/TFT_eSPI/User_Setup.h` with the settings below. In `User_Setup_Select.h` only `#include <User_Setup.h>` should be active.
5. **Quit and reopen the Arduino IDE** after editing the setup file.
6. Open the Serial Monitor at **115200 baud**.

```cpp
#define ILI9488_DRIVER       // if the screen stays white/garbled, try ST7796_DRIVER

#define TFT_MISO 19
#define TFT_MOSI 23
#define TFT_SCLK 18
#define TFT_CS    5
#define TFT_DC   27
#define TFT_RST  26
#define TOUCH_CS 25          // enables getTouch / setTouch / calibrateTouch

#define LOAD_GLCD
#define LOAD_FONT2
#define LOAD_FONT4
#define SMOOTH_FONT

#define SPI_FREQUENCY       20000000
#define SPI_READ_FREQUENCY  16000000
#define SPI_TOUCH_FREQUENCY  2500000
```

### PlatformIO (final firmware)

The final firmware will be built with the Arduino framework in PlatformIO (VS Code), inside `esp32-firmware/`. The prototype sketches below are used to prove each part of the hardware first.

### Prototype sketches

| Sketch | Purpose |
|---|---|
| `touch_test.ino` | Touch calibration and drawing test (also prints `calData`) |
| `fingerprint_touch_app.ino` | Touch menu: **ENROLL / DELETE / TEST / ERASE ALL**, touch calibration saved to flash |

---

## 🧠 How the Current Terminal Works

`fingerprint_touch_app.ino` shows a touch menu and a header with the number of stored fingerprints.

| Button | Action |
|---|---|
| **ENROLL** | Takes the first free ID, asks for the finger twice, stores the template and shows the ID |
| **DELETE** | Scan a finger; the matching ID is deleted |
| **TEST** | Scan a finger; shows `MATCH` with ID and score, or `ACCESS DENIED` |
| **ERASE ALL** | Two taps within 3 seconds wipe every stored fingerprint |

- Touching the screen during any "place finger" step cancels it; every wait times out after 10 seconds
- Touch calibration is done once and saved to flash; touch the screen at boot (2-second window) to recalibrate
- A button press must be read twice at nearly the same spot, which filters touch noise

In the final system the terminal will send the matched **template ID** and the chosen vote to the server, and the **server** decides whether the vote is accepted.

---

## 🔧 Troubleshooting (Problems We Actually Faced)

| Problem | Cause | Fix |
|---|---|---|
| Board powers on but no COM port / cannot upload | Charging-only or poor USB cable | Use a known-good Micro-USB **data** cable and test it before buying |
| `'class TFT_eSPI' has no member named 'getTouch'` | Default `User_Setup.h` still active: `TOUCH_CS` commented out, wrong driver, ESP8266 pins | Use the setup above, check `User_Setup_Select.h`, restart the IDE |
| Display and sensor show no sign of power after a successful upload | 3V3 wires (TFT LED, DY50 3V3, DY50 T-3V3) not connected | Join them on a breadboard 3.3 V rail; verify with a multimeter |
| Touch does nothing or gives random values | TFT `SDO(MISO)` and `T_DO` both on D19 | Leave TFT `SDO(MISO)` unconnected |
| Touch drawing dotted or jittery | Redrawing and Serial printing on every sample; resistive panel noise | Large buttons, double-read filter, press firmly, saved calibration |
| `Sensor NOT found` | TX/RX not crossed, no 3V3/GND, wrong baud | DY50 TX to RX2, DY50 RX to TX2; check power; try 9600 or 115200 baud |
| White or garbled screen | Wrong display driver | Try `ST7796_DRIVER` instead of `ILI9488_DRIVER` |

### Safety checklist before powering on

- Unplug USB before changing any wire
- Jumper J1 open when VCC is on 5 V
- No 5 V on any GPIO or on the sensor
- Continuity test (power off): no beep between 3V3 and GND, none between VIN and GND
- Check voltages with the multimeter: VIN about 5 V, 3V3 about 3.3 V
- Discharge static before touching bare boards

---

# 🗺️ A-to-Z Development Plan

Build and test each component separately before combining everything. Get a working local prototype first, then refine.

## Technology stack

| Component | Technology |
|---|---|
| Admin and public websites | React + Vite |
| Backend / API | Python + FastAPI |
| Database (prototype) | SQLite |
| PDF storage | Local server folder initially |
| ESP32 firmware | Arduino framework, PlatformIO in VS Code |
| Fingerprint sensor | AS608 with a compatible UART library |
| TFT display | ILI9488 with TFT_eSPI |
| Communication | Wi-Fi + HTTP/HTTPS REST API |
| Version control | Git + GitHub |

Keep the first version local: run the server and database on your computer and connect the ESP32 to the same Wi-Fi network. Consider hosting and secure remote access only after the local prototype works.

## Roadmap

### Phase 0 - Freeze the requirements
Deliverable: a fixed feature list and workflow.
- Main Host approves or rejects proposed sessions
- Assistant Host manages members, fingerprint enrollment, parliamentary terms and bill preparation
- A bill has a Bangla/English title, detailed question, voting options, session duration and a full proposal PDF
- Members authenticate by fingerprint and vote on the ESP32 terminal
- Vote revision is allowed only while the session is active, if enabled
- The server finalizes results and publishes the approved record

### Phase 1 - Prepare the development environment
Deliverable: tools installed and a working GitHub repository.
- Verify Python, VS Code, Git, Node.js and PlatformIO
- Create the project folder, initialize Git and a Python virtual environment
- Create a minimal FastAPI server and a minimal React site, and confirm they can talk

### Phase 2 - Test the hardware separately
Deliverable: each physical component works independently.
1. ESP32 LED blink ✅
2. Buzzer tones ⬜
3. Button presses ⬜
4. Wi-Fi connection and reconnection ⬜
5. AS608 enrollment and matching ✅
6. ILI9488 TFT text, colours and touch buttons ✅
7. Bangla and English text rendering, including font memory and layout limits ⬜

### Phase 3 - Build the database and backend
Deliverable: an API and database that store and return test records.
- Tables for users, terms, fingerprint references, bills, sessions, votes, audit events and publication records
- API to create, edit, approve, start, close and fetch sessions
- Store PDFs on the server with metadata and path in the database
- Server-side deadline and vote validation; a small mock parliament as test data

### Phase 4 - Build the administrative website
Deliverable: Main Host and Assistant Host can manage the system from a browser.
- Login and role-based access
- Term and member management
- Fingerprint enrollment that links a sensor template ID to the right member
- Bill form with Bangla/English inputs and PDF upload
- Session preview and submission; Main Host approval/rejection and monitoring
- Show the active session, remaining time, participation count and device status

### Phase 5 - Connect the ESP32 to the server
Deliverable: the device displays a session received over Wi-Fi.
- Safe Wi-Fi credential handling and a unique device ID
- Fetch the current session; show bill title, question, options and status
- Loading, disconnected, reconnecting and server-error screens
- The terminal must **not** use its own clock to decide whether voting is allowed

### Phase 6 - Implement fingerprint voting
Deliverable: an eligible mock member can cast a verified vote.
- Scan, match with the AS608, send the template ID and vote to the backend
- The backend checks membership, term eligibility, session state, deadline and duplicate/revision rules
- Clear success or failure on the TFT, plus buzzer and LED feedback

### Phase 7 - Build the public website and archive
Deliverable: visitors can find finalized bills and results.
- Home page and finalized-bills list with search and filters (term, title, date)
- Bill detail page: question, options, final totals and proposal PDF
- Historical term pages
- Drafts, rejected proposals and active sessions are never shown as final records

### Phase 8 - Security, reliability and integration
Deliverable: correct behaviour in normal use and during failures.
- Protect admin accounts and restrict actions by role
- Validate PDF type and size
- Use HTTPS beyond a trusted local network
- Make vote submission safe to retry without double counting
- Keep a revision/audit trail
- Test Wi-Fi loss, device restarts, repeated scans, expired sessions and simultaneous requests
- Back up the database and PDFs together

### Phase 9 - Final demonstration and documentation
Deliverable: a repeatable end-to-end demo.
- Run a mock term with sample members, prepare a bill with a PDF, get Main Host approval
- Conduct fingerprint voting, verify the final count and publish
- Confirm the PDF and result appear in the public archive
- Prepare the circuit diagram, database diagram, API documentation, screenshots, test results and report

## Project folder structure

```text
parliament-voting-system/
│
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── database.py
│   │   ├── models.py
│   │   ├── schemas.py
│   │   ├── security.py
│   │   ├── api/
│   │   │   ├── auth.py
│   │   │   ├── members.py
│   │   │   ├── terms.py
│   │   │   ├── bills.py
│   │   │   ├── sessions.py
│   │   │   ├── votes.py
│   │   │   └── public.py
│   │   └── services/
│   │       ├── voting.py
│   │       ├── finalization.py
│   │       └── documents.py
│   ├── uploads/
│   ├── tests/
│   └── requirements.txt
│
├── admin-website/
│   └── src/ (pages, components, services, App.jsx)
│
├── public-website/
│   └── src/ (pages, components, services)
│
├── esp32-firmware/
│   ├── platformio.ini
│   ├── src/main.cpp
│   └── prototypes/            # touch_test.ino, fingerprint_touch_app.ino
│
├── docs/
│   ├── requirements.md
│   ├── architecture.md
│   ├── database-design.md
│   ├── circuit-diagram.md
│   └── testing.md
│
├── .gitignore
└── README.md
```

## Database design

| Table | Purpose |
|---|---|
| users | Hosts, members, roles, account status |
| parliamentary_terms | Five-year terms and their status |
| term_memberships | Which members belong to which term |
| fingerprint_references | Sensor template IDs linked to members |
| bills | Title, detailed question, languages and metadata |
| documents | Proposal PDF path, size and checksum |
| sessions | Approval, start/end time, options and status |
| votes | Latest accepted vote per member and session |
| vote_audit | Accepted revisions and vote events |
| audit_events | Administrative changes, approvals and other key actions |

Design rules:

- Store only the AS608 **template reference**, not raw fingerprint images or biometric data, in the website database. Test the sensor's template capacity.
- Put a unique constraint on each member/session vote so duplicate network requests cannot create duplicate votes.
- Record revisions in an audit table if changing a vote is allowed.
- Keep PDFs outside the database and store their metadata in it.
- The server is the authority for deadline, eligibility, vote acceptance and final totals.

## API endpoints (proposed)

| Endpoint | Purpose |
|---|---|
| `POST /api/auth/login` | Administrative login |
| `GET /api/terms` | List parliamentary terms |
| `POST /api/members` | Register a member |
| `POST /api/bills` | Create bill metadata |
| `POST /api/bills/{id}/document` | Upload proposal PDF |
| `POST /api/sessions` | Submit a proposed session |
| `POST /api/sessions/{id}/approve` | Main Host approves |
| `POST /api/sessions/{id}/reject` | Main Host rejects |
| `POST /api/sessions/{id}/start` | Start an approved session |
| `GET /api/device/session` | ESP32 fetches the active session |
| `POST /api/device/vote` | Submit a member's vote |
| `GET /api/sessions/{id}/results` | Retrieve authorized results |
| `GET /api/public/bills` | List finalized public bills |
| `GET /api/public/bills/{id}` | Read a finalized bill |
| `GET /api/public/bills/{id}/document` | Access its published proposal PDF |

These are proposed names, not existing code. Admin endpoints must enforce role permissions, and device endpoints must identify the terminal and check the voting state.

## Voting session state machine

```
Draft  →  Pending Approval  →  Approved  →  Active  →  Finalized
(Assistant   (waiting for       (authorized   (valid votes   (votes locked,
 Host         Main Host)         to start)     accepted       result published)
 prepares)                                     until deadline)
```

- Rejection moves a proposal to a non-active rejected state; resubmission or reopening needs explicit authorization.
- When a session expires, the backend stops accepting votes and finalizes the result with a reliable server-side process.
- A device showing an old "active" screen must never override the server's session state.
- By default, publish only final totals; do **not** publish individual members' vote choices.

## Suggested schedule

| Week | Target |
|---|---|
| 1 | Requirements, tools, GitHub repository |
| 2 | ESP32, buttons, buzzer, TFT and AS608 individual tests |
| 3 | Database schema and basic backend APIs |
| 4 | Login, member management, parliamentary terms |
| 5 | Bill preparation, Bangla/English text, PDF uploads |
| 6 | Main Host approval and session controls |
| 7 | ESP32 Wi-Fi connection and active-session display |
| 8 | Fingerprint authentication and vote submission |
| 9 | Finalization, result calculation, public archive |
| 10 | Security, integration tests, bug fixes, demonstration |

Allow extra time for the TFT driver, Bangla font rendering, fingerprint enrollment or network reliability issues.

## Final testing checklist

**Hardware**
- [x] ESP32 starts reliably
- [x] AS608 enrollment and matching work
- [x] TFT displays readable English text
- [ ] Bangla font rendering tested
- [ ] Buttons, LEDs and buzzer work

**Administration**
- [ ] Assistant Host can register members
- [ ] Assistant Host can upload a proposal PDF
- [ ] Main Host can approve or reject a session
- [ ] Unauthorized roles cannot perform restricted actions

**Voting**
- [ ] Only eligible members can vote
- [ ] Invalid fingerprints are rejected
- [ ] Duplicate requests do not double-count votes
- [ ] Permitted vote revisions update the latest vote
- [ ] Votes are rejected after the server deadline
- [ ] Network failure does not falsely confirm a vote

**Publication**
- [ ] Final result matches accepted votes
- [ ] Finalized bill and PDF appear in the public archive
- [ ] Draft and rejected proposals stay private
- [ ] Past parliamentary terms remain searchable
- [ ] Database and PDF backups can be restored

---

## ▶️ Next Step

Prepare the development environment (Phase 1), then add the remaining Phase 2 hardware tests (buzzer, buttons, Wi-Fi) before building the backend.

```bash
python3 --version
git --version
node --version
npm --version

git init
mkdir -p backend admin-website public-website esp32-firmware docs
```

Create a `.gitignore` so passwords, environment files, real-data database files and private uploads are never committed.

---

## 👥 Team

Developed by a team of four members for a Spring Semester 2026 embedded systems project.

Md Arif Mahmud Oyion,
Md Rakibul Islam,
Niloy Bormon,
Sheum Ahmed

American International University-Bangladesh
