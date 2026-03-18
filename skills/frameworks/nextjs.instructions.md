---
applyTo: "**/src/app/**,**/src/pages/**,**/*.server.ts,**/*.server.tsx,**/next.config*"
---

# Next.js Coding Standards

Apply these standards to all Next.js applications. Assumes Next.js 14+ with App Router.

---

## Architecture

### App Router Structure

```
src/
├── app/
│   ├── layout.tsx            # Root layout
│   ├── page.tsx              # Home page
│   ├── (auth)/               # Auth route group (no layout nesting)
│   │   ├── login/
│   │   └── register/
│   ├── (dashboard)/          # Authenticated route group
│   │   ├── layout.tsx        # Dashboard layout (auth guard here)
│   │   └── users/
│   │       ├── page.tsx      # /users
│   │       └── [id]/
│   │           └── page.tsx  # /users/:id
│   └── api/                  # API Route Handlers
│       └── users/
│           └── route.ts
├── components/               # Shared components (see React standards)
├── lib/                      # Utility functions, API clients
├── server/                   # Server-only code (db, auth session)
│   ├── db.ts
│   └── auth.ts
└── types/                    # Shared types
```

---

## Server vs. Client Components

```tsx
// Server Component (default — no directive needed)
// ✅ Fetch data directly; no useState/useEffect; runs on server only
// src/app/users/page.tsx
import { getUsers } from '@/server/users';

export default async function UsersPage() {
  const users = await getUsers(); // direct server-side data access
  return (
    <main>
      <h1>Users</h1>
      <UserList users={users} />
    </main>
  );
}

// Client Component
// ✅ Only add 'use client' when component needs browser APIs or interactivity
'use client';

import { useState } from 'react';

export function UserSearch({ initialUsers }: { initialUsers: User[] }) {
  const [query, setQuery] = useState('');
  const filtered = initialUsers.filter(u => u.email.includes(query));
  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <UserList users={filtered} />
    </>
  );
}
```

---

## Security in Next.js

### Route Handler Security (API Routes)

```typescript
// src/app/api/documents/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { getServerSession } from 'next-auth';
import { authOptions } from '@/server/auth';
import { z } from 'zod';

// ✅ Always authenticate and authorize in route handlers
export async function GET(req: NextRequest) {
  // 1. Authenticate
  const session = await getServerSession(authOptions);
  if (!session?.user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  // 2. Authorize
  if (!session.user.permissions.includes('documents:read')) {
    return NextResponse.json({ error: 'Forbidden' }, { status: 403 });
  }

  // 3. Validate query params
  const { searchParams } = req.nextUrl;
  const page = Math.max(1, parseInt(searchParams.get('page') ?? '1', 10));

  const documents = await documentService.list({ userId: session.user.id, page });
  return NextResponse.json(documents);
}

export async function POST(req: NextRequest) {
  const session = await getServerSession(authOptions);
  if (!session?.user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  // ✅ Validate body with Zod
  const body = await req.json().catch(() => null);
  const result = CreateDocumentSchema.safeParse(body);
  if (!result.success) {
    return NextResponse.json({ error: 'Invalid input', details: result.error.flatten() }, { status: 400 });
  }

  const doc = await documentService.create({ ...result.data, ownerId: session.user.id });
  return NextResponse.json(doc, { status: 201 });
}
```

### Middleware for Auth Protection

```typescript
// middleware.ts — protect routes at the edge
import { withAuth } from 'next-auth/middleware';
import { NextResponse } from 'next/server';

export default withAuth(
  function middleware(req) {
    const token = req.nextauth.token;
    
    // Protect admin routes
    if (req.nextUrl.pathname.startsWith('/admin')) {
      if (!token?.permissions?.includes('admin:access')) {
        return NextResponse.redirect(new URL('/unauthorized', req.url));
      }
    }
    
    return NextResponse.next();
  },
  {
    callbacks: {
      authorized: ({ token }) => !!token, // must be authenticated for all protected routes
    },
  }
);

// Apply middleware to these paths:
export const config = {
  matcher: ['/dashboard/:path*', '/admin/:path*', '/api/documents/:path*'],
};
```

### Security Headers

```typescript
// next.config.ts
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  headers: async () => [
    {
      source: '/(.*)',
      headers: [
        { key: 'X-Content-Type-Options', value: 'nosniff' },
        { key: 'X-Frame-Options', value: 'DENY' },
        { key: 'X-XSS-Protection', value: '1; mode=block' },
        { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
        { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
        {
          key: 'Content-Security-Policy',
          value: [
            "default-src 'self'",
            "script-src 'self' 'nonce-{NONCE}'",  // use nonce for inline scripts
            "style-src 'self' 'unsafe-inline'",
            "img-src 'self' data: https:",
            "connect-src 'self' https://api.example.com",
            "frame-ancestors 'none'",
          ].join('; '),
        },
        {
          key: 'Strict-Transport-Security',
          value: 'max-age=31536000; includeSubDomains; preload',
        },
      ],
    },
  ],
};

export default nextConfig;
```

---

## Data Fetching Patterns

```tsx
// ✅ Server Component — fetch on server, pass to client
// src/app/users/[id]/page.tsx
import { notFound } from 'next/navigation';

export default async function UserPage({ params }: { params: { id: string } }) {
  const user = await userService.getById(params.id);
  if (!user) notFound(); // renders the nearest not-found.tsx
  
  return <UserProfile user={user} />;
}

// ✅ Generate static params for static generation
export async function generateStaticParams() {
  const users = await userService.listAll();
  return users.map(u => ({ id: u.id }));
}

// ✅ Revalidation
export const revalidate = 3600; // revalidate every hour
```

---

## Environment Variables

```typescript
// ✅ Only expose public vars to the client with NEXT_PUBLIC_ prefix
// .env.local (development only — never committed)
DATABASE_URL=postgresql://...
JWT_SECRET=...
NEXT_PUBLIC_API_URL=https://api.dev.example.com

// Access in server code:
const dbUrl = process.env.DATABASE_URL; // server-side only

// Access in client code:
const apiUrl = process.env.NEXT_PUBLIC_API_URL; // safe to expose
```

**Never** put secrets in `NEXT_PUBLIC_*` variables — they are embedded in the JavaScript bundle sent to users.

---

## Image Optimization

```tsx
// ✅ Always use next/image for images
import Image from 'next/image';

function UserAvatar({ user }: { user: User }) {
  return (
    <Image
      src={user.avatarUrl || '/default-avatar.png'}
      alt={`${user.firstName} ${user.lastName} avatar`}
      width={48}
      height={48}
      className="rounded-full"
    />
  );
}

// Configure allowed domains in next.config.ts
const nextConfig = {
  images: {
    remotePatterns: [
      { protocol: 'https', hostname: 'cdn.example.com', pathname: '/avatars/**' },
    ],
  },
};
```
