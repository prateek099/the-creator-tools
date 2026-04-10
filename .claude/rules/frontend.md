---
paths:
  - "ct-frontend/**/*.{ts,tsx}"
---

# Frontend Rules — React + TypeScript + Vite

## Stack
- React 18, TypeScript (strict), Vite, React Query v5, React Router v6, Axios

## Project Layout
```
ct-frontend/src/
├── api/
│   ├── client.ts            # Axios instance + Bearer interceptor + auto-refresh
│   ├── useAuth.ts           # useMe, useLogin, useRegister, useLogout
│   └── useUsers.ts          # React Query hooks per resource
├── components/
│   ├── Navbar.tsx           # Top bar with user info + logout
│   └── ProtectedRoute.tsx   # Redirects to /login if unauthenticated
├── context/
│   └── AuthContext.tsx      # AuthProvider + useAuth() hook
├── pages/
│   ├── LoginPage.tsx        # Login + Register (toggled, single page)
│   └── UsersPage.tsx        # Protected page
├── types/
│   ├── auth.ts              # LoginRequest, TokenResponse, AuthUser
│   └── user.ts              # User, UserCreate
└── main.tsx                 # QueryClient + Router + AuthProvider wiring
```

## Auth Rules
- Tokens stored in `localStorage` (`access_token`, `refresh_token`)
- `client.ts` attaches `Authorization: Bearer <token>` on every request
- On 401, auto-refresh via `refresh_token` then retry once — redirect to `/login` if refresh fails
- Use `useAuth()` (from `AuthContext`) to read `user`, `isAuthenticated`, `logout`
- Use `<ProtectedRoute>` to guard any route that requires login
- After login/register, redirect to `location.state.from` (or `/`)

## Component Rules
- Functional components only — no class components
- One component per file, filename matches component name
- Props typed with explicit `interface`, not inline
- Co-locate `*.test.tsx` files next to the component

## Data Fetching
- All API calls via React Query hooks in `src/api/`
- **Never** call `axios`/`fetch` directly inside components
- Invalidate queries after mutations via `queryClient.invalidateQueries`
- Auth-specific queries use key `["auth", "me"]` — invalidate on login/logout

## Code Style
- Prettier + ESLint enforced
- 2-space indentation, double quotes in JSX
- No `any` type — use proper types or `unknown`
- `noUnusedLocals` and `noUnusedParameters` enabled

## Environment Variables
- Prefix all Vite env vars with `VITE_`
- `VITE_API_URL` — backend base URL (default `/api/v1` via Vite proxy)
- Never hardcode API URLs

## Testing
- Vitest + React Testing Library
- Test file: `ComponentName.test.tsx` beside the component
