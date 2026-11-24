# Interactive E-Learning Platform — Frontend Architecture & Phase-by-Phase Implementation Guide

> **Project Standard:** Production / Client-Ready Web Application  
> **Target Stack:** React 18+ (Vite), Tailwind CSS, React Router v6, Monaco Editor (`@monaco-editor/react`), Lucide React, Axios  
> **UI/UX Standard:** Original design system (W3Schools-inspired simplicity with modern SaaS aesthetics), WCAG 2.1 AA accessible

---

## 1. System Architecture & Component Hierarchy

```
   [ App Root ] (BrowserRouter, AuthProvider, Theme/Notification Providers)
        │
   [ Route Guards ] (PublicRoute, ProtectedRoute by Role: Student / Instructor / Admin)
        │
   ├── Public Pages
   │    ├── HomePage (Hero search, category pills, featured tracks)
   │    ├── CourseCatalogPage (Filters: Category, Difficulty, Search query)
   │    ├── CourseDetailsPage (Syllabus tree, objectives, enrollment CTA)
   │    └── CertificateVerifyPage (/verify/:code - Public proof of badge)
   │
   ├── Account Pages
   │    ├── LoginPage / RegisterPage (Form validation, error banners)
   │    └── ProfilePage (Avatar, personal info, active sessions)
   │
   ├── Student Learning Portal
   │    ├── StudentDashboard (Progress rings, "Resume Learning" shortcut, quiz history)
   │    ├── LessonWorkspacePage
   │    │    ├── ModuleLessonSidebar (Collapsible hierarchy, completed checkmarks)
   │    │    ├── LessonContent (Syntax highlighted markdown, next/prev navigation)
   │    │    └── InlinePracticeWidget (Quick try-it snippet preview)
   │    ├── ExerciseEditorPage (Mandatory Core Feature)
   │    │    ├── MultiTabMonacoEditor (HTML / CSS / JavaScript tabs)
   │    │    ├── ActionToolbar (Run [Amber], Reset, Submit [Indigo])
   │    │    ├── SandboxedIframePreview (Isolated runtime, zero parent access)
   │    │    ├── ConsoleOutputDrawer (Captured postMessage console.log stream)
   │    │    └── TestResultsPanel (Green pass checks / Red failure rule messages)
   │    └── QuizTakingPage (Question paginator, timer countdown, submission summary)
   │
   ├── Instructor Studio
   │    ├── InstructorDashboard (Published courses, student count, average scores)
   │    ├── CourseEditor (Metadata, curriculum module drag-and-drop / ordering)
   │    ├── LessonBuilder (Markdown editor with live preview, starter code definition)
   │    ├── ExerciseRuleBuilder (Visual rule builder for HTML/CSS/Regex tests)
   │    └── QuizManager (Question pool: MCQ, True/False, Short answer)
   │
   └── Admin Governance Console
        ├── AdminDashboard (Platform stats: active users, total certificates, courses)
        ├── UserManagementTable (Role changer: student <-> instructor <-> admin)
        └── CertificateRegistry (Searchable database of issued certificates)
```

---

## 2. Design System & Style Specification

| Design Element | Specification & Values | Purpose in Platform |
| :--- | :--- | :--- |
| **Primary Colour** | Indigo (`#4F46E5` / Tailwind `indigo-600`) | Main brand buttons, active lesson highlight, primary links |
| **Accent Colour** | Amber (`#F59E0B` / Tailwind `amber-500`) | **Run Code** action button, badges, warning states, streak highlights |
| **Success Colour** | Emerald Green (`#16A34A` / Tailwind `emerald-600`) | Exercise passed alert, completed lesson checkmarks, passing quiz scores |
| **Error / Alert** | Crimson Red (`#DC2626` / Tailwind `red-600`) | Exercise failed checks, syntax errors, validation failures |
| **Surfaces** | Soft Gray (`#F8FAFC`), Pure White (`#FFFFFF`) | Main background contrast, clean content card cards |
| **Code Surfaces**| Slate Dark (`#0F172A`) | Monaco Editor background, sandboxed console terminal drawer |
| **Typography** | `Inter` (UI elements), `JetBrains Mono` / `Fira Code` (Code tabs) | High legibility for long tutorials and exact code alignments |
| **Geometry** | 8px base grid, `rounded-xl` (12px), subtle elevations (`shadow-sm` to `shadow-md`) | Modern, friendly, polished client-ready aesthetic |

