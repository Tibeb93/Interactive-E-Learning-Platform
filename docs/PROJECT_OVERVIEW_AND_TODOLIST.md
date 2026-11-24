# Interactive E-Learning Platform — Master Execution Plan & To-Do List

> **Project Title:** Interactive E-Learning Platform (W3Schools-Inspired, Original Design)  
> **Course:** Final Summer Bootcamp Project  
> **Standard:** Client-Ready, Production-Grade Full-Stack Application  
> **Evaluation Focus:** Browser Code Execution Sandbox & Automated Rule-Based Exercise Evaluation

---

## 1. Executive Summary & Success Test

The platform is a comprehensive learning and assessment system where students register, enroll in programming tracks (HTML, CSS, JavaScript), study structured interactive tutorials, write and execute code directly in their browser without server security risk, receive automated instant feedback on coding exercises, take assessments, and earn cryptographic, verifiable PDF certificates with scannable QR codes.

### The Success Test (The Ultimate Evaluator Demo Flow):

$$\text{Register} \longrightarrow \text{Enroll} \longrightarrow \text{Study Lesson} \longrightarrow \text{Run Code in Editor} \longrightarrow \text{Pass Exercise} \longrightarrow \text{Pass Quiz} \longrightarrow \text{Download Verifiable Certificate}$$

---

## 2. System Architecture at a Glance

```
                                  +------------------------------+
                                  |   Client Browser (React)     |
                                  +--------------+---------------+
                                                 |
                       +-------------------------+-------------------------+
                       |                                                   |
              REST API Calls (HTTPS / JSON)                       Learner Code Execution
                       |                                                   |
                       v                                                   v
         +----------------------------+                          +--------------------+
         | Node.js + Express REST API |                          |  Sandboxed <iframe>|
         +-------------+--------------+                          |   (No Same-Origin) |
                       |                                         +--------------------+
        +--------------+--------------+
        |                             |
        v                             v
+---------------+             +----------------+
| MySQL Database|             | Cloud / Static |
|  (19 Tables)  |             |  Asset Storage |
+---------------+             +----------------+
```

---

## 3. Deep Dive: Key Technical Concepts & Core Logic

### 3.1 Why run learner code in an iframe instead of on the server?

- **Security:** Executing arbitrary, untrusted user code (especially JavaScript) on a Node.js server exposes the backend to remote code execution (RCE), Denial of Service (infinite loops `while(true){}`), file system tampering, and credential theft.
- **Client Sandboxing:** The browser provides native isolation using `<iframe sandbox="allow-scripts">` **without** `allow-same-origin`.
  - **`allow-scripts`**: Allows the learner's JavaScript to run and manipulate DOM elements inside that frame.
  - **Omission of `allow-same-origin`**: Treats the iframe as a completely distinct, unique origin. The code inside **cannot** read parent `localStorage`, session cookies, authentication tokens, or access the host application's DOM.
- **Resource Protection:** If a student writes an infinite loop, only the isolated iframe tab freezes; the parent React application remains responsive, and the student can click "Reset".

### 3.2 How Automatic Exercise Checking Works

Instead of relying on human graders or brittle string matching:

1. Each exercise carries machine-readable `check_rules` stored as structured JSON in the database.
2. When the student clicks **Submit Exercise**:
   - The code payload is dispatched to `POST /api/exercises/:id/submit`.
   - The backend checks student enrollment and loads the rule specification.
   - For **`html_structure`**: The backend parses the HTML with Cheerio and evaluates selector queries (e.g. requires `<h1>`, verifies `<ul>` has at least 3 `<li>` children).
   - For **`css_rule`**: The backend parses CSS declarations and validates property-value pairs (e.g., `color: red`, `display: flex`).
   - For **`text_match`**: The backend normalizes whitespace and matches regex patterns.
   - For **`js_test`**: Assertions execute in the client sandbox and verify state transformations.
3. The server generates a detailed report: `{ passed: true/false, feedback: ["Rule 1 Passed", "Missing <h1> tag"] }`.
4. If passed for the first time, a record is created in `exercise_submissions`, unlocking progress.

### 3.3 Dynamic Course Progress vs. Static Counters

- **The Problem:** Storing static fields like `courses.completion_percentage` in the database causes desynchronization bugs whenever lessons or exercises are added or updated.
- **The Solution:** The completion percentage is **calculated dynamically** from the database rows upon query:
  - Total lessons vs. rows in `lesson_progress`.
  - Required exercises vs. passing rows in `exercise_submissions`.
  - Final quiz attempt status in `quiz_attempts`.

### 3.4 Cryptographically Verifiable PDF Certificates

1. **Condition Gate:** A certificate can **only** be minted if `100% of lessons are completed`, `all exercises are passed`, and the `quiz score >= pass_mark`.
2. **Tamper-Proof Code:** When issued, a random, unique alphanumeric code is generated (e.g., `CERT-A8F9-2026`).
3. **On-Demand Vector PDF:** PDFKit dynamically compiles the student's name, course name, issue date, instructor signature, and an embedded QR code linking to `/verify/:code`.
4. **Public Verification:** Anyone (employers, recruiters) can visit `/verify/:code` without logging in to verify authenticity directly against the database record.

