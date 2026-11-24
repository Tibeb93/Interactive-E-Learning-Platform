# Interactive E-Learning Platform — Backend Architecture & Phase-by-Phase Implementation Guide

> **Project Standard:** Production / Client-Ready Web Application  
> **Target Stack:** Node.js, Express.js (ES Modules), MySQL2 (Connection Pool), JWT, Bcrypt, Cheerio, PostCSS, PDFKit, QRCode  
> **Architecture Pattern:** Controller - Service - Repository / Model Pattern with Strict Role-Based Middleware

---

## 1. System Architecture & High-Level Backend Flow

```
   [ Client Requests ] (HTTP / JSON)
           │
           ▼
   [ Express App & Global Middlewares ] (CORS, Helmet, Rate-Limit, JSON Parser)
           │
           ▼
   [ Routes Layer ] (/api/auth, /api/courses, /api/exercises, etc.)
           │
           ▼
   [ Auth & RBAC Middlewares ] (verifyToken, requireRole('student'|'instructor'|'admin'))
           │
           ▼
   [ Controllers Layer ] (Request Validation, Response Formatting 200/400/401/403/404)
           │
           ▼
   [ Services / Business Logic Layer ]
      ├── exerciseChecker.js   (DOM & CSS rule validation engine)
      ├── gradingService.js    (Server-side quiz scoring & integrity)
      ├── certificateService.js (Eligibility calculation + PDF/QR generator)
      └── progressService.js   (Dynamically calculated completion metrics)
           │
           ▼
   [ Database Layer (MySQL2 Pool) ] (19 Normalized relational tables with Foreign Keys)
```

---

## 2. Database Schema Specification (19 Normalized Tables)

All tables must use InnoDB, UTF8MB4, integer auto-increment primary keys, and foreign key constraints.

### 2.1 Table Directory
1. **`roles`**: `id`, `name` ('student', 'instructor', 'admin')
2. **`users`**: `id`, `role_id` (FK), `name`, `email` (UNIQUE INDEX), `password_hash`, `avatar_url`, `created_at`, `updated_at`
3. **`password_resets`**: `id`, `user_id` (FK), `token_hash`, `expires_at`, `used_at`, `created_at`
4. **`categories`**: `id`, `name`, `slug` (UNIQUE INDEX), `description`, `created_at`
5. **`courses`**: `id`, `instructor_id` (FK), `category_id` (FK), `title`, `slug` (UNIQUE INDEX), `description`, `thumbnail_url`, `objectives` (JSON/TEXT), `level` ('beginner', 'intermediate', 'advanced'), `duration_minutes`, `status` ('draft', 'published'), `created_at`, `updated_at`
6. **`course_modules`**: `id`, `course_id` (FK CASCADE), `title`, `position`, `created_at`
7. **`lessons`**: `id`, `module_id` (FK CASCADE), `title`, `content` (Markdown/HTML), `starter_code` (JSON `{html, css, js}`), `position`, `created_at`
8. **`materials`**: `id`, `lesson_id` (FK CASCADE), `file_url`, `title`, `created_at`
9. **`enrollments`**: `id`, `user_id` (FK CASCADE), `course_id` (FK CASCADE), `enrolled_at`, `last_lesson_id` (FK SET NULL), `UNIQUE KEY (user_id, course_id)`
10. **`lesson_progress`**: `id`, `user_id` (FK CASCADE), `lesson_id` (FK CASCADE), `completed_at`, `UNIQUE KEY (user_id, lesson_id)`
11. **`exercises`**: `id`, `lesson_id` (FK CASCADE), `instructions`, `starter_code` (JSON), `check_type` ('html_structure', 'css_rule', 'text_match', 'js_test'), `check_rules` (JSON), `created_at`
12. **`exercise_submissions`**: `id`, `exercise_id` (FK CASCADE), `user_id` (FK CASCADE), `submitted_code` (JSON/TEXT), `passed` (BOOLEAN), `feedback` (JSON), `submitted_at`
13. **`quizzes`**: `id`, `course_id` (FK CASCADE), `title`, `pass_mark` (INT, e.g. 80%), `time_limit_minutes`, `created_at`
14. **`questions`**: `id`, `quiz_id` (FK CASCADE), `type` ('multiple_choice', 'true_false', 'short_answer'), `prompt`, `correct_text` (NULL for MCQs), `points` (DEFAULT 1)
15. **`question_options`**: `id`, `question_id` (FK CASCADE), `label`, `is_correct` (BOOLEAN)
16. **`quiz_attempts`**: `id`, `quiz_id` (FK CASCADE), `user_id` (FK CASCADE), `score` (DECIMAL(5,2)), `passed` (BOOLEAN), `started_at`, `finished_at`
17. **`quiz_answers`**: `id`, `attempt_id` (FK CASCADE), `question_id` (FK CASCADE), `answer` (TEXT), `is_correct` (BOOLEAN)
18. **`certificates`**: `id`, `enrollment_id` (FK CASCADE, UNIQUE), `certificate_code` (UNIQUE INDEX, UUID/hash), `issued_at`
19. **`course_reviews`**: `id`, `course_id` (FK CASCADE), `user_id` (FK CASCADE), `rating` (INT 1-5), `comment` (TEXT), `created_at`

