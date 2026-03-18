---
name: web-fullstack-react-vite
description: Complete React + TypeScript + Vite + Redux development patterns for 2026. Includes RTK Query, modern tooling, and performance optimization.
---

# Web Full-Stack: React + Vite + Redux (2026)

## Architecture Overview

```
src/
├── app/                    # Application layer
│   ├── routes/            # Route definitions (React Router v7)
│   ├── providers.tsx      # Global providers wrapper
│   └── layouts/           # Layout components
├── features/              # Feature-based modules (self-contained)
│   ├── auth/
│   ├── users/
│   └── dashboard/
├── shared/                # Cross-feature code
│   ├── components/
│   ├── hooks/
│   └── lib/
├── types/                 # Shared TypeScript types
└── utils/                 # Utility functions
```

## Core Technologies (2026)

| Technology | Version | Purpose |
|------------|---------|---------|
| **React** | 19.x | UI component library with server components |
| **TypeScript** | 5.8.x | Type safety with strict mode enabled |
| **Vite** | 6.x | Next-gen build tool with HMR |
| **Redux Toolkit** | 2.x | State management with RTK Query |
| **React Router** | 7.x | Declarative routing |
| **Zod** | 3.24.x | Runtime validation with TypeScript inference |

## State Management Strategy

### Global State (Zustand or RTK)
```typescript
// features/users/stores/userStore.ts
import { create } from 'zustand';

interface UserState {
  users: User[];
  loading: boolean;
  error: string | null;
  fetchUsers: () => Promise<void>;
}

export const useUserStore = create<UserState>((set) => ({
  users: [],
  loading: false,
  error: null,
  fetchUsers: async () => {
    set({ loading: true });
    try {
      const response = await api.get('/users');
      set({ users: response.data, loading: false });
    } catch (error) {
      set({ error: error.message, loading: false });
    }
  }
}));
```

### Server State (RTK Query)
```typescript
// features/users/api/usersApi.ts
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

export const usersApi = createApi({
  reducerPath: 'usersApi',
  baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
  endpoints: (builder) => ({
    getUsers: builder.query<User[], void>({
      query: () => '/users',
    }),
    addUser: builder.mutation<User, Partial<User>>({
      query: (userData) => ({
        url: '/users',
        method: 'POST',
        body: userData,
      }),
    }),
  }),
});

export const { useGetUsersQuery, useAddUserMutation } = usersApi;
```

## Component Patterns

### Feature-Scoped Components
```typescript
// features/users/components/UserCard.tsx
import { useGetUsersQuery } from '../api/usersApi';

export function UserCard({ userId }: { userId: string }) {
  const { data: user, isLoading } = useGetUsersQuery();
  
  if (isLoading) return <Spinner />;
  
  return (
    <div className="user-card">
      <h3>{user.name}</h3>
    </div>
  );
}
```

### Custom Hooks
```typescript
// features/users/hooks/useUserPermissions.ts
export function useUserPermissions(userId: string) {
  const { data: user } = use GetUserQuery(userId);
  
  return useMemo(() => {
    return {
      canEdit: user.role === 'admin',
      canDelete: user.role === 'superadmin',
      permissions: user.permissions,
    };
  }, [user]);
}
```

## Build & Development Setup

### Vite Configuration (2026)
```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
      '@features': path.resolve(__dirname, './src/features'),
      '@shared': path.resolve(__dirname, './src/shared'),
    },
  },
  server: {
    port: 5173,
    proxy: {
      '/api': {
        target: 'http://localhost:3000',
        changeOrigin: true,
      },
    },
  },
  build: {
    sourcemap: true,
    rollupOptions: {
      output: {
        manualChunks: {
          react: ['react', 'react-dom', 'react-router-dom'],
          redux: ['@reduxjs/toolkit', 'react-redux'],
          zod: ['zod'],
        },
      },
    },
  },
});
```

### Performance Optimization

#### Code Splitting
```typescript
// Router with lazy loading
const Dashboard = lazy(() => import('@features/dashboard/DashboardPage'));
const Users = lazy(() => import('@features/users/UsersPage'));

const router = createBrowserRouter([
  {
    path: '/dashboard',
    element: (
      <Suspense fallback={<Spinner />}>
        <Dashboard />
      </Suspense>
    ),
  },
]);
```

#### Memoization
```typescript
// Use React.memo for expensive components
export const UserList = memo(({ users }: { users: User[] }) => {
  return (
    <div className="user-list">
      {users.map((user) => (
        <UserCard key={user.id} user={user} />
      ))}
    </div>
  );
});
```

## TypeScript Best Practices

### Type Safety Rules
1. **Strict mode enabled**: `"strict": true` in tsconfig.json
2. **No implicit any**: `"noImplicitAny": true`
3. **Exact optional property types**: `"exactOptionalPropertyTypes": true`
4. **Import types only**: `import type { User } from './types'`

### Advanced Types
```typescript
// Utility types
type Optional<T, K extends keyof T> = Omit<T, K> & Partial<Pick<T, K>>;
type Nullable<T> = T | null;
type DeepReadonly<T> = {
  readonly [K in keyof T]: DeepReadonly<T[K]>;
};

// Discriminated unions
interface CreateUser {
  type: 'create';
  data: User;
}

interface UpdateUser {
  type: 'update';
  id: string;
  data: Partial<User>;
}

type UserAction = CreateUser | UpdateUser;

// Type guards
function isCreateUser(action: UserAction): action is CreateUser {
  return action.type === 'create';
}
```

## Testing Strategy

### Unit Testing
```typescript
// features/users/__tests__/UserCard.test.tsx
import { render, screen } from '@testing-library/react';
import { UserCard } from '../UserCard';

describe('UserCard', () => {
  it('displays user name', () => {
    render(<UserCard user={{ id: '1', name: 'Test User' }} />);
    expect(screen.getByText('Test User')).toBeInTheDocument();
  });
});
```

### Integration Testing
```typescript
// features/users/__tests__/userSlice.test.ts
import { userSlice } from '../stores/userSlice';
import { fetchUsers } from '../api/usersApi';

describe('userSlice', () => {
  it('handles fetchUsers.fulfilled', () => {
    const state = userSlice.getInitialState();
    const newState = userSlice.reducer(state, {
      type: fetchUsers.fulfilled.type,
      payload: [{ id: '1', name: 'Test User' }],
    });
    expect(newState.users).toHaveLength(1);
  });
});
```

## Common Patterns & Anti-Patterns

### Do
✅ Component in `features/[name]/components/`  
✅ Types in `features/[name]/types/`  
✅ API calls in `features/[name]/api/`  
✅ Use `import type` for TypeScript types  
✅ Memoize expensive calculations with `useMemo`  

### Don't
❌ Import from other features (violates feature isolation)  
❌ Mix feature and shared code structure  
❌ Use `any` type  
❌ Export barrel files (`index.ts`) in features  
❌ Nest components more than 3 levels deep  

## References

* [React 19 Documentation](https://react.dev)
* [Vite 6 Guide](https://vitejs.dev)
* [Redux Toolkit 2.x](https://redux-toolkit.js.org)
* [TypeScript 5.8 Handbook](https://typescriptlang.org/docs/handbook)
* [React Router 7](https://reactrouter.com)

## 2026 Updates

* **React 19 Server Components**: Use for data-heavy pages
* **Vite 6 HMR**: Faster development with improved Hot Module Replacement
* **Zod v3.24**: Better TypeScript inference
* **RTK Query**: Default for server state management
* **ESM-First**: All new projects should use ES modules