---

## 3. Mandatory Core Feature: In-Depth Technical Specification

### 3.1 Monaco Editor & Isolated Sandboxed Iframe Architecture
To prevent malicious code from compromising the host app (e.g., stealing student JWTs or reading cookies), the preview executes inside a strictly sandboxed iframe.

#### Technical Execution Flow:
```
 [ Monaco Editors: HTML, CSS, JS ]
                │
         (User clicks "Run")
                │
 [ Document Assembler ]
   Injects CSS inside <style>, HTML in <body>, and a console-override script before JS
                │
 [ Sandboxed <iframe> ]
   Attributes: sandbox="allow-scripts" (Strictly NO "allow-same-origin")
                │
 [ Virtual Console Interceptor inside Iframe ]
   console.log(...) -> window.parent.postMessage({ type: 'CONSOLE_LOG', payload }, '*')
                │
 [ Host React App Listener ]
   window.addEventListener('message', ...) -> Appends to Console Terminal State
```

#### Code Implementation Blueprint (`SandboxedRunner.jsx`):
```javascript
export function generatePreviewDoc(html, css, js) {
  return `
    <!DOCTYPE html>
    <html lang="en">
      <head>
        <meta charset="UTF-8" />
        <style>
          ${css}
        </style>
        <script>
          // Override console.log to broadcast to parent window
          (function() {
            const originalLog = console.log;
            console.log = function(...args) {
              window.parent.postMessage({
                type: 'CONSOLE_LOG',
                data: args.map(arg => typeof arg === 'object' ? JSON.stringify(arg) : String(arg)).join(' ')
              }, '*');
              originalLog.apply(console, args);
            };
            window.onerror = function(message, source, lineno, colno, error) {
              window.parent.postMessage({
                type: 'CONSOLE_ERROR',
                data: message + ' (line ' + lineno + ')'
              }, '*');
            };
          })();
        <\/script>
      </head>
      <body>
        ${html}
        <script>
          try {
            ${js}
          } catch (err) {
            console.error(err);
          }
        <\/script>
      </body>
    </html>
  `;
}
```

---

## 4. Frontend Phase-by-Phase Roadmap & Action Plan

### Phase 1: Client Tooling & Design System Setup
- [ ] Initialize React application with Vite: `npm create vite@latest client -- --template react`.
- [ ] Install dependencies:
  - Styling & UI: `tailwindcss`, `postcss`, `autoprefixer`, `lucide-react`, `clsx`, `tailwind-merge`
  - Routing & State: `react-router-dom`, `axios`
  - Code Editor: `@monaco-editor/react`
  - Feedback & Effects: `canvas-confetti` (for certificate celebration)
- [ ] Configure `tailwind.config.js` with project custom palette (Indigo `#4F46E5`, Amber `#F59E0B`, Slate `#0F172A`).
- [ ] Configure Axios base instance (`src/services/api.js`) with request interceptor attaching `Bearer <token>` from `localStorage` and response interceptor catching `401 Unauthorized` for automatic logout.

### Phase 2: Authentication Views & Role-Based Routing
- [ ] **Auth Context (`src/context/AuthContext.jsx`)**:
  - Global state: `user` (`{ id, name, email, role }`), `token`, `isAuthenticated`, `loading`.
  - Methods: `login(email, password)`, `register(userData)`, `logout()`.
- [ ] **Role Protection Router (`src/components/routing/ProtectedRoute.jsx`)**:
  - Validates authentication status.
  - Validates allowed roles: e.g. `<ProtectedRoute allowedRoles={['admin']}><AdminDashboard /></ProtectedRoute>`.
  - Redirects unauthorized users with toast alerts.
- [ ] **Authentication Pages**:
  - `LoginPage.jsx`: Clean card layout, email/password validation, link to forgot password.
  - `RegisterPage.jsx`: Full name, email, password confirmation.
  - `ProfilePage.jsx`: View account details, role badge, update profile form.

