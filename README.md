# DriveEase — Peer-to-Peer Vehicle Rental Marketplace

DriveEase is a complete full-stack rental marketplace: vehicle owners list cars, renters book them, and the
platform handles the whole rental — owner approval, a three-party digital agreement, pickup and return
inspections, simulated GPS tracking, simulated payments, reviews and an admin back office.

* **Backend:** Java 21, Spring Boot 3.3.5, Spring Security + JWT, Spring Data JPA / Hibernate, MySQL 8, Maven
* **Frontend:** React 18 (JavaScript), Vite, React Router, Axios, Tailwind CSS, Lucide icons, Recharts,
  React Leaflet + OpenStreetMap
* **Roles:** Renter, Vehicle Owner, Manager/Admin

> **Demo disclaimer:** payments and GPS tracking are simulated. The rental agreement is a demo document, not a
> legally certified e-signature. Nothing here handles real money or real vehicle telemetry.

---

## 1. What you need (Windows)

Install these three things, in this order. Restart your computer afterwards so PATH changes apply.

| Software | Version | Download | Check in a new Command Prompt |
| --- | --- | --- | --- |
| JDK | **21** (LTS) | https://adoptium.net/temurin/releases/?version=21 (choose *Windows x64 .msi*) | `java -version` → must say 21 |
| Node.js | **18 or 20 LTS** | https://nodejs.org/en/download (Windows Installer .msi) | `node -v` and `npm -v` |
| MySQL | **8.x** | https://dev.mysql.com/downloads/installer/ → *MySQL Installer for Windows* | see step 2 |

During the MySQL installer choose **Server only** (or *Custom* → MySQL Server + MySQL Workbench). Set a
**root password** and write it down — you will need it in step 3. Leave the port at **3306** and let the
installer register MySQL as a Windows service (it then starts automatically).

You do **not** need to install Maven or Git: the backend ships with the Maven Wrapper (`mvnw.cmd`), which
downloads Maven on first run.

---

## 2. Create the database

The backend can create the `driveease` database itself, but the MySQL user needs permission, so the simplest
path is to create it once.

**Option A — one click:** double-click `scripts\setup-database.bat`, type your MySQL username (`root`) and
password when asked.

**Option B — MySQL Workbench:** open Workbench → connect to your local instance → open a query tab → run:

```sql
CREATE DATABASE IF NOT EXISTS driveease
  CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

**Option C — Command Prompt:**

```bat
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS driveease CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
```

You do **not** have to create tables. On first start the backend creates all 21 tables and inserts the demo
data automatically (`SEED_ENABLED=true`). If you prefer to load the exact database from the original demo
(including bookings, inspections, reviews and audit logs), import `database\driveease-demo.sql` in Workbench
instead — see section 7.

---

## 3. Configure your database password

1. In the `backend` folder, copy `.env.example` and rename the copy to **`.env`**
   (in Explorer: *View → Show → File name extensions*, then rename `.env.example` → `.env`).
2. Open `backend\.env` in Notepad and set your own values:

```properties
DB_URL=jdbc:mysql://localhost:3306/driveease?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC&createDatabaseIfNotExist=true
DB_USERNAME=root
DB_PASSWORD=your_actual_mysql_root_password
JWT_SECRET=any-long-random-string-you-like-at-least-32-chars
SERVER_PORT=8080
SEED_ENABLED=true
CORS_ORIGINS=http://localhost:5173,http://127.0.0.1:5173
```

`backend\.env` is read automatically at startup. It is git-ignored, so your password stays out of version
control. The shipped `application.properties` contains only placeholders (`YOUR_MYSQL_PASSWORD`) and safe
defaults.

The frontend needs no configuration for local use: it calls `/api`, which Vite proxies to
`http://localhost:8080` (see `frontend\vite.config.js`). `frontend\.env.example` explains the optional
`VITE_API_URL` override.

---

## 4. Start the app

### Easiest: double-click `start-driveease.bat`

It opens two console windows (backend + frontend), waits ~25 seconds and opens
**http://localhost:5173** in your browser. Keep both windows open while you use the app; closing them stops
the servers.

### Or start the two halves manually

**Backend** (Command Prompt):

```bat
cd DriveEase\backend
mvnw.cmd spring-boot:run
```

Wait for `Started DriveEaseApplication` and `Tomcat started on port 8080`. First run downloads Maven and all
Java dependencies, so it can take a few minutes.

**Frontend** (a second Command Prompt):

```bat
cd DriveEase\frontend
npm install
npm run dev
```

`npm install` is needed only the first time (or after `package.json` changes). Vite prints
`Local: http://localhost:5173/`.

### URLs

| URL | Purpose |
| --- | --- |
| http://localhost:5173 | The application |
| http://localhost:8080/api/vehicles | Sample API response (JSON) |
| http://localhost:8080/swagger-ui.html | Swagger UI — every endpoint, try them live |