---

## 3. Core Business Logic Engines

### 3.1 Exercise Auto-Checker Engine (`exerciseChecker.js`)
The exercise evaluation engine runs on the backend to enforce verification without trusting client payloads:

1. **`html_structure`**:
   - Uses `cheerio` to parse learner HTML into a DOM tree.
   - Evaluates CSS selectors, node counts, attribute presence (`attr()`), child hierarchy, and inner text.
   - *Example rule:* `[{ selector: "h1", count: 1, message: "Add exactly one <h1> heading." }]`
2. **`css_rule`**:
   - Uses `postcss` or regex AST to parse learner CSS.
   - Matches selector declarations to check if specific properties (e.g. `color`, `display`, `margin`) match expected values.
3. **`text_match`**:
   - Cleans whitespace, trims input, and performs exact string match or regex evaluation (`new RegExp(pattern, flags)`).
4. **`js_test`**:
   - Evaluates assertion results provided by client iframe execution or Node `vm2`/isolated environment.
   - Validates execution pass flags with structural AST checking (e.g. verifying function definitions).

### 3.2 Dynamic Course Progress Calculation Logic
Progress is **never stored as a hardcoded static counter** to avoid synchronization errors. It is computed dynamically:
$$\text{Progress \%} = \left( \frac{\text{Completed Lessons}}{\text{Total Course Lessons}} \times 0.6 \right) + \left( \frac{\text{Passed Mandatory Exercises}}{\text{Total Course Exercises}} \times 0.2 \right) + \left( \frac{\text{Quiz Passed ? } 1 : 0}{1} \times 0.2 \right) \times 100$$
*(Or standard linear ratio of completed items).*

### 3.3 Certificate Verification & Generation Logic
When a student triggers `POST /api/courses/:id/certificate`:
1. Check: Are **all lessons** for this course marked complete in `lesson_progress`?
2. Check: Are **all exercises** passed in `exercise_submissions`?
3. Check: Is there at least one record in `quiz_attempts` where `passed = 1`?
4. If **any check fails**, reject with HTTP 400 and an itemized checklist of unfinished requirements.
5. If **all checks pass**:
   - Generate a cryptographically unique certificate code: e.g. `CERT-2026-` + random nanoid/UUID.
   - Insert row into `certificates` table.
   - Dynamically generate branded PDF using `PDFKit` with embedded QR Code (`qrcode` library) pointing to `https://<DOMAIN>/verify/<CODE>`.

---

## 4. Backend Phase-by-Phase Roadmap & Action Plan

### Phase 1: Planning & Database Initialization
- [ ] Initialize Node.js project: `npm init -y` with `"type": "module"`.
- [ ] Install core dependencies: `express`, `mysql2`, `dotenv`, `cors`, `helmet`, `bcryptjs`, `jsonwebtoken`, `express-rate-limit`.
- [ ] Create `database/schema.sql` containing all 19 tables, indexes, and constraints.
- [ ] Create `database/seed.sql` with default roles (`student`, `instructor`, `admin`), categories (HTML, CSS, JS), initial instructor user, sample courses, modules, lessons, starter exercises, and quizzes.
- [ ] Create `src/config/db.js` using `mysql2/promise` connection pool.

### Phase 2: Authentication, Security & RBAC Middleware
- [ ] **`src/middleware/authMiddleware.js`**:
  - `authenticateToken`: Extracts JWT from `Authorization: Bearer <token>` or HTTP-only cookies.
  - Verifies signature using `JWT_SECRET`, queries user status, attaches `req.user` (`{ id, role, email }`).
- [ ] **`src/middleware/roleMiddleware.js`**:
  - `requireRole(...roles)`: Rejects with HTTP 403 if `req.user.role` is not permitted.
- [ ] **Auth Routes (`/api/auth`)**:
  - `POST /register`: Password validation, bcrypt hash ($cost \ge 10$), insert user with default role `student`.
  - `POST /login`: Match email, verify bcrypt hash, sign JWT token with expiration (e.g. 24h).
  - `POST /forgot-password` & `POST /reset-password`: Tokenized password reset system.
