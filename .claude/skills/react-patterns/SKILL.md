---
name: react-patterns
description: React 19 in practice — Server Components, Server Actions, Suspense, the new hooks, and composition patterns
---

# React patterns

## The use() hook (React 19)

`use()` pulls a value straight out of a Promise or Context during render — and, unlike every other hook, it's allowed inside conditionals and loops.

```tsx
import { use } from 'react';

function UserProfile({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise);
  return <h1>{user.name}</h1>;
}

function ThemeButton() {
  const theme = use(ThemeContext);
  return <button style={{ background: theme.primary }}>Click</button>;
}
```

Any component that feeds a Promise to `use()` needs a `<Suspense>` boundary above it.

## Server Components

```tsx
// app/users/page.tsx - Server Component (default, no directive needed)
import { UserList } from './UserList';

export default async function UsersPage() {
  const users = await fetch('https://api.example.com/users', {
    next: { revalidate: 60 },
  }).then(r => r.json());

  return <UserList users={users} />;
}

// app/users/UserList.tsx - Still a Server Component
export function UserList({ users }: { users: User[] }) {
  return (
    <ul>
      {users.map(u => (
        <li key={u.id}>
          {u.name}
          <DeleteButton userId={u.id} />
        </li>
      ))}
    </ul>
  );
}
```

Drive `'use client'` as far down the tree as you can — only the interactive leaves should become Client Components.

## Server Actions

```tsx
// app/actions.ts
'use server';

import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string;
  const body = formData.get('body') as string;

  await db.insert(posts).values({ title, body });

  revalidatePath('/posts');
  redirect('/posts');
}
```

```tsx
// app/posts/new/page.tsx
import { createPost } from '../actions';

export default function NewPostPage() {
  return (
    <form action={createPost}>
      <input name="title" required />
      <textarea name="body" required />
      <button type="submit">Create</button>
    </form>
  );
}
```

## useActionState (React 19)

```tsx
'use client';

import { useActionState } from 'react';
import { createUser } from './actions';

function SignupForm() {
  const [state, formAction, isPending] = useActionState(createUser, {
    errors: {},
    message: '',
  });

  return (
    <form action={formAction}>
      <input name="email" />
      {state.errors.email && <p>{state.errors.email}</p>}
      <button disabled={isPending}>
        {isPending ? 'Creating...' : 'Sign Up'}
      </button>
      {state.message && <p>{state.message}</p>}
    </form>
  );
}
```

## useOptimistic (React 19)

```tsx
'use client';

import { useOptimistic } from 'react';
import { likePost } from './actions';

function LikeButton({ count, postId }: { count: number; postId: string }) {
  const [optimisticCount, addOptimistic] = useOptimistic(count);

  async function handleLike() {
    addOptimistic(prev => prev + 1);
    await likePost(postId);
  }

  return (
    <form action={handleLike}>
      <button type="submit">{optimisticCount} Likes</button>
    </form>
  );
}
```

## Placing Suspense boundaries

```tsx
import { Suspense } from 'react';

function Dashboard() {
  return (
    <div>
      <Suspense fallback={<StatsSkeleton />}>
        <StatsPanel />
      </Suspense>
      <div className="grid grid-cols-2">
        <Suspense fallback={<ChartSkeleton />}>
          <RevenueChart />
        </Suspense>
        <Suspense fallback={<ListSkeleton />}>
          <RecentActivity />
        </Suspense>
      </div>
    </div>
  );
}
```

Wrap each independent data-fetching unit in its own boundary. A single boundary around the whole page throws away the streaming you're paying for.

## Error boundaries

```tsx
'use client';

import { Component, type ReactNode } from 'react';

class ErrorBoundary extends Component<
  { fallback: ReactNode; children: ReactNode },
  { hasError: boolean }
> {
  state = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  componentDidCatch(error: Error, info: React.ErrorInfo) {
    reportError(error, info.componentStack);
  }

  render() {
    if (this.state.hasError) return this.props.fallback;
    return this.props.children;
  }
}
```

For route-level handling, Next.js's `error.tsx` convention does the same job.

## Custom hooks

```tsx
function useDebounce<T>(value: T, delay: number): T {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debounced;
}

function useLocalStorage<T>(key: string, initial: T) {
  const [value, setValue] = useState<T>(() => {
    const stored = localStorage.getItem(key);
    return stored ? JSON.parse(stored) : initial;
  });

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);

  return [value, setValue] as const;
}
```

Guidelines for custom hooks:
- Name them with a `use` prefix
- Pull logic into a hook once two or more components share it
- Keep each hook to a single concern
- Return a tuple `[value, setter]` or an object `{ data, error, loading }`

## Compound components

```tsx
function Tabs({ children }: { children: ReactNode }) {
  const [active, setActive] = useState(0);
  return (
    <TabsContext value={{ active, setActive }}>
      <div role="tablist">{children}</div>
    </TabsContext>
  );
}

Tabs.Tab = function Tab({ index, children }: { index: number; children: ReactNode }) {
  const { active, setActive } = use(TabsContext);
  return (
    <button
      role="tab"
      aria-selected={active === index}
      onClick={() => setActive(index)}
    >
      {children}
    </button>
  );
};

Tabs.Panel = function Panel({ index, children }: { index: number; children: ReactNode }) {
  const { active } = use(TabsContext);
  if (active !== index) return null;
  return <div role="tabpanel">{children}</div>;
};

// Usage
<Tabs>
  <Tabs.Tab index={0}>Profile</Tabs.Tab>
  <Tabs.Tab index={1}>Settings</Tabs.Tab>
  <Tabs.Panel index={0}><ProfileForm /></Tabs.Panel>
  <Tabs.Panel index={1}><SettingsForm /></Tabs.Panel>
</Tabs>
```

## Keeping it fast

- Don't build fresh objects or arrays inline in JSX props — each one triggers a re-render
- Add `React.memo` only after profiling actually shows wasted re-renders
- Save `useMemo`/`useCallback` for expensive work or stable references handed to memoized children
- Change a `key` when you deliberately want to reset a component's state
- Keep state colocated — as close to where it's used as you can
- Stop prop drilling past two levels; lift to Context or restructure via composition
