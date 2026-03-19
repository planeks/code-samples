# React + Vite Patterns

A collection of production-grade React patterns using Vite as the build tool. Covers Context API providers for global state (user authentication with RBAC, UI preferences with localStorage persistence), route protection with React Router v6, custom hooks for DOM interactions (resize handling, re-render triggers), and a full Vite configuration with proxy setup, chunk splitting, and Vitest integration.

---

## 1. UserProvider — Context API + useMemo + RBAC

A global user context that fetches the current user and their role via React Query, then exposes helper methods (`getEmail`, `getInitials`, `hasPermissions`) to the entire app. The `hasPermissions` function enables role-based access control (RBAC) by checking if the user's role includes all required permissions.

**Patterns covered:** Context API, useMemo, custom state management, role-based access control (RBAC), React Query

```tsx
import {
  createContext,
  ReactElement,
  useEffect,
  useMemo,
  useState,
} from 'react';
import { useQuery } from 'react-query';
import { CurrentUserData, ListAllRolesResponse, Permissions, RoleSchema } from './types';
import { getCurrentUser } from './api/users';
import { getRoles } from './api/roles';

// Reusable type for permission checks — used across the app in UI guards
export type HasPermissions = (permissions?: Permissions[]) => boolean;

// Explicit interface for the context value — makes it self-documenting
// and prevents accidental omission of required fields
interface IUserContext {
  user: CurrentUserData | null;
  role: RoleSchema | null;
  getEmail(): string;
  getInitials(): string;
  getName(): string;
  getOrganizationName(): string;
  getHashedOrg(): string;
  getHashedUser(): string;

  // Checks whether the current user has ALL the provided permissions.
  // If no permissions are required, access is granted by default.
  hasPermissions: HasPermissions;

  canSwitchTenants: boolean;
  isInternalUser: boolean;
}

// Default context value — used when a consumer is rendered outside the provider.
// Returning safe fallbacks (empty strings, false) prevents runtime crashes.
const UserContext = createContext<IUserContext>({
  user: null,
  role: null,
  getEmail: () => '',
  getInitials: () => '',
  getName: () => '',
  getOrganizationName: () => '',
  getHashedOrg: () => '',
  getHashedUser: () => '',
  hasPermissions: () => false,
  canSwitchTenants: false,
  isInternalUser: false,
});

function UserProvider({ children }: { children: ReactElement }) {
  const [user, setUser] = useState<CurrentUserData | null>(null);
  const [role, setRole] = useState<RoleSchema | null>(null);

  // Fetch current user data via React Query — handles caching and deduplication
  const { data: userData } = useQuery<CurrentUserData>(
    ['current-user'],
    getCurrentUser,
  );

  // Fetch roles only after user is loaded — avoids unnecessary API calls.
  // `enabled: !!user` is a React Query conditional fetching pattern.
  useQuery<ListAllRolesResponse>(['roles'], getRoles, {
    onSuccess: (data) => {
      const roles = data.items;
      // Match user's role_name against the full list of roles
      const matchingRole = roles.find((r) => r.name === user?.role_name);
      setRole(matchingRole ?? null);
    },
    enabled: !!user,
  });

  // Role names that represent internal/global access
  const globalRoleNames = ['global_admin', 'global_member', 'global_viewer'];

  // Helper functions close over `user` state — defined inline for simplicity
  const getEmail = () => user?.email ?? '';
  const getInitials = () =>
    user?.name
      .toUpperCase()
      .split(' ')
      .map((x: string) => x.charAt(0))
      .join('') ?? '';
  const getName = () => user?.name ?? '';
  const getOrganizationName = () => user?.organization_name ?? '';
  const getHashedOrg = () => user?.hashed_org ?? '';
  const getHashedUser = () => user?.hashed_user ?? '';

  // Check if the user belongs to the internal domain
  const isInternalUser = () => getEmail().endsWith('@company.io');

  // Sync React Query result into local state.
  // Kept separate from useQuery to allow manual setUser calls elsewhere if needed.
  useEffect(() => {
    if (userData) {
      setUser(userData);
    }
  }, [userData]);

  return (
    <UserContext.Provider
      // useMemo ensures the context object reference stays stable —
      // prevents all consumers from re-rendering on unrelated state changes
      value={useMemo(
        () => ({
          user,
          role,
          getEmail,
          getInitials,
          getName,
          getOrganizationName,
          getHashedOrg,
          getHashedUser,
          isInternalUser: isInternalUser(),

          // RBAC check: user must have EVERY listed permission.
          // If no permissions are defined on the route, access is open by default.
          hasPermissions(permissions) {
            if (permissions === undefined) return true;
            if (!role) return false;
            return permissions.every((p) => role.permissions?.includes(p));
          },

          // Tenant switching is available for internal users or users with a global role
          canSwitchTenants:
            isInternalUser() || (!!role && globalRoleNames.includes(role.name)),
        }),
        // Only recompute when user or role actually changes
        [user, role],
      )}
    >
      {children}
    </UserContext.Provider>
  );
}

export { UserProvider, UserContext };
```

