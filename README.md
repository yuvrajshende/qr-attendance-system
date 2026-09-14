# Attend — QR Code Based Student Attendance Management System

A full-stack MERN application that replaces manual roll-calls with a rotating QR
code: a teacher projects a code that refreshes every few seconds, students scan
it with their phone camera, and attendance is recorded instantly with duplicate
protection, expiry, and live reporting.

Built as a final-year engineering project, but structured like a production
codebase — role-based auth, a documented REST API, and a clean React front end.

---

## Table of contents

1. [Features](#features)
2. [Tech stack](#tech-stack)
3. [How the QR flow works](#how-the-qr-flow-works)
4. [Project structure](#project-structure)
5. [Getting started](#getting-started)
6. [Demo accounts](#demo-accounts)
7. [API reference](#api-reference)
8. [Data models](#data-models)
9. [Security notes](#security-notes)
10. [Possible extensions](#possible-extensions)
11. [License](#license)

---

## Features

| # | Feature | Notes |
|---|---------|-------|
| 1 | Admin login | Role-based JWT auth |
| 2 | Teacher login | Same auth endpoint, role returned in token |
| 3 | Student registration | Self-signup with roll number + optional class code |
| 4 | Dynamic QR generation | One QR per class session, encoded as a signed session token |
| 5 | Scan-to-mark attendance | Camera-based scanning via `html5-qrcode` |
| 6 | QR auto-expiry | Token rotates every `QR_EXPIRY_SECONDS` (default 30s) |
| 7 | Duplicate prevention | Unique DB index on `(session, student)` |
| 8 | Date & time logging | Every record stores `markedAt` and `sessionDate` |
| 9 | Attendance percentage | Calculated per class and across all classes |
| 10 | Teacher dashboard | Live present/absent counter while a session is open |
| 11 | Student dashboard | Per-class percentage with a low-attendance warning |
| 12 | Attendance history | Filterable by class and date range |
| 13 | Search & filter | By student name/roll number and by date |
| 14 | CSV export | Full attendance sheet, one column per session date |

## Tech stack

- **Frontend:** React 18, React Router 6, Axios, `html5-qrcode`
- **Backend:** Node.js, Express.js
- **Database:** MongoDB with Mongoose
- **Auth:** JSON Web Tokens (JWT) + bcrypt password hashing
- **QR generation:** `qrcode` (server-side PNG data URLs)
- **QR scanning:** `html5-qrcode` (client-side camera access)
- **Reports:** `json2csv`

## How the QR flow works

```
Teacher opens a session for a class
        │
        ▼
Server creates a Session doc with a random token + 30s expiry,
encodes { sessionId, token } as a QR PNG, returns it to the teacher
        │
        ▼
Teacher's screen shows the QR and silently calls
POST /sessions/:id/refresh every 30s → new token, new QR image
        │
        ▼
Student scans the code with their phone camera
        │
        ▼
App decodes { sessionId, token } and POSTs it to
POST /attendance/mark
        │
        ▼
Server checks: session open? token matches the *current* one?
not expired? student enrolled? not already marked?
        │
        ├── all pass → Attendance record created, student sees "Marked ✓"
        └── any fail → specific error returned (expired / duplicate / etc.)
```

Because the token embedded in the QR rotates continuously, a screenshot shared
in a group chat stops working within seconds — you cannot mark attendance for
someone else without physically pointing a camera at the live screen.

## Project structure

```
qr-attendance-system/
├── backend/
│   ├── config/db.js               MongoDB connection
│   ├── models/                    User, Class, Session, Attendance
│   ├── middleware/                auth (JWT), role (RBAC), errorHandler
│   ├── controllers/                auth, class, session, attendance, report
│   ├── routes/                    one router per resource
│   ├── utils/                     token signing, QR generation, DB seed script
│   ├── server.js
│   └── package.json
│
├── frontend/
│   ├── public/index.html
│   └── src/
│       ├── api/                   axios instance + grouped endpoint calls
│       ├── context/AuthContext.jsx
│       ├── components/            DashboardLayout, QRDisplay, QRScanner, ...
│       ├── pages/
│       │   ├── admin/             Overview, Users, Classes
│       │   ├── teacher/           Overview, Classes, GenerateQR
│       │   └── student/           Overview, ScanQR, AttendanceHistory
│       ├── styles/index.css       design tokens + component styles
│       └── App.jsx                routing
│
└── README.md
```

## Getting started

### Prerequisites

- Node.js 18+
- A MongoDB instance — either local (`mongod`) or a free [MongoDB Atlas](https://www.mongodb.com/atlas) cluster
- A phone or webcam for testing the QR scanner (camera access requires HTTPS or `localhost`)

### 1. Backend

```bash
cd backend
cp .env.example .env
# edit .env: set MONGO_URI and a strong JWT_SECRET
npm install
npm run seed     # optional: creates demo admin/teacher/student + a class
npm run dev      # starts on http://localhost:5000
```

### 2. Frontend

```bash
cd frontend
cp .env.example .env
# edit .env if your backend isn't on localhost:5000
npm install
npm start         # starts on http://localhost:3000
```

Open `http://localhost:3000` in two browser windows (or one desktop + one
phone on the same network with the API URL updated to your machine's LAN IP)
to simulate a teacher projecting a QR code and a student scanning it.

### Running in production

- Build the frontend with `npm run build` and serve the static files from any
  static host or from Express itself.
- Set `NODE_ENV=production`, a long random `JWT_SECRET`, and restrict
  `CLIENT_URL` to your real frontend origin for CORS.
- Camera access in browsers requires a secure context — deploy behind HTTPS.

## Demo accounts

After running `npm run seed` in `backend/`:

| Role    | Email                  | Password      |
|---------|------------------------|---------------|
| Admin   | admin@campus.edu       | Admin@123     |
| Teacher | teacher@campus.edu     | Teacher@123   |
| Student | student1@campus.edu    | Student@123   |

The seed script also creates a class `CS301 — Data Structures & Algorithms`
with three enrolled students.

## API reference

All routes are prefixed with `/api`. Protected routes require
`Authorization: Bearer <token>`.

### Auth
| Method | Route | Access | Description |
|---|---|---|---|
| POST | `/auth/register` | Public | Student self-registration |
| POST | `/auth/login` | Public | Login for any role |
| GET | `/auth/me` | Any | Current user profile |
| POST | `/auth/admin/create-user` | Admin | Create teacher/admin/student accounts |

### Classes
| Method | Route | Access | Description |
|---|---|---|---|
| POST | `/classes` | Admin/Teacher | Create a class |
| GET | `/classes` | Any | List classes scoped to the caller's role |
| GET | `/classes/:id` | Any (enrolled/owning) | Class details |
| POST | `/classes/:id/students` | Admin/Teacher | Enroll students by ID or roll number |
| DELETE | `/classes/:id/students/:studentId` | Admin/Teacher | Remove a student |
| DELETE | `/classes/:id` | Admin/Teacher | Deactivate a class |

### Sessions (QR lifecycle)
| Method | Route | Access | Description |
|---|---|---|---|
| POST | `/sessions/open` | Admin/Teacher | Open/resume today's session, get first QR |
| POST | `/sessions/:id/refresh` | Admin/Teacher | Rotate the QR token |
| GET | `/sessions/:id` | Admin/Teacher | Live present/absent stats |
| POST | `/sessions/:id/close` | Admin/Teacher | Close session, invalidate QR |

### Attendance
| Method | Route | Access | Description |
|---|---|---|---|
| POST | `/attendance/mark` | Student | Submit a scanned `{sessionId, token}` |
| GET | `/attendance/history` | Any | Filter by `classId`, `from`, `to` |
| GET | `/attendance/percentage` | Any | Per-class attendance percentage |
| GET | `/attendance/class/:classId` | Admin/Teacher | Full roster summary, `search` & `date` filters |

### Reports
| Method | Route | Access | Description |
|---|---|---|---|
| GET | `/reports/class/:classId/export` | Admin/Teacher | Download CSV, one column per session date |

## Data models

- **User** — `name, email, password (hashed), role, rollNumber?, department?, enrolledClasses[]`
- **Class** — `name, subjectCode, teacher, students[], schedule`
- **Session** — `class, teacher, currentToken, tokenExpiresAt, status, sessionDate`
- **Attendance** — `session, class, student, markedAt, sessionDate` (unique on `session + student`)

## Security notes

- Passwords are hashed with bcrypt; plaintext is never stored or logged.
- JWTs are short-lived (`JWT_EXPIRES_IN`, default 8h) and required on every protected route.
- Role checks happen server-side (`middleware/role.js`), not just hidden in the UI.
- QR tokens are single-use per rotation window and re-validated against the
  database on every scan, so replaying an old QR image always fails.
- A student can only mark attendance for classes they are actually enrolled in.
- Basic rate limiting is applied to the whole API to slow down brute-force attempts.

This is a student project reference implementation — before real deployment,
add HTTPS, refresh tokens, audit logging, and a privacy review for camera/location data.

## Possible extensions

- Geofencing: only accept scans from within a set radius of the classroom.
- Push/email notification when a student's attendance drops below a threshold.
- Bulk student import via CSV/Excel.
- Biometric or face-verification as a second factor for high-stakes exams.
- Native mobile app (React Native) reusing the same REST API.

## License

MIT — free to use and adapt for coursework, portfolios, or further development.
