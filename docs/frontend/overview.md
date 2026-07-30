# Shieldgrid Web — Frontend Overview

## Architecture

`shieldgrid-web` is the single-pane-of-glass operator dashboard for Shieldgrid. It is built as a single-page application (SPA) using React, TypeScript, Vite, and Tailwind CSS.

### Tech Stack
- **Framework:** React 19 + TypeScript 5
- **Build Tool:** Vite 8
- **Styling:** Tailwind CSS v4 (@theme design tokens)
- **Routing:** React Router v7 (`react-router-dom`)
- **API Communication:** Native `fetch` wrapper (`src/lib/api.ts`)

### Project Structure
```
src/
├── components/          # Reusable UI components & layouts
│   ├── Layout.tsx       # Main navigation layout with sidebar
│   ├── ProtectedRoute.tsx # Route-level authentication & role gate
│   ├── LoadingSkeleton.tsx # Shared skeleton loading indicator
│   ├── ErrorDisplay.tsx  # Shared API error container with retry
│   └── EmptyState.tsx   # Inviting empty-state display
├── lib/
│   ├── api.ts           # Typed API client with auto Bearer token header
│   ├── auth.tsx         # AuthContext with strict in-memory JWT storage
│   └── types.ts         # TypeScript interfaces matching backend models
├── pages/               # Route page views
│   ├── LoginPage.tsx    # Authentication entry point
│   ├── DashboardPage.tsx # Overview dashboard & Core Orbit node graph
│   ├── AlertsPage.tsx   # Merged alert triage queue
│   ├── CasesPage.tsx    # Incident case list & creation
│   ├── CaseDetailPage.tsx # Investigation workspace & alert linking
│   ├── ConnectorsPage.tsx # Read-only connector health view
│   └── AuditPage.tsx    # Immutable system audit trail
└── routes/
    └── index.tsx        # Central router definitions
```

## Authentication & Token Management
Authentication uses JWT tokens issued by `shieldgrid-core`. For security, tokens are stored **in-memory only** within the React state (`AuthProvider`). Tokens are never persisted to `localStorage` or `sessionStorage` to mitigate Cross-Site Scripting (XSS) session hijacking risks.