---

## 2. NavbarProvider — Persistent UI State + Media Query + Consumer Hook Pattern

Manages the sidebar collapsed/expanded state with localStorage persistence so it survives page refreshes. Automatically collapses on mobile screens via a media query. Exposes a custom `useNavbarContext` hook instead of the raw context — a cleaner API that throws a descriptive error if used outside the provider.

**Patterns covered:** Custom consumer hook, localStorage persistence, responsive design via `useMediaQuery`, `useMemo`

```tsx
import { createContext, ReactElement, useContext, useMemo } from 'react';
import useLocalStorageState from 'use-local-storage-state';
import { useMediaQuery } from '@mui/material';

interface INavbarContext {
  isCollapsed: boolean;
  setIsCollapsed: (value: boolean) => void;
}

// Context is typed but NOT exported directly —
// consumers must use the `useNavbarContext` hook below (safer, cleaner API)
const NavbarContext = createContext<INavbarContext | null>(null);

function NavbarProvider({ children }: { children: ReactElement }) {
  // Persist sidebar collapsed state in localStorage so it survives page refresh
  const [isCollapsed, setIsCollapsed] = useLocalStorageState('navbar-collapsed', {
    defaultValue: false,
  });

  // Auto-collapse on small screens — responsive UX without manual handling in CSS
  const isMobile = useMediaQuery('(max-width:768px)');

  return (
    <NavbarContext.Provider
      // useMemo prevents a new object reference on every render,
      // which would cause all navbar consumers to unnecessarily re-render
      value={useMemo(
        () => ({
          isCollapsed: isMobile ? true : isCollapsed,
          setIsCollapsed,
        }),
        [isCollapsed, isMobile],
      )}
    >
      {children}
    </NavbarContext.Provider>
  );
}

// Consumer hook — encapsulates the null-check and provides a clean import API.
// Throws a descriptive error if used outside the provider (fail-fast pattern).
function useNavbarContext(): INavbarContext {
  const context = useContext(NavbarContext);
  if (!context) {
    throw new Error('useNavbarContext must be used within a NavbarProvider');
  }
  return context;
}

export { NavbarProvider, useNavbarContext };
```

---

## 3. ProtectedRoute — Declarative Route Guarding (RBAC + Feature Flags)

A wrapper component for React Router v6 that guards child routes based on permissions and feature flags. If access is denied, it redirects to a fallback route. Keeps the router configuration declarative and easy to audit — guards are visible in the route tree rather than scattered across page components.

**Patterns covered:** React Router v6 `<Outlet>`, declarative access control, composition over imperative checks

