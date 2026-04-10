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
│   ├── client.ts            # Axios instance (baseURL from VITE_API_URL)
│   └── useUsers.ts          # React Query hooks per resource
├── components/              # Shared reusable UI components
├── pages/                   # Route-level components
├── types/                   # Shared TypeScript interfaces
└── main.tsx                 # App entrypoint — QueryClient + Router setup
```

## Code Style
- Prettier + ESLint enforced
- 2-space indentation, double quotes in JSX
- No `any` type — use proper types or `unknown`
- `noUnusedLocals` and `noUnusedParameters` enabled

## Component Rules
- Functional components only — no class components
- One component per file, filename matches component name
- Props typed with explicit `interface`, not inline
- Co-locate `*.test.tsx` files next to the component

## Data Fetching
- All API calls via React Query hooks in `src/api/`
- Never call `axios` / `fetch` directly inside components
- Invalidate queries after mutations via `queryClient.invalidateQueries`
- API base URL from `import.meta.env.VITE_API_URL`

## Environment Variables
- Prefix all Vite env vars with `VITE_`
- Never hardcode API URLs — always read from env

## Testing
- Vitest + React Testing Library
- Test file: `ComponentName.test.tsx` beside the component