---

## 5. Demo accounts

All accounts are created by the seeder (passwords are BCrypt-hashed in the database).

| Role | Email | Password | Notes |
| --- | --- | --- | --- |
| Manager / Admin | `admin@driveease.com` | `Admin@123` | Users, approvals, disputes, analytics, audit logs |
| Vehicle owner | `owner@driveease.com` | `Owner@123` | Priya Sharma, Bengaluru, verified |
| Vehicle owner | `meera@driveease.com` | `Owner@123` | Mumbai |
| Vehicle owner | `rohan@driveease.com` | `Owner@123` | Pending verification (useful for the admin demo) |
| Renter | `renter@driveease.com` | `Renter@123` | Karthik Rao |
| Renter | `ananya@driveease.com` | `Renter@123` | Mumbai |
| Renter | `vikram@driveease.com` | `Renter@123` | Delhi |

You can also register a new renter or owner from the site. New owner accounts start unverified and their
listings wait for manager approval — that is the intended flow, not a bug.

**Fast demo path:** sign in as the renter → open a car → pick dates → *Request booking* → sign in as the owner
in a second window → accept → both parties (plus admin) accept the agreement → pay (simulated) → owner records
the pickup inspection → renter opens *Track my rental* → owner records the return inspection → renter leaves a
review. `docs\DEMO.md` has a full 10-minute script.

---

## 6. Project structure

```
DriveEase/
├── backend/
│   ├── pom.xml                     Maven build (Spring Boot 3.3.5, Java 21)
│   ├── mvnw / mvnw.cmd             Maven Wrapper — no Maven install needed
│   ├── .env.example                DB + JWT configuration template
│   ├── uploads/seed/               Seeded vehicle photos (served at /uploads/**)
│   └── src/main/
│       ├── java/com/driveease/
│       │   ├── config/             Security, JWT filter, CORS, OpenAPI, seed runner
│       │   ├── controller/         REST controllers
│       │   ├── dto/                Request/response records (entities never exposed)
│       │   ├── entity/             JPA entities + enums
│       │   ├── exception/          Global error handling
│       │   ├── repository/         Spring Data repositories
│       │   ├── security/           JWT service, principal, ownership checks
│       │   ├── service/            Business rules, lifecycle, GPS simulator
│       │   └── seed/DataSeeder.java
│       └── resources/application.properties
├── frontend/
│   ├── package.json                React, Vite, Tailwind, Axios, Recharts, Leaflet, Lucide
│   ├── .env.example                Optional API URL override
│   ├── index.html, vite.config.js, tailwind.config.js, postcss.config.js
│   ├── public/favicon.svg
│   └── src/
│       ├── components/             UI kit, VehicleCard, MapView, AgreementView, InspectionForm …
│       ├── context/                Auth + toast providers
│       ├── hooks/                  useApi, useDebounce
│       ├── lib/                    api.js (axios), format.js
│       └── pages/
│           ├── public/             Home, Browse, VehicleDetail, Login, Register, ListYourCar, Safety
│           ├── renter/             Overview, Bookings, Favorites, Reviews
│           ├── owner/              Overview, Vehicles, VehicleForm, Requests, Rentals, Earnings …
│           ├── admin/              16 manager screens
│           └── shared/             BookingDetail, Agreements, Inspections, Tracking, Notifications …
├── database/
│   ├── driveease-demo.sql          Full dump: schema + all demo rows (Workbench import)
│   └── seed-data.sql               Data-only INSERTs (schema created by the backend)
├── docs/DEMO.md                    10-minute demo script and Q&A prep
├── scripts/
│   ├── setup-database.bat          Creates the driveease database (Windows)
│   ├── start.sh                    macOS/Linux starter
│   └── stop.sh
├── schema.sql                      Table structure only (21 tables)
├── start-driveease.bat             Starts backend + frontend and opens the browser
├── .env.example                    Configuration template (copy to backend\.env)
├── .gitignore
└── README.md
```

---

## 7. Database: schema and demo data

* **Automatic (recommended):** leave `SEED_ENABLED=true`. On first start Hibernate creates the 21 tables and
  `DataSeeder` inserts 7 users, 13 vehicles, 9 bookings across every status, agreements, inspections, reviews,
  safety reports, notifications, transactions, audit logs and one active rental with a live simulated GPS
  session. The seeder is idempotent — restarting does not duplicate data.
* **Exact copy of the tested demo database:** in MySQL Workbench open
  `database/driveease-demo.sql` and execute it (it contains `CREATE TABLE` + `INSERT` statements, and drops
  existing tables first). Then set `SEED_ENABLED=false` in `backend\.env` if you want to keep the data
  untouched, or leave it `true` — it will not duplicate existing records.