```tsx
import { Navigate, Outlet } from 'react-router-dom';
import { useContext } from 'react';
import { UserContext } from './providers/UserProvider';
import { Permissions } from './types';

interface ProtectedRouteProps {
  // Required permissions — user must have ALL of them to access the route
  requiredPermissions?: Permissions[];
  // Feature flag — route is only accessible when the flag is enabled
  featureFlag?: boolean;
  // Redirect target when access is denied
  redirectTo?: string;
}

// Wraps React Router's <Outlet> with permission and feature flag checks.
// Keeping guards here makes the router config declarative and easy to audit:
//
//   <Route element={<ProtectedRoute requiredPermissions={[Permissions.ManageTokens]} />}>
//     <Route path="/tokens" element={<Tokens />} />
//   </Route>
function ProtectedRoute({
  requiredPermissions,
  featureFlag = true,
  redirectTo = '/',
}: ProtectedRouteProps) {
  const { hasPermissions } = useContext(UserContext);

  // Deny access if the feature is toggled off OR user lacks required permissions
  const canAccess = featureFlag && hasPermissions(requiredPermissions);

  // <Outlet> renders the matched child route — standard React Router v6 pattern
  return canAccess ? <Outlet /> : <Navigate to={redirectTo} replace />;
}

export default ProtectedRoute;

```

---

## 4. useDrawerResize — useState + useEffect + useCallback

A custom hook that enables mouse-driven resizing of a drawer or side panel. Tracks resize state and cursor position, calculates width from the right edge, and exposes handlers for the resizer element. All DOM event listener setup and cleanup is encapsulated here.

**Patterns:** `useCallback` for stable references, DOM event listeners, cleanup function

```ts
import { useState, useEffect, useCallback } from 'react';

// Enables mouse-driven resizing of a drawer/panel.
// All resize logic lives here — the consuming component stays clean.
export default function useDrawerResize() {
  const [isResizing, setIsResizing] = useState<boolean>(false);
  const defaultSize = '70vw';
  const [customWidth, setCustomWidth] = useState<string | number>(defaultSize);
  const [isExpanderHovered, setIsExpanderHovered] = useState<boolean>(false);

  const handleMouseDown = () => setIsResizing(true);
  const handleMouseUp = () => setIsResizing(false);

  // useCallback memoizes the handler so the useEffect dependency stays stable.
  // Without it, a new function reference is created every render,
  // causing the effect to re-run and re-register listeners on each render.
  const handleMouseMove = useCallback(
    (e: MouseEvent) => {
      if (!isResizing) return;
      // Measure from the right edge so width grows as the cursor moves left
      const offsetRight = document.body.offsetWidth - e.clientX;
      setCustomWidth(offsetRight);
    },
    [isResizing], // re-create the function only when resize state changes
  );

  const handleMouseLeave = () => setIsExpanderHovered(false);
  const handleMouseOver = () => setIsExpanderHovered(true);

  useEffect(() => {
    // Attach to `document` so dragging works even when cursor leaves the element
    document.addEventListener('mouseup', handleMouseUp, true);
    document.addEventListener('mousemove', handleMouseMove, true);

    // Cleanup — removes listeners on unmount or when handleMouseMove changes.
    // Prevents memory leaks and duplicate listener accumulation.
    return () => {
      document.removeEventListener('mouseup', handleMouseUp, true);
      document.removeEventListener('mousemove', handleMouseMove, true);
    };
  }, [handleMouseMove]); // re-subscribe only when the memoized handler changes

  return {
    isResizing,
    defaultSize,
    customWidth,
    isExpanderHovered,
    handleMouseDown,
    handleMouseLeave,
    handleMouseOver,
  };
}
```

---

## 5. useResizeRerender — useState + useEffect + useRef

A custom hook that triggers a component re-render whenever a referenced DOM element's size changes, using ResizeObserver. Useful for components that need to recalculate layout or redraw (e.g., charts, canvas elements) when their container resizes.

**Patterns:** `useRef` for imperative APIs, triggering re-renders without exposing state

