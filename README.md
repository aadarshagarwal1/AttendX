# AttendX

GPS-based attendance management for educational institutions. Teachers start short, location-bound sessions; students mark attendance from their phones only when they are physically near the classroom. Sessions are time-limited, verified with Haversine distance checks, and persisted in PostgreSQL.

## Features

- **OTP login** — Email one-time password (no passwords stored on device)
- **Role-based app** — Separate flows for students, teachers, and admins
- **Geo-fenced check-in** — Students must be within ~20 m of the teacher’s session coordinates
- **Timed sessions** — Attendance windows close automatically (~3 minutes after start)
- **Push notifications** — Firebase Cloud Messaging alerts students when a session opens
- **Institution data model** — Classes, semesters, departments, subjects, and teacher–class assignments

## How it works

```text
Teacher                          Server                           Student
   |                                |                                 |
   |-- Start session (GPS + class) ->|                                 |
   |                                |-- Create Attendance record       |
   |                                |-- Notify via FCM --------------->|
   |                                |                                 |
   |                                |<-- Mark attendance (GPS + JWT) -|
   |                                |-- Verify distance to teacher     |
   |                                |-- Update student_records         |
   |-- End session ---------------->|-- Persist final records to DB  |
```

1. A teacher selects branch, semester, and subject, then starts a session with their current GPS coordinates.
2. The API creates an `Attendance` row, initializes all enrolled students as `absent`, and sends push notifications.
3. Students open the app, share their location, and call the mark-attendance endpoint.
4. The server compares student and teacher coordinates; if within range, the student is marked `present`.
5. When the teacher ends the session, records are written to the database.

## Project structure

```text
attendx/
├── client/                 # React Native mobile app (Expo)
│   └── app/
│       ├── auth/           # Login & OTP verification
│       ├── student/        # Student home, profile, logs
│       ├── teacher/        # Teacher home, profile, logs
│       ├── student-others/ # Attendance calendar, notifications
│       └── teacher-others/ # Session overlay & timer UI
└── server/                 # Node.js API
    ├── controllers/        # Route handlers
    ├── routes/             # Express routers
    ├── middlewares/        # Auth & error handling
    ├── prisma/             # Schema & migrations
    └── utils/              # Mail, FCM, location, seeding
```

## Tech stack

| Layer | Technologies |
|-------|----------------|
| Mobile | React Native, Expo 53, Expo Router, Expo Location |
| API | Node.js, Express 5, JWT |
| Database | PostgreSQL, Prisma ORM |
| Notifications | Firebase Admin SDK (FCM) |
| Email | Nodemailer (OTP delivery) |

## Prerequisites

- Node.js 18+
- PostgreSQL
- [Expo CLI](https://docs.expo.dev/) and Android Studio or a physical device for the mobile app
- Firebase project with FCM enabled (for push notifications)
- SMTP credentials for OTP email (e.g. Mailtrap for development)

## Getting started

### 1. Clone and install

```bash
git clone <repository-url>
cd attendx

cd server && npm install
cd ../client && npm install
```

### 2. Database setup

Create a PostgreSQL database, then configure the server environment.

Create `server/.env`:

```env
DATABASE_URL="postgresql://USER:PASSWORD@localhost:5432/attendx"
JWT_SECRET="your-secret-key"
NODE_ENV="development"
```

Run migrations and generate the Prisma client:

```bash
cd server
npx prisma migrate deploy
npx prisma generate
```

Optional — seed sample users, classes, and subjects:

```bash
node utils/DataSeeder.js
```

### 3. Firebase (push notifications)

Place your Firebase Admin SDK service account JSON in `server/` (see `server/utils/firebase.js` for the expected filename). The server uses it to send session alerts to student devices.

### 4. Email (OTP login)

Configure SMTP in `server/utils/mailer.js` (or refactor to use environment variables). OTPs expire after 5 minutes.

### 5. Start the API

```bash
cd server
npm run dev
```

The server listens on `http://0.0.0.0:3000`.

### 6. Start the mobile app

Point the client at your API. Set `extra.apiUrl` in `client/app.config.js` to your machine’s LAN IP or a tunnel URL (e.g. ngrok) so a physical device can reach the backend:

```js
extra: {
  apiUrl: "http://192.168.x.x:3000",
}
```

Then:

```bash
cd client
npx expo start
```

Use a [development build](https://docs.expo.dev/develop/development-builds/introduction/) for Firebase Messaging and location permissions on device.

## API overview

Base path: `/api/v1`

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/user/loginOtpSend` | Send OTP to email |
| POST | `/user/loginOtpVerify` | Verify OTP, returns JWT + role |
| POST | `/user/verifyToken` | Validate JWT |
| POST | `/user/storeFcm` | Save device FCM token |
| POST | `/home/getUser` | Teacher profile and classes taught |
| POST | `/teacher/getSubjects` | Subjects for a teacher |
| POST | `/attendance/startSession` | Start attendance (teacher GPS) |
| POST | `/attendance/getMarked` | Mark student present (student GPS) |
| POST | `/attendance/endSession` | End session for teacher |
| POST | `/attendance/storeRecords` | Persist session to database |
| POST | `/admin/addTeacher` | Admin: onboard a teacher |
| GET | `/admin/test` | Health check for admin routes |

Authenticated routes expect a JWT in the request body as `token` (issued after OTP verification).

## Data model

Core entities (see `server/prisma/schema.prisma`):

- **User** — Profile, email, role (`student` | `teacher` | `admin`), department, FCM token
- **Student** — Registration number, class, year
- **Teacher** — Staff ID, assigned classes/subjects via **TeacherClass**
- **Class** — Branch, semester (I–VIII), enrolled students
- **Subject** — Course code and name
- **Attendance** — Session with teacher coordinates, timestamps, and `student_records` JSON

## Mobile app routes

| Route | Role | Purpose |
|-------|------|---------|
| `/auth/login` | All | Email entry |
| `/auth/otp` | All | OTP verification |
| `/teacher/home` | Teacher | Start attendance sessions |
| `/student/home` | Student | Subject overview |
| `/student-others/attendance` | Student | Attendance calendar |
| `/student-others/notification` | Student | Notifications panel |

## Development notes

- **Active sessions** use a temporary JSON file (`server/records.temp.json`) during the marking window before records are committed to PostgreSQL.
- **Location validation** uses the Haversine formula in `server/utils/checkLocation.js` (default radius: 20 meters).
- Some **client screens still use mock data** (student home stats, calendar, notifications). The attendance API flow on the server is the most complete integration.
- `protectRoute` middleware in `server/middlewares/auth.middleware.js` is not fully implemented; most routes validate JWT inline in controllers.
- Do not commit secrets: `.env`, Firebase service account JSON, or SMTP credentials.

## License

ISC (see `server/package.json`).
