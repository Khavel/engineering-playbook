# Next.js 15 App Router Patterns

## Project Structure

```
web/
├── src/
│   ├── app/              # App Router pages and layouts
│   │   ├── layout.tsx    # Root layout
│   │   ├── page.tsx      # Home page
│   │   ├── admin/        # Admin area (protected routes)
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx
│   │   │   └── disputes/
│   │   └── api/          # API routes (if needed)
│   ├── components/       # Reusable React components
│   │   ├── admin/        # Feature-specific components
│   │   ├── common/       # Shared UI components
│   │   └── forms/        # Form components
│   ├── lib/             # Utilities and helpers
│   │   ├── api.ts       # API client functions
│   │   └── hooks.ts     # Custom React hooks
│   ├── types/           # TypeScript types
│   │   └── api.ts       # API response/request types
│   └── i18n/            # Internationalization
│       └── locales/     # en.ts, es.ts, pt.ts
├── package.json
└── tsconfig.json
```

## API Integration Pattern

Keep API calls in `lib/api.ts` for centralized management:

```typescript
// lib/api.ts
const API_BASE = process.env.NEXT_PUBLIC_API_URL || "http://localhost:5000";

export async function getModerationQueue(type: "disputes" | "reports") {
  const response = await fetch(
    `${API_BASE}/api/admin/moderation/queue?type=${type}`,
    {
      headers: {
        Authorization: `Bearer ${getAuthToken()}`,
      },
    }
  );
  
  if (!response.ok) {
    throw new Error(`Failed to fetch ${type}`);
  }
  
  return response.json() as Promise<ModerationQueueDto>;
}
```

Use in components:
```typescript
"use client";
import { useEffect, useState } from "react";
import { getModerationQueue } from "@/lib/api";

export function DisputesList() {
  const [disputes, setDisputes] = useState([]);
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    getModerationQueue("disputes")
      .then(setDisputes)
      .catch((err) => setError(err.message))
      .finally(() => setLoading(false));
  }, []);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  
  return <div>{/* render disputes */}</div>;
}
```

## TypeScript Types

Always define types for API responses:

```typescript
// types/api.ts
export interface ModerationQueueDto {
  disputes: DisputeDto[];
  reports: ReportDto[];
}

export interface DisputeDto {
  id: number;
  feedbackId: number;
  reason: string;
  initiatorName: string;
  createdAt: string;
}
```

Never use `any` types—use generics and interfaces instead.

## Layout & Navigation

Use layouts for shared UI (navigation, sidebar):

```typescript
// app/admin/layout.tsx
import { AdminSidebar } from "@/components/admin/AdminSidebar";

export default function AdminLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="flex gap-4">
      <AdminSidebar />
      <main className="flex-1">{children}</main>
    </div>
  );
}
```

## Theme Support (Light/Dark)

Use CSS variables for theme-aware styling in `globals.css`:

```css
:root {
  --background: #f5f5f5;
  --foreground: #171717;
  --card-bg: #ffffff;
  --border: #e5e5e5;
}

.dark {
  --background: #0a0a0a;
  --foreground: #ededed;
  --card-bg: #1c1c1c;
  --border: #2a2a2a;
}
```

Apply in components with Tailwind:
```tsx
<div className="bg-[var(--card-bg)] border border-[var(--border)]">
  {/* Content that respects theme */}
</div>
```

## Form Validation

Use `react-hook-form` with `zod` for type-safe validation:

```typescript
import { useForm } from "react-hook-form";
import { z } from "zod";
import { zodResolver } from "@hookform/resolvers/zod";

const schema = z.object({
  reason: z.string().min(1, "Reason required").max(255),
  notes: z.string().max(500).optional(),
});

export function DisputeForm() {
  const { register, handleSubmit, formState: { errors } } = useForm({
    resolver: zodResolver(schema),
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register("reason")} />
      {errors.reason && <span>{errors.reason.message}</span>}
    </form>
  );
}
```

## Internationalization (i18n)

Use the `useTranslation` hook to access translations:

```typescript
// lib/hooks.ts
import { useCallback } from "react";

export function useTranslation() {
  const locale = "en"; // Get from context/localStorage
  const messages = locales[locale];
  
  return useCallback((key: string) => messages[key], [messages]);
}
```

Always use translation keys in UI:
```tsx
<h1>{t("admin.disputes.title")}</h1>
```

## Testing Components

Test with `@testing-library/react`:

```typescript
import { render, screen } from "@testing-library/react";
import { DisputesList } from "./DisputesList";

test("renders dispute list", async () => {
  render(<DisputesList />);
  expect(await screen.findByText(/disputes/i)).toBeInTheDocument();
});
```

Focus on observable behavior, not implementation details.