* **Structure only:** `schema.sql` at the root, or `database/seed-data.sql` for data on top of tables created
  by the backend.

Schema management uses Hibernate `ddl-auto=update` for demo convenience; production projects should use
Flyway/Liquibase migrations.

---

## 8. Troubleshooting (Windows)

| Symptom | Cause / fix |
| --- | --- |
| `java -version` shows 17 or "not recognized" | Install JDK 21, then open a **new** Command Prompt. Check `JAVA_HOME` points to the JDK 21 folder. |
| Backend: `Access denied for user 'root'@'localhost'` | Wrong `DB_PASSWORD` in `backend\.env`, or MySQL not running. Test with `mysql -u root -p`. |
| Backend: `Communications link failure` | MySQL service stopped. Start it: *Services* → **MySQL80** → Start, or `net start MySQL80` in an admin prompt. |
| Backend: `Port 8080 was already in use` | Another app uses 8080. Set `SERVER_PORT=8081` in `backend\.env` **and** change the proxy target in `frontend\vite.config.js` to `http://localhost:8081`. |
| Backend: `Unknown database 'driveease'` | Run section 2 (create the database) or add `createDatabaseIfNotExist=true` (already in the default `DB_URL`). |
| Frontend: `'vite' is not recognized` | Run `npm install` inside the `frontend` folder first. |
| Frontend: `Port 5173 is in use` | Vite will offer another port — accept it, then update `CORS_ORIGINS` in `backend\.env` to that origin. |
| Frontend loads but every list is empty / "Network Error" | Backend not running, or running on a different port. Check http://localhost:8080/api/vehicles. |
| Login fails for all demo accounts | The seeder did not run: check `SEED_ENABLED=true`, or import `database\driveease-demo.sql`. |
| Vehicle photos show a broken image | The backend must be started **from the `backend` folder** (uploads are served from `./uploads`). Use `mvnw.cmd spring-boot:run` inside `backend`. |
| `npm install` fails behind a proxy | `npm config set proxy http://user:pass@host:port` and `npm config set https-proxy ...`. |
| CORS error in the browser console | `CORS_ORIGINS` must contain the exact frontend URL, e.g. `http://localhost:5173`. |

macOS / Linux: the same steps apply; use `./mvnw spring-boot:run`, `npm install`, `npm run dev`, and
`scripts/start.sh` if you prefer.

---

## 9. How it works (for the viva)

**Booking lifecycle**

```
PENDING ─owner accepts→ ACCEPTED ─all 3 parties accept→ AGREEMENT_SIGNED ─renter pays→ CONFIRMED
   │                        │                                                          │
   │ owner rejects          │ renter cancels                                   pickup inspection
   ▼                        ▼                                                          ▼
REJECTED                 CANCELLED                                                IN_PROGRESS (GPS live)
                                                                                       │ return inspection
                                                                                       ▼
                                                                                   COMPLETED → review
```

* Every transition is validated in `BookingService`; illegal jumps return HTTP 409.
* **Double-booking prevention:** an overlap query rejects any request whose dates intersect an existing active
  booking or an owner-blocked window (409 Conflict). The same check runs again when the owner accepts.
* **Pricing snapshot:** daily rate × days + service fee + optional damage protection is stored on the booking,
  so later price edits never rewrite history.
* **Security:** JWT (HS256) + `@PreAuthorize` role guards, plus per-object ownership checks in services. All
  input goes through DTOs, so `role`, `status` and `ownerId` cannot be mass-assigned. Passwords are BCrypt
  hashed and never returned by any endpoint.
* **Notifications, transactions, audit logs:** all backend-driven and persisted, not mocked in React.

---

## 10. Known limitations (honest list)

1. Payments are simulated — no gateway, no card data, "transactions" and refunds are database records only.
2. GPS is simulated by a scheduler; `POST /api/tracking/location-update` is the seam for a real device feed.
3. The rental agreement records three-party acceptance with user id + timestamp but is **not** a legally
   certified e-signature.
4. Overlap prevention is a transactional check, adequate for a demo but not race-proof under heavy concurrency;
   production would add a database range constraint or row lock.
5. Schema uses `ddl-auto=update`, not versioned migrations.
6. Uploaded photos are stored on local disk (`backend/uploads`), suitable for a single machine only.
7. Notifications are in-app only (no email/SMS/push).
8. Search uses SQL `LIKE`, not a dedicated search engine.
9. No automated test suite is bundled; the project was verified with scripted API calls and Playwright browser
   tests during development (52 routes + 10 interaction checks passing).

---

## 11. Credits

Vehicle photography is bundled locally in `backend/uploads/seed/` so the demo works offline. All users,
vehicles, bookings, reviews and messages are fictional. Map tiles © OpenStreetMap contributors.
