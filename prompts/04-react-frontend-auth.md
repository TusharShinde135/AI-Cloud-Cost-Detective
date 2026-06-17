# Prompt 4: React Frontend + Custom JWT Auth

Create the React frontend in a `frontend/` folder with custom JWT authentication.

## What to build

### Setup

- Use Vite + React + TypeScript.
- Use Tailwind CSS for styling. Clean, modern, dark-themed UI with AWS-inspired colors.

### Auth (Custom JWT)

- Add auth endpoints to the FastAPI backend:
  - `POST /api/auth/signup` — accepts email + password, hashes password with bcrypt, stores in `users` table, returns JWT.
  - `POST /api/auth/login` — validates credentials, returns JWT.
- Use `PyJWT` and `bcrypt` on the backend. Store `JWT_SECRET` in `.env`.
- On the frontend, store JWT in localStorage. Include it in all API requests via `Authorization: Bearer <token>` header.
- Redirect to login page if not authenticated.
- Add "Logout" button in navbar.

### Pages

1. **Login / Signup** — Clean auth forms with email and password. Links to switch between login and signup.

2. **Dashboard** — AWS account selection and analysis trigger:
   - Dropdown/input to select AWS Account ID
   - Dropdown to select AWS Region (pre-populated: us-east-1, us-west-2, eu-west-1, ap-southeast-1, etc.)
   - "Run Analysis" button
   - Progress section showing live status (connected to WebSocket)
   - Recent analyses quick view

3. **Analysis Report** — Displays the AI analysis:
   - Summary card: total resources scanned, issues found, estimated monthly savings, analysis timestamp
   - Severity filter buttons (Show all / High / Medium / Low)
   - Issues list with:
     - Resource name and type
     - Issue description
     - Severity badge (color-coded)
     - Estimated savings
     - Fix command in copyable code block
   - Quick wins section (top 3 actions)
   - Export as PDF/JSON option

4. **History** — Lists past analyses:
   - Paginated table with columns: Date, Account, Region, Resources Scanned, Issues, Estimated Savings
   - Search/filter by account, region, or date range
   - Click to open full report
   - Delete analysis option

5. **Settings** — User profile and preferences:
   - Email display
   - Change password
   - Notification preferences
   - Delete account option

### Project structure

```
frontend/
├── src/
│   ├── App.tsx
│   ├── main.tsx
│   ├── pages/
│   │   ├── Login.tsx
│   │   ├── Signup.tsx
│   │   ├── Dashboard.tsx
│   │   ├── Report.tsx
│   │   ├── History.tsx
│   │   └── Settings.tsx
│   ├── components/
│   │   ├── ProgressTracker.tsx
│   │   ├── Navbar.tsx
│   │   ├── IssueCard.tsx
│   │   ├── SeverityBadge.tsx
│   │   └── CodeBlock.tsx
│   ├── hooks/
│   │   ├── useAuth.ts
│   │   └── useWebSocket.ts
│   ├── utils/
│   │   ├── api.ts
│   │   └── auth.ts
│   └── styles/
│       └── tailwind.css
├── package.json
├── tailwind.config.js
├── vite.config.ts
└── index.html
```

### Backend Updates

Update `backend/requirements.txt` — add `PyJWT`, `bcrypt`.

Add to `backend/main.py`:
- JWT middleware to validate tokens on protected routes
- `/api/auth/signup` endpoint
- `/api/auth/login` endpoint
- `/api/auth/verify` endpoint to check token validity

### UI/UX Tips

- Use AWS orange/amber color scheme for primary actions
- Show AWS service icons next to resource types
- Display estimated savings prominently (use green for positive impact)
- Add loading states and error messages
- Include tooltips for AWS-specific terms

Refer to `Architecture.MD`, `RequestFlow.MD`, and `AWS_MIGRATION_GUIDE.md`. This covers step ① of the request flow and the UI layer.