### Phase 3: Public Catalog & Course Discovery
- [ ] **Navigation & Shell (`Navbar.jsx`, `Footer.jsx`)**:
  - Responsive header with role-aware navigation links (My Dashboard, Instructor Studio, Admin Panel).
  - Mobile slide-out drawer menu.
- [ ] **Homepage (`HomePage.jsx`)**:
  - Hero search bar with instant autocomplete.
  - Technology category pill filters (HTML5, CSS3, JavaScript ES6+).
  - Featured course cards highlighting estimated hours, student count, and difficulty badges.
- [ ] **Course Details Page (`CourseDetailsPage.jsx`)**:
  - Course header: Title, banner, instructor avatar, enrollment status.
  - Expandable accordion curriculum view (Modules & Lessons).
  - One-click "Enroll Now" CTA button updating user state.

### Phase 4: Interactive Practice & Monaco Code Workspace (Mandatory Core)
- [ ] **Lesson Tutorial Shell (`LessonPage.jsx`)**:
  - Left collapsible curriculum drawer showing all modules and checkmarked completed lessons.
  - Main panel: Rich markdown tutorial reader with syntax-highlighted code examples.
  - Bottom navigation bar: "Previous Lesson", "Mark as Completed", "Next Lesson".
- [ ] **Interactive Code Editor (`EditorWorkspace.jsx`)**:
  - Tab bar: HTML (index.html), CSS (styles.css), JavaScript (script.js).
  - Integrated Monaco instance with automatic syntax highlighting and autocompletion.
  - Control bar:
    - **Run Code** (`#F59E0B` Amber button) -> Re-renders sandboxed iframe.
    - **Reset Code** -> Prompts confirmation and restores original starter code.
    - **Submit Exercise** (`#4F46E5` Indigo button) -> Sends payload to API.
  - Split viewport: Resizable desktop split (Editor on left, Live preview on right); tabbed on mobile.
  - Sandboxed `<iframe>` implementation with `sandbox="allow-scripts"` and `postMessage` console listener.
  - Console output panel tab showing logged output.
  - Test Feedback Modal/Drawer: Displays passing indicators or red diagnostic messages returned by backend rule checks.

### Phase 5: Assessments, Dashboard & Verifiable Certificates
- [ ] **Quiz Interface (`QuizPage.jsx`)**:
  - Countdown timer header component.
  - Question card rendering Multiple Choice, True/False, or Short Answer.
  - Review screen and "Submit Exam" confirmation dialog.
  - Result score card: Pass/Fail banner, percentage score, breakdown of mistakes.
- [ ] **Student Dashboard (`StudentDashboard.jsx`)**:
  - Enrolled courses grid with dynamic animated progress bars.
  - "Continue Where You Left Off" prominent card linking directly to `enrollments.last_lesson_id`.
  - Quiz history table (scores, attempts, pass/fail status).
  - Earned certificates trophy shelf.
- [ ] **Certificate Display & Verification (`CertificateModal.jsx`, `VerifyPage.jsx`)**:
  - Digital certificate preview with official seal, student name, course title, and completion date.
  - Confetti burst animation upon initial certification grant.
  - One-click "Download PDF" button calling `/api/certificates/:code/pdf`.
  - Public route `/verify/:code`: Accessible to anyone, rendering verification validity badge and metadata.

### Phase 6: Instructor Studio & Admin Governance UI
- [ ] **Instructor Studio (`InstructorDashboard.jsx`)**:
  - Course creator wizard: Title, category dropdown, difficulty, syllabus builder.
  - Lesson content editor with split Markdown preview.
  - Exercise rule builder: UI to configure `check_rules` JSON without typing raw JSON.
  - Quiz builder: Add questions, define options, mark correct answer.
- [ ] **Administrator Control Center (`AdminDashboard.jsx`)**:
  - Statistical summary widgets (Total Students, Courses, Active Exercises, Issued Certificates).
  - Data table of all platform users with instant role-switching dropdown (`student`, `instructor`, `admin`).
  - Course moderation table to toggle `draft` / `published` status.