```ts
import { useState, useEffect, useRef, RefObject } from 'react';

// Triggers a re-render whenever a DOM element's size changes.
// Useful for components that need to recalculate layout on resize.
export default function useResizeRerender(ref: RefObject<HTMLElement>) {
  // The state value itself is unused — only the setter matters here.
  // Calling setSize forces React to re-render the consuming component.
  const [, setSize] = useState({ width: 0, height: 0 });

  // useRef stores the observer instance across renders WITHOUT triggering re-renders.
  // This is the correct tool for mutable, non-reactive values (imperative APIs).
  // Using useState here would cause an infinite loop.
  const observer = useRef<ResizeObserver | null>(null);

  useEffect(() => {
    observer.current = new ResizeObserver((entries) => {
      const { width, height } = entries[0].contentRect;
      setSize({ width, height }); // trigger re-render with updated dimensions
    });

    if (ref.current) {
      observer.current.observe(ref.current);
    }

    // Cleanup: disconnect the observer to prevent memory leaks on unmount
    return () => {
      if (observer.current) {
        observer.current.disconnect();
      }
    };
  }, [ref.current]);
}
```

---

## 6. Vite Configuration

A complete Vite config with React Fast Refresh, TypeScript path aliases, SVG-as-component imports, API proxy for local development, manual chunk splitting for optimal caching (React, MUI, React Query, Router in separate bundles), and colocated Vitest configuration for unit testing.

```ts
import { defineConfig, loadEnv } from 'vite';
import react from '@vitejs/plugin-react';
import tsconfigPaths from 'vite-tsconfig-paths';
import svgr from 'vite-plugin-svgr';

// loadEnv gives access to .env variables at config-time (before the app boots).
// `mode` is injected by Vite — "development", "production", etc.
export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), '');

  return {
    plugins: [
      // Enables React Fast Refresh (HMR) and JSX transform.
      // Without this plugin, JSX won't compile and HMR won't work.
      react(),

      // Resolves TypeScript path aliases (e.g. `@/components/Button`)
      // directly from tsconfig.json `paths` — no need to duplicate them here.
      tsconfigPaths(),

      // Allows importing SVG files as React components:
      //   import { ReactComponent as Logo } from './logo.svg'
      // Useful for icon systems without a separate icon library.
      svgr(),
    ],

    resolve: {
      alias: {
        // Explicit fallback alias if tsconfig paths aren't picked up.
        // Maps `@/` to the `src/` directory.
        '@': '/src',
      },
    },

    server: {
      port: 3000,

      // Proxy API requests to the backend during local development.
      // This avoids CORS issues — the browser thinks everything is on port 3000.
      proxy: {
        '/api': {
          target: env.VITE_API_URL,
          changeOrigin: true,   // rewrites the Host header to match the target
          secure: false,        // allows self-signed certs in local backend
        },
      },
    },

    build: {
      outDir: 'dist',

      // Generate source maps in production so errors in monitoring tools
      // (e.g. Sentry, Datadog) can be mapped back to original source code.
      sourcemap: true,

      rollupOptions: {
        output: {
          // Manual chunk splitting — keeps the main bundle small
          // by separating large, rarely-changing vendor libraries.
          // These chunks are cached by the browser independently.
          manualChunks: {
            // React core — almost never changes, safe to cache aggressively
            'vendor-react': ['react', 'react-dom'],

            // MUI is large (~300kb) — isolating it prevents it from
            // invalidating the app bundle on every deployment
            'vendor-mui': ['@mui/material', '@mui/icons-material'],

            // React Query — data-fetching layer, changes rarely
            'vendor-query': ['react-query'],

            // Router — separated so route changes don't bust vendor cache
            'vendor-router': ['react-router-dom'],
          },
        },
      },

      // Warn when any single chunk exceeds 500kb.
      // Keeps developers aware of bundle size regressions.
      chunkSizeWarningLimit: 500,
    },

    // Makes env variables available inside the app as import.meta.env.VITE_*
    // Only variables prefixed with VITE_ are exposed to the client (security default).
    envPrefix: 'VITE_',

    test: {
      // Vitest config — colocated here so no separate vitest.config.ts is needed.
      globals: true,       // allows `describe`, `it`, `expect` without imports
      environment: 'jsdom', // simulates a browser DOM for component tests
      setupFiles: './src/setupTests.ts', // runs before each test file (e.g. jest-dom matchers)
      css: false,          // skip CSS processing in tests — faster, not needed for logic tests
    },
  };
});

