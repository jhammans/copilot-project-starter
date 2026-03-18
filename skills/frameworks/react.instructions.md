---
applyTo: "**/*.tsx,**/*.jsx,**/components/**,**/pages/**,**/app/**"
---

# React Coding Standards

Apply these standards to all React applications.

---

## Project Setup

- React version: **19+**
- TypeScript: **required** (see `skills/languages/typescript.instructions.md`)
- Build tool: **Vite** (for SPAs) or **Next.js** (for SSR/SSG — see separate instructions)
- State management: **Zustand** (simple), **TanStack Query** (server state), **Redux Toolkit** (complex)
- Forms: **React Hook Form** + **Zod** for validation

---

## Component Architecture

### Component Types

```
components/
├── ui/                    # Primitive, reusable UI components (Button, Input, Modal)
├── features/              # Feature-specific components
│   └── users/
│       ├── UserCard.tsx
│       ├── UserList.tsx
│       └── UserForm.tsx
├── layouts/               # Layout wrappers (AppLayout, AuthLayout)
└── providers/             # Context providers
```

### Component Guidelines

```tsx
// ✅ Good — typed props with defined interface
interface UserCardProps {
  user: User;
  onEdit?: (id: string) => void;
  className?: string;
}

// ✅ Good — named export (enables better tree-shaking and IDE support)
export function UserCard({ user, onEdit, className }: UserCardProps) {
  return (
    <article className={cn('card', className)}>
      <h2>{user.firstName} {user.lastName}</h2>
      <p>{user.email}</p>
      {onEdit && (
        <button type="button" onClick={() => onEdit(user.id)}>
          Edit
        </button>
      )}
    </article>
  );
}

// ❌ Bad — default export for components (harder to refactor)
export default function UserCard(props: any) { ... }
```

### Hooks Rules

```tsx
// ✅ Good — custom hook encapsulates logic
function useUserById(id: string) {
  return useQuery({
    queryKey: ['users', id],
    queryFn: () => userApi.getById(id),
    staleTime: 5 * 60 * 1000,
  });
}

// ✅ Custom hooks must start with 'use'
function useDocumentTitle(title: string) {
  useEffect(() => {
    document.title = title;
    return () => { document.title = 'My App'; };
  }, [title]);
}
```

---

## Security Patterns

### Never Use `dangerouslySetInnerHTML` with Unsanitized Content

```tsx
// ❌ BAD — XSS vulnerability
function UserBio({ bio }: { bio: string }) {
  return <div dangerouslySetInnerHTML={{ __html: bio }} />;
}

// ✅ Good — sanitize before rendering HTML
import DOMPurify from 'dompurify';

function UserBio({ bio }: { bio: string }) {
  const cleanBio = DOMPurify.sanitize(bio);
  return <div dangerouslySetInnerHTML={{ __html: cleanBio }} />;
}

// ✅ Best — display as text unless HTML is truly needed
function UserBio({ bio }: { bio: string }) {
  return <p>{bio}</p>; // React auto-escapes text content
}
```

### Authentication State

```tsx
// ✅ Good — protected route component
interface ProtectedRouteProps {
  requiredPermission?: string;
  children: React.ReactNode;
}

function ProtectedRoute({ requiredPermission, children }: ProtectedRouteProps) {
  const { user, isLoading } = useAuth();

  if (isLoading) return <LoadingSpinner />;
  if (!user) return <Navigate to="/login" replace />;
  if (requiredPermission && !user.permissions.includes(requiredPermission)) {
    return <Navigate to="/unauthorized" replace />;
  }

  return <>{children}</>;
}

// Usage:
<Route path="/admin/users" element={
  <ProtectedRoute requiredPermission="users:manage">
    <AdminUsersPage />
  </ProtectedRoute>
} />
```

### Token Storage

```typescript
// ✅ Good — use in-memory store for access tokens
// Tokens stored in memory are not accessible after page refresh (use refresh token flow)
class TokenStore {
  private static accessToken: string | null = null;

  static set(token: string) { this.accessToken = token; }
  static get(): string | null { return this.accessToken; }
  static clear() { this.accessToken = null; }
}

// ❌ Bad — DO NOT store tokens in localStorage (XSS accessible)
localStorage.setItem('access_token', token);
```

### Content Security Policy

Always define CSP headers on the server that serves the React app:

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self';
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https://cdn.example.com;
  connect-src 'self' https://api.example.com;
  frame-ancestors 'none';
```

---

## Form Handling

```tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const CreateUserSchema = z.object({
  email: z.string().email('Please enter a valid email'),
  firstName: z.string().min(1, 'First name is required').max(100),
  lastName: z.string().min(1, 'Last name is required').max(100),
});

type CreateUserFormData = z.infer<typeof CreateUserSchema>;

export function CreateUserForm({ onSuccess }: { onSuccess: () => void }) {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<CreateUserFormData>({
    resolver: zodResolver(CreateUserSchema),
  });

  const onSubmit = async (data: CreateUserFormData) => {
    await userApi.create(data);
    onSuccess();
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)} noValidate>
      <div>
        <label htmlFor="email">Email</label>
        <input id="email" type="email" {...register('email')} />
        {errors.email && <p role="alert">{errors.email.message}</p>}
      </div>
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Creating...' : 'Create User'}
      </button>
    </form>
  );
}
```

---

## Accessibility (a11y)

All React components must meet **WCAG 2.1 AA** standards:

- Every interactive element must be keyboard navigable
- All images must have descriptive `alt` text (or `alt=""` if decorative)
- Form inputs must have associated `<label>` elements (via `htmlFor` + `id`)
- Error messages must use `role="alert"` or `aria-live="polite"`
- Color alone must not convey information
- Minimum contrast ratio: 4.5:1 for normal text, 3:1 for large text

```tsx
// ✅ Accessible dialog
function ConfirmDialog({ isOpen, title, message, onConfirm, onCancel }) {
  return (
    <dialog 
      open={isOpen}
      aria-modal="true"
      aria-labelledby="dialog-title"
      aria-describedby="dialog-message"
    >
      <h2 id="dialog-title">{title}</h2>
      <p id="dialog-message">{message}</p>
      <button onClick={onConfirm}>Confirm</button>
      <button onClick={onCancel}>Cancel</button>
    </dialog>
  );
}
```

---

## Performance Patterns

```tsx
// ✅ Memoize expensive computations
const sortedUsers = useMemo(
  () => [...users].sort((a, b) => a.lastName.localeCompare(b.lastName)),
  [users]
);

// ✅ Memoize callbacks passed to child components
const handleDelete = useCallback((id: string) => {
  setUsers(prev => prev.filter(u => u.id !== id));
}, []);

// ✅ Lazy load route-level components
const AdminDashboard = React.lazy(() => import('./pages/AdminDashboard'));

// ❌ Don't over-memoize — only when profiling confirms a problem
```
