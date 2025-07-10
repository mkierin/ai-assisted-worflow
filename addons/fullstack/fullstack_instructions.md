# Full-Stack Web Development Instructions (bhvr Stack)

## Overview

The bhvr stack is a modern, lightweight full-stack TypeScript monorepo approach using:
- **Bun**: Fast JavaScript runtime and package manager
- **Hono**: Lightweight, performant backend framework
- **Vite**: Modern frontend build tool
- **React**: UI component library
- **TypeScript**: End-to-end type safety

## Project Structure

```
.
├── client/               # React frontend
├── server/               # Hono backend
├── shared/               # Shared TypeScript definitions
│   └── src/types/        # Type definitions used by both client and server
└── package.json          # Root package.json with workspaces
```

## Implementation Guide

### 1. Project Structure Setup

Create this structure with the following command:

```bash
mkdir -p my-bhvr-app/{client,server,shared/src/types}
cd my-bhvr-app
```

### 2. Root Configuration

```json
// package.json
{
  "name": "bhvr-project",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "workspaces": [
    "client",
    "server",
    "shared"
  ],
  "scripts": {
    "dev": "concurrently \"bun run dev:shared\" \"bun run dev:server\" \"bun run dev:client\"",
    "dev:client": "cd client && bun run dev",
    "dev:server": "cd server && bun run dev",
    "dev:shared": "cd shared && bun run dev",
    "build": "bun run build:shared && bun run build:client",
    "build:client": "cd client && bun run build",
    "build:shared": "cd shared && bun run build"
  },
  "devDependencies": {
    "concurrently": "^8.2.2",
    "typescript": "^5.3.3"
  }
}
```

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "esModuleInterop": true,
    "strict": true,
    "skipLibCheck": true
  }
}
```

### 3. Shared Package Setup

```json
// shared/package.json
{
  "name": "shared",
  "private": true,
  "version": "0.1.0",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc",
    "dev": "tsc --watch"
  },
  "devDependencies": {
    "typescript": "^5.3.3"
  }
}
```

```json
// shared/tsconfig.json
{
  "extends": "../tsconfig.json",
  "compilerOptions": {
    "outDir": "dist",
    "declaration": true
  },
  "include": ["src/**/*"]
}
```

```typescript
// shared/src/index.ts
export * from "./types";
```

### 4. Shared Types Implementation

```typescript
// shared/src/types/index.ts
export interface ApiResponse<T = unknown> {
  message: string;
  success: boolean;
  data?: T;
}

export interface User {
  id: string;
  name: string;
  email: string;
}
```

### 5. Server Package Setup

```json
// server/package.json
{
  "name": "server",
  "private": true,
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "dev": "bun --watch src/index.ts",
    "start": "bun src/index.ts"
  },
  "dependencies": {
    "hono": "^3.12.0",
    "shared": "workspace:*"
  },
  "devDependencies": {
    "@types/bun": "latest"
  }
}
```

### 6. Server Implementation

```typescript
// server/src/index.ts
import { Hono } from 'hono';
import { cors } from 'hono/cors';
import { logger } from 'hono/logger';
import type { ApiResponse, User } from 'shared';

const app = new Hono();

app.use(logger());
app.use(cors());

// Basic route
app.get('/', (c) => {
  return c.text('API is running');
});

// Example API endpoint
app.get('/api/users', async (c) => {
  const users: User[] = [
    { id: '1', name: 'Alice', email: 'alice@example.com' },
    { id: '2', name: 'Bob', email: 'bob@example.com' }
  ];
  
  const response: ApiResponse<User[]> = {
    message: 'Users retrieved successfully',
    success: true,
    data: users
  };
  
  return c.json(response);
});

// Start the server
const port = process.env.PORT || 3000;
console.log(`Server running at http://localhost:${port}`);

export default {
  port,
  fetch: app.fetch
};
```

### 7. Client Package Setup

```json
// client/package.json
{
  "name": "client",
  "private": true,
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "shared": "workspace:*"
  },
  "devDependencies": {
    "@types/react": "^18.2.43",
    "@types/react-dom": "^18.2.17",
    "@vitejs/plugin-react": "^4.2.1",
    "typescript": "^5.3.3",
    "vite": "^5.0.8"
  }
}
```

```typescript
// client/vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173
  }
});
```

### 8. Client Implementation

```typescript
// client/src/App.tsx
import { useState } from 'react';
import type { ApiResponse, User } from 'shared';
import './App.css';

const API_URL = import.meta.env.VITE_API_URL || 'http://localhost:3000';

