# Interactive E-Learning Platform

> A production-grade, client-ready interactive web learning platform inspired by W3Schools. Features in-browser isolated code execution, automated rule-based exercise checking, and verifiable PDF certificates with scannable QR codes.

---

## 📁 Repository Structure

```
Finalproject/
├── docs/                                  # Centralized Technical Documentation
│   ├── PROJECT_OVERVIEW_AND_TODOLIST.md   # Master Roadmap, Success Test & Checklist
│   ├── BACKEND_ARCHITECTURE_AND_ROADMAP.md# Express, MySQL Schema (19 Tables) & Services
│   └── FRONTEND_ARCHITECTURE_AND_ROADMAP.md# React, Tailwind, Monaco Editor & Sandboxed Iframe
├── backend/                               # Node.js & Express REST API
│   ├── database/                          # Schema, seed scripts & migrations
│   └── src/                               # Controllers, models, routes, services & utils
└── frontend/                              # Vite + React Client Application
    ├── public/                            # Static assets, icons, and images
    └── src/                               # Components, pages, hooks, context & services
```

---

## 📖 Documentation Index

All in-depth architecture designs, database specifications, and development roadmaps are located in the [`docs/`](./docs/) directory:

1. [**Master Project Overview & Execution Plan**](./docs/PROJECT_OVERVIEW_AND_TODOLIST.md)
   - Evaluator demo flow and success test criteria.
   - Core concepts: client-side iframe sandboxing, automated rule checking, dynamic progress engine.
   - Master 6-phase checklist.

2. [**Backend Architecture & Implementation Roadmap**](./docs/BACKEND_ARCHITECTURE_AND_ROADMAP.md)
   - Controller-Service-Repository pattern.
   - Complete 19-table normalized relational MySQL schema.
   - Exercise evaluation engine (Cheerio DOM & PostCSS rule checker).
   - Dynamic PDF certificate generator (PDFKit & QRCode).

3. [**Frontend Architecture & Implementation Roadmap**](./docs/FRONTEND_ARCHITECTURE_AND_ROADMAP.md)
   - Component hierarchy & design system (Indigo `#4F46E5`, Amber `#F59E0B`, Slate `#0F172A`).
   - Monaco Editor multi-tab workspace integration.
   - Isolated sandboxed iframe runtime & virtual console interceptor.
   - Role-based routing (Student, Instructor, Admin).

---

## 🚀 Tech Stack

- **Frontend:** React 18+ (Vite), Tailwind CSS, Monaco Editor (`@monaco-editor/react`), Lucide React, Axios.
- **Backend:** Node.js, Express.js (ES Modules), MySQL2 (Connection Pool), JWT, Bcrypt, Cheerio, PDFKit, QRCode.
- **Security:** Strict `<iframe sandbox="allow-scripts">` isolation (zero parent storage/cookie access), parameterized SQL queries, rate limiting, and RBAC middleware.