---

## 4. Master Master To-Do Checklist by Phase

```
Phase 1: Planning, Schema & Environment
├── [ ] Design ER Diagram for all 19 normalized MySQL tables
├── [ ] Create database/schema.sql with constraints, foreign keys & indexes
├── [ ] Create database/seed.sql with roles, default admin/instructor, sample HTML/CSS/JS courses
├── [ ] Initialize git repository with clean .gitignore
└── [ ] Set up environment configuration files (.env.example)

Phase 2: Backend & Frontend Foundation (Auth & RBAC)
├── [ ] Backend: Express server skeleton with CORS, Helmet, Rate-Limit
├── [ ] Backend: MySQL2 connection pool setup
├── [ ] Backend: Auth routes (Register, Login, Password Reset, Profile)
├── [ ] Backend: Passwords hashed with bcrypt (cost >= 10)
├── [ ] Backend: JWT verification & role authorization middleware (requireRole)
├── [ ] Frontend: Vite React application setup with Tailwind CSS
├── [ ] Frontend: AuthContext & API Axios client with interceptors
└── [ ] Frontend: ProtectedRoute component for Student, Instructor, and Admin views

Phase 3: Course Catalog & Curriculum Management
├── [ ] Backend: Public course endpoints (Search, Category filter, Level filter)
├── [ ] Backend: Instructor course CRUD (Create, Edit, Publish/Draft, Delete)
├── [ ] Backend: Module and Lesson CRUD with sequential ordering (position)
├── [ ] Frontend: Navigation bar with mobile responsiveness
├── [ ] Frontend: Homepage with search hero and category filter chips
├── [ ] Frontend: Course catalog and course details with curriculum tree
└── [ ] Seed database with at least 1 complete HTML course, 1 CSS course, and 1 JS course

Phase 4: Interactive Practice & Monaco Editor (MANDATORY CORE)
├── [ ] Frontend: Split-screen learning workspace (Markdown Lesson reader + Sidebar)
├── [ ] Frontend: Monaco Editor integration with multi-language tabs (HTML, CSS, JS)
├── [ ] Frontend: Sandboxed <iframe> preview with sandbox="allow-scripts"
├── [ ] Frontend: Console.log interceptor via postMessage displayed in custom terminal
├── [ ] Frontend: Run Code and Reset Code state management
├── [ ] Backend: Exercise CRUD and check_rules JSON storage
├── [ ] Backend: exerciseChecker service (HTML selector check, CSS rule check, text match)
├── [ ] Backend: POST /api/exercises/:id/submit endpoint storing attempts
└── [ ] Frontend: Test feedback drawer showing live pass/fail diagnostics

Phase 5: Assessment, Progress Tracking & Verifiable Certificates
├── [ ] Backend: Quiz & Question CRUD for instructors
├── [ ] Backend: Quiz taking API (hiding correct answers from payload)
├── [ ] Backend: Server-side quiz auto-grading and attempt scoring
├── [ ] Backend: Dynamic progress calculation engine for student dashboard
├── [ ] Backend: POST /api/courses/:id/certificate validation gate
├── [ ] Backend: Dynamic PDF certificate generation with PDFKit and QRCode
├── [ ] Backend: Public GET /verify/:code verification endpoint
├── [ ] Frontend: Interactive Quiz taking interface with countdown timer
├── [ ] Frontend: Student dashboard with "Continue Learning" button and progress rings
└── [ ] Frontend: Certificate view modal with confetti and direct PDF download

Phase 6: Admin Governance, Security Hardening & Final Presentation
├── [ ] Backend: Admin dashboard statistics aggregation API
├── [ ] Backend: Admin user role management and content moderation
├── [ ] Frontend: Admin metrics cards and user data table
├── [ ] Security Review: Sanitize all inputs, parameterize all SQL queries, ensure no token leaks
├── [ ] Testing: Execute test cases (T1 - T11 from project specification)
└── [ ] Prepare 10-minute presentation demo following the student journey script
```

---

## 5. Defense & Evaluator Presentation Guide

### 10-Minute Demonstration Script

1. **Introduction (1 min):** High-level problem statement, the 3 user roles, and the 2 mandatory features (Monaco browser editor + automated exercise evaluation).
2. **Instructor Workflow (2 mins):** Log in as Instructor, showcase course curriculum, and build an exercise showing how JSON `check_rules` are created.
3. **Student Learning Flow (2 mins):** Register new student, browse catalog, enroll, navigate lesson sidebar, mark lesson complete.
4. **Code Execution & Exercise Checking (2 mins):** Open the editor, write HTML/CSS/JS, hit **Run** (show sandboxed preview and console log), hit **Submit** with an error to show diagnostic feedback, fix it, and show the passing badge.
5. **Quiz & Certificate Generation (2 mins):** Complete the quiz, show the auto-updated dynamic progress on the dashboard, generate the PDF certificate, and scan/open the public `/verify/:code` URL.
6. **Administrator Console (1 min):** Show platform-wide metrics, user role management, and certificate audit logs.