function App() {
  const [users, setUsers] = useState<User[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  async function fetchUsers() {
    setLoading(true);
    setError(null);
    
    try {
      const response = await fetch(`${API_URL}/api/users`);
      const result: ApiResponse<User[]> = await response.json();
      
      if (result.success && result.data) {
        setUsers(result.data);
      } else {
        setError(result.message || 'Failed to fetch users');
      }
    } catch (err) {
      setError('An error occurred while fetching users');
      console.error(err);
    } finally {
      setLoading(false);
    }
  }

  return (
    <div className="app">
      <h1>bhvr Full-Stack App</h1>
      
      <div className="card">
        <button onClick={fetchUsers} disabled={loading}>
          {loading ? 'Loading...' : 'Fetch Users'}
        </button>
        
        {error && <div className="error">{error}</div>}
        
        {users.length > 0 && (
          <div className="users">
            <h2>Users</h2>
            <ul>
              {users.map(user => (
                <li key={user.id}>
                  <strong>{user.name}</strong> ({user.email})
                </li>
              ))}
            </ul>
          </div>
        )}
      </div>
    </div>
  );
}

export default App;
```

## Development Workflow

```bash
# Install dependencies
bun install

# Start all development servers
bun run dev

# Individual commands
bun run dev:shared  # Watch and compile shared types
bun run dev:server  # Run Hono backend
bun run dev:client  # Run Vite dev server
```

## Key Implementation Principles

### 1. API Design & Type Synchronization
- Design API contracts with shared types first
- Establish clear boundaries between client and server responsibilities
- Implement consistent error handling and status codes
- Ensure type definitions are propagated correctly between layers
- Validate API responses match type definitions

### 2. State Management Architecture
- Identify state ownership (server vs client vs shared)
- Design optimistic UI updates with fallback patterns
- Implement appropriate caching strategies
- Create clear data fetching patterns
- Establish state synchronization mechanisms

### 3. Component Architecture
- Break UI into logical component hierarchies
- Separate presentational from container components
- Identify reusable component patterns
- Design component prop interfaces
- Implement proper component composition patterns

### 4. Data Flow Optimization
- Analyze data transfer efficiency
- Implement appropriate loading states
- Design pagination and infinite scroll patterns
- Optimize API payload size and structure
- Implement proper data transformation layers

### 5. Authentication & Authorization
- Design secure authentication flows
- Implement proper token management
- Create role-based access control systems
- Secure API endpoints with appropriate middleware
- Handle authentication state across application

### 6. Performance Optimization
- Implement code splitting and lazy loading
- Optimize bundle size and dependencies
- Design efficient rendering patterns
- Implement proper memoization strategies
- Create performance monitoring mechanisms

## Validation Checklist

### Type Safety
- [ ] Shared types are properly exported and imported
- [ ] API responses match defined interfaces
- [ ] No `any` types in critical code paths
- [ ] Type guards are used for runtime validation
- [ ] Generic types are properly constrained

### API Implementation
- [ ] All endpoints return proper status codes
- [ ] Error handling is consistent across endpoints
- [ ] Authentication is properly implemented
- [ ] Rate limiting is considered for public endpoints
- [ ] API documentation is generated and up-to-date

### Frontend Components
- [ ] Components follow single responsibility principle
- [ ] Props are properly typed with defaults where appropriate
- [ ] State management follows established patterns
- [ ] Components handle loading and error states
- [ ] UI is responsive and accessible

### Performance
- [ ] Bundle size is optimized
- [ ] Code splitting is implemented for larger applications
- [ ] Critical rendering paths are optimized
- [ ] Network requests are minimized and efficient
- [ ] Proper caching strategies are implemented

### Security
- [ ] Authentication tokens are properly stored
- [ ] CSRF protection is implemented
- [ ] Input validation is thorough
- [ ] Environment variables are properly used
- [ ] No sensitive information is exposed to the client

### Testing
- [ ] Unit tests cover critical business logic
- [ ] API endpoints have integration tests
- [ ] UI components have appropriate tests
- [ ] End-to-end tests cover critical user flows
- [ ] Test coverage meets project standards

## Enhancement Options

### UI Enhancements
- shadcn/ui for component library
- Tailwind CSS for styling
- React Router for navigation
- React Query / SWR for data fetching

### Database Options
- Supabase
- Drizzle ORM
- Prisma ORM
- Direct database drivers

### Deployment Options
- **Client**: Orbiter, GitHub Pages, Netlify, Cloudflare Pages
- **Server**: Cloudflare Workers, Bun, Node.js, Deno Deploy