- [ ] **User Profile Routes (`/api/users`)**:
  - `GET /api/users/me`: Return authenticated user info (without password hash).
  - `PUT /api/users/me`: Update profile details (name, avatar).

### Phase 3: Course Catalog & Instructor Management API
- [ ] **Public Course Routes (`/api/courses`)**:
  - `GET /`: Search, category filter (`category_id`), level filter (`beginner|intermediate|advanced`), pagination (`page`, `limit`).
  - `GET /:id`: Full course overview, instructor info, list of modules and lesson titles.
- [ ] **Instructor Course CRUD (`/api/courses`)**:
  - `POST /`: Create new draft course (Instructor/Admin only).
  - `PUT /:id`: Update title, description, objectives, level, status (`draft` / `published`).
  - `DELETE /:id`: Delete course (cascade deletes modules, lessons).
- [ ] **Module & Lesson Management**:
  - `POST /courses/:id/modules`: Create module with `position` ordering.
  - `POST /modules/:id/lessons`: Create lesson with Markdown content and starter code JSON.
  - `PUT /lessons/:id`: Update lesson content, ordering, and materials.

### Phase 4: Enrollment, Learning & Exercise Checker Engine
- [ ] **Enrollment API**:
  - `POST /api/courses/:id/enroll`: Creates enrollment record for the logged-in student.
  - `GET /api/me/enrollments`: Lists user's enrolled courses with dynamic progress %.
- [ ] **Lesson Content Delivery**:
  - `GET /api/lessons/:id`: Verified against enrollment. Returns lesson text, starter code, and next/previous lesson IDs.
  - `POST /api/lessons/:id/complete`: Inserts or updates `lesson_progress`. Updates `enrollments.last_lesson_id`.
- [ ] **Exercise Verification API (`/api/exercises`)**:
  - `GET /api/exercises/:id`: Get instructions and starter code template.
  - `POST /api/exercises/:id/submit`:
    - Checks enrollment.
    - Loads `check_rules` JSON from database.
    - Dispatches to `exerciseChecker.js` (Cheerio for HTML, PostCSS for CSS, Regex/Text match).
    - Writes attempt to `exercise_submissions` with status `passed: true/false` and itemized feedback.
    - Returns JSON: `{ passed: boolean, feedback: string[] }`.

### Phase 5: Assessments, Dynamic Progress & Verifiable Certificates
- [ ] **Quiz Engine (`/api/quizzes`)**:
  - `GET /api/quizzes/:id`: Returns quiz metadata, time limit, and questions **WITHOUT** correct answer indicators.
  - `POST /api/quizzes/:id/attempts`: Begins an attempt session, marks `started_at`.
  - `POST /api/attempts/:id/submit`:
    - Compares submitted option IDs / text against backend database truth.
    - Computes percentage score against `quizzes.pass_mark`.
    - Writes to `quiz_attempts` and `quiz_answers`.
    - Returns score, pass/fail result, and detailed review.
- [ ] **Student Dashboard Aggregator (`GET /api/me/dashboard`)**:
  - Returns enrolled courses, % progress calculated on the fly, last accessed lesson shortcut, quiz scores, and issued certificates.
- [ ] **Certificate Generation API (`/api/certificates`)**:
  - `POST /api/courses/:id/certificate`: Validates full completion of lessons, exercises, and quiz. Generates UUID certificate code.
  - `GET /api/certificates/:code/pdf`: Generates and streams branded vector PDF certificate on the fly using `PDFKit` and embedded `QRCode`.
  - `GET /api/verify/:code`: Public verification endpoint returning student name, course title, and completion date.

### Phase 6: Admin Governance, Security Hardening & Testing
- [ ] **Admin Governance API (`/api/admin`)**:
  - `GET /api/admin/stats`: Aggregates total students, instructors, courses, active enrollments, and certificates issued.
  - `GET /api/admin/users`: User management with role update endpoint (`PATCH /api/admin/users/:id/role`).
  - `GET /api/admin/certificates`: Audit log of all issued credentials.
- [ ] **Security Hardening**:
  - CORS whitelist restricted strictly to `CLIENT_URL`.
  - Express Rate Limiting applied to `/api/auth/*` routes.
  - Helmet for security HTTP response headers.
  - Parameterized MySQL queries everywhere to prevent SQL injection.
  - Global error handler returning standard JSON: `{ error: "User-friendly message" }` (no raw stack traces in production).
- [ ] **Unit & Integration Tests**:
  - Jest / Supertest test suites for Auth, Role authorization, Exercise rule evaluator, and Quiz scoring logic.
