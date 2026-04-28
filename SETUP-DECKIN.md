# DeckIn

Self-hosted, email-gated deck viewer. Captures viewer email, geo, user agent, referrer, and time on each slide. Stack: Next.js (App Router) + Supabase + react-pdf, deployed to Vercel.

The viewer ships with a small footer ("DeckIn · Founders use DeckIn to share decks") that links back to the project. Default-on so other founders discover the OSS project. Removable in one block of JSX (see Notes).

This doc assumes an agent (Cursor / Claude Code) is doing the install. Founder asks the agent to "install DeckIn"; the agent reads this file and executes.

---

## Install

### 1. Schema

Run in Supabase SQL Editor. Table names are prefixed `deckin_` so they coexist with existing tables in a shared project.

```sql
create table deckin_viewers (
  id uuid primary key default gen_random_uuid(),
  email text not null unique check (length(email) <= 254),
  first_seen_at timestamptz not null default now(),
  last_seen_at timestamptz not null default now()
);

create table deckin_sessions (
  id uuid primary key default gen_random_uuid(),
  viewer_id uuid not null references deckin_viewers(id) on delete cascade,
  deck_slug text not null check (length(deck_slug) <= 200),
  -- SHA-256 hash of the auth token sent to the client as an httpOnly cookie.
  -- Track and view requests must present a cookie whose hash matches this row.
  token_hash text not null unique check (length(token_hash) = 64),
  started_at timestamptz not null default now(),
  ended_at timestamptz,
  ip text check (length(ip) <= 64),
  user_agent text check (length(user_agent) <= 1000),
  country text check (length(country) <= 8),
  region text check (length(region) <= 64),
  city text check (length(city) <= 128),
  latitude numeric,
  longitude numeric,
  referrer text check (length(referrer) <= 2048)
);

create index deckin_sessions_viewer_idx on deckin_sessions(viewer_id);
create index deckin_sessions_deck_idx on deckin_sessions(deck_slug);
create index deckin_sessions_started_at_idx on deckin_sessions(started_at desc);

create table deckin_slide_events (
  id bigserial primary key,
  session_id uuid not null references deckin_sessions(id) on delete cascade,
  slide int not null check (slide >= 1 and slide <= 1000),
  duration_ms int not null check (duration_ms > 0 and duration_ms <= 600000),
  is_visible boolean not null default true,
  recorded_at timestamptz not null default now()
);

create index deckin_slide_events_session_idx on deckin_slide_events(session_id);
create index deckin_slide_events_session_slide_idx on deckin_slide_events(session_id, slide);
create index deckin_slide_events_recorded_at_idx on deckin_slide_events(recorded_at desc);
```

### 2. Supabase setup

**If the founder already has a Supabase project**: paste the schema into SQL Editor, then grab `Project Settings → API → Project URL` and `service_role` key. Done.

**If not**: create a project at supabase.com (free tier), wait for provisioning, run schema, grab the same two values.

### 3. Project scaffold

```bash
npx create-next-app@latest deckin --typescript --tailwind --app --src-dir=false --import-alias="@/*" --eslint --no-turbopack
cd deckin
npm install @supabase/supabase-js react-pdf
```

Do **not** install `pdfjs-dist` separately. `react-pdf` bundles its own pinned version. Installing a different version alongside it causes a worker/runtime version mismatch and a "Failed to load PDF file" error.

After installing, copy the worker from react-pdf's bundled pdfjs-dist:

```bash
# With npm/pnpm — find the version react-pdf pinned and copy from there
node -e "console.log(require('react-pdf/package.json').dependencies['pdfjs-dist'])"
# then:
cp node_modules/pdfjs-dist/build/pdf.worker.min.mjs public/
# If pnpm hoisted it differently, find it with:
# find node_modules -path "*/pdfjs-dist/build/pdf.worker.min.mjs" | head -1
```

The `cp` puts the worker into `/public` so it's self-hosted instead of pulled from a CDN. The worker version must match the pdfjs-dist version react-pdf uses — re-copy after every `npm update` of `react-pdf`.

### 4. Environment variables

**If installing into an existing Next.js + Supabase project**, check what env vars are already set before adding new ones. Common situations:

- `NEXT_PUBLIC_SUPABASE_URL` already exists → `lib/deckin-supabase.ts` will use it as a fallback; no need to add `SUPABASE_URL`.
- `SUPABASE_SERVICE_ROLE_KEY` already exists → nothing to add.
- Check both `.env.local` (local) and Vercel project settings (production). They may differ.

**Minimum required in `.env.local`** (add only what isn't already present):

```
SUPABASE_URL=https://YOUR_PROJECT.supabase.co
SUPABASE_SERVICE_ROLE_KEY=YOUR_SERVICE_ROLE_KEY
SITE_URL=https://yourdomain.com
```

Service role key is server-only. Already gitignored by Next.js default.

`SITE_URL` is the production base URL (no trailing slash). The agent reads it when returning gate links to the founder so it doesn't have to ask every time.

**Before pushing any code**, confirm the same vars exist in Vercel project settings (Settings → Environment Variables) for Production, Preview, and Development. A missing var in Vercel causes a build failure, not a runtime error, because `lib/deckin-supabase.ts` throws at module load time. Set them in Vercel first, then push.

### 5. Source files

#### `lib/deckin-supabase.ts`

Named `deckin-supabase.ts` (not `supabase.ts`) to avoid colliding with an existing `lib/supabase/` directory in projects that already use Supabase.

Falls back to `NEXT_PUBLIC_SUPABASE_URL` so projects that already have that var set don't need to add a duplicate `SUPABASE_URL`.

```ts
import { createClient } from '@supabase/supabase-js';

const url = process.env.SUPABASE_URL ?? process.env.NEXT_PUBLIC_SUPABASE_URL;
const key = process.env.SUPABASE_SERVICE_ROLE_KEY;

if (!url || !key) {
  throw new Error('SUPABASE_URL (or NEXT_PUBLIC_SUPABASE_URL) and SUPABASE_SERVICE_ROLE_KEY must be set');
}

export const supabaseAdmin = createClient(url, key, {
  auth: { persistSession: false, autoRefreshToken: false },
});
```

#### `lib/geo.ts`

```ts
import { headers } from 'next/headers';

function trim(s: string | null, max: number): string | null {
  if (!s) return null;
  return s.length > max ? s.slice(0, max) : s;
}

/**
 * Reads geo + UA from request headers. Vercel populates the x-vercel-ip-*
 * headers in production; in dev these are null.
 */
export async function getGeo() {
  const h = await headers();
  const cityRaw = h.get('x-vercel-ip-city');
  return {
    ip: trim(h.get('x-forwarded-for')?.split(',')[0]?.trim() ?? null, 64),
    country: trim(h.get('x-vercel-ip-country'), 8),
    region: trim(h.get('x-vercel-ip-country-region'), 64),
    city: cityRaw ? trim(decodeURIComponent(cityRaw), 128) : null,
    latitude: h.get('x-vercel-ip-latitude') ? Number(h.get('x-vercel-ip-latitude')) : null,
    longitude: h.get('x-vercel-ip-longitude') ? Number(h.get('x-vercel-ip-longitude')) : null,
    userAgent: trim(h.get('user-agent'), 1000),
    referrer: trim(h.get('referer'), 2048),
  };
}
```

#### `lib/decks.ts`

```ts
import { readdir } from 'fs/promises';
import path from 'path';

const SLUG_RE = /^[a-z0-9][a-z0-9-]{0,199}$/;

/**
 * Given a deck slug (e.g. "seed"), find the PDF in public/decks/[slug]/
 * and return its public URL path. Returns null if the folder doesn't
 * exist, contains no PDF, or the slug is malformed (also blocks
 * directory traversal).
 */
export async function findDeckPdf(slug: string): Promise<string | null> {
  if (!SLUG_RE.test(slug)) return null;

  const dir = path.join(process.cwd(), 'public', 'decks', slug);
  let files: string[];
  try {
    files = await readdir(dir);
  } catch {
    return null;
  }

  const pdf = files.filter((f) => f.toLowerCase().endsWith('.pdf')).sort()[0];
  if (!pdf) return null;
  return `/decks/${slug}/${encodeURIComponent(pdf)}`;
}
```

#### `lib/session-token.ts`

```ts
import crypto from 'node:crypto';

/** 32 random bytes as hex (64 chars). Sent to the client as an httpOnly cookie. */
export function generateToken(): string {
  return crypto.randomBytes(32).toString('hex');
}

/** SHA-256 hex of the token. Stored on the session row. */
export function hashToken(token: string): string {
  return crypto.createHash('sha256').update(token).digest('hex');
}

export const COOKIE_NAME = 'deckin_token';
export const COOKIE_MAX_AGE_SECONDS = 60 * 60 * 24; // 24h
```

#### `app/decks/[name]/page.tsx`

```tsx
import { notFound } from 'next/navigation';
import { findDeckPdf } from '@/lib/decks';
import GateForm from './GateForm';

export default async function Page({
  params,
}: {
  params: Promise<{ name: string }>;
}) {
  const { name } = await params;
  const pdfUrl = await findDeckPdf(name);
  if (!pdfUrl) notFound();
  return <GateForm slug={name} />;
}
```

#### `app/decks/[name]/GateForm.tsx`

```tsx
'use client';
import { useState } from 'react';
import { useRouter } from 'next/navigation';

export default function GateForm({ slug }: { slug: string }) {
  const [email, setEmail] = useState('');
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const router = useRouter();

  async function submit(e: React.FormEvent) {
    e.preventDefault();
    setLoading(true);
    setError(null);
    try {
      const res = await fetch('/api/capture', {
        method: 'POST',
        headers: { 'content-type': 'application/json' },
        body: JSON.stringify({ email, slug, referrer: document.referrer }),
      });
      if (!res.ok) throw new Error('capture failed');
      const { sessionId } = await res.json();
      router.push(`/view/${sessionId}`);
    } catch {
      setError('Something went wrong. Try again.');
      setLoading(false);
    }
  }

  return (
    <main className="min-h-screen flex items-center justify-center p-6">
      <form onSubmit={submit} className="max-w-md w-full space-y-4">
        <h1 className="text-2xl font-semibold">View the deck</h1>
        <p className="text-sm opacity-70">Drop your email to continue.</p>
        <input
          type="email"
          autoComplete="email"
          required
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          placeholder="you@company.com"
          maxLength={254}
          className="w-full border rounded px-3 py-2"
        />
        <button
          type="submit"
          disabled={loading}
          className="w-full bg-black text-white rounded py-2 disabled:opacity-50"
        >
          {loading ? 'Loading…' : 'Continue'}
        </button>
        {error && <p className="text-sm text-red-600">{error}</p>}
      </form>
    </main>
  );
}
```

#### `app/api/capture/route.ts`

```ts
import { NextRequest, NextResponse } from 'next/server';
import { supabaseAdmin } from '@/lib/deckin-supabase';
import { getGeo } from '@/lib/geo';
import { findDeckPdf } from '@/lib/decks';
import {
  generateToken,
  hashToken,
  COOKIE_NAME,
  COOKIE_MAX_AGE_SECONDS,
} from '@/lib/session-token';

export const runtime = 'nodejs';

const EMAIL_RE = /^[^@\s]+@[^@\s]+\.[^@\s]+$/;

export async function POST(req: NextRequest) {
  let body: unknown;
  try {
    body = await req.json();
  } catch {
    return NextResponse.json({ error: 'bad input' }, { status: 400 });
  }

  if (!body || typeof body !== 'object') {
    return NextResponse.json({ error: 'bad input' }, { status: 400 });
  }

  const { email: emailIn, slug: slugIn, referrer: refIn } = body as Record<string, unknown>;

  const email = typeof emailIn === 'string' ? emailIn.toLowerCase().trim() : '';
  const slug = typeof slugIn === 'string' ? slugIn.trim() : '';
  const referrer = typeof refIn === 'string' ? refIn.slice(0, 2048) : null;

  if (!email || email.length > 254 || !EMAIL_RE.test(email)) {
    return NextResponse.json({ error: 'bad input' }, { status: 400 });
  }
  if (!slug || slug.length > 200) {
    return NextResponse.json({ error: 'bad input' }, { status: 400 });
  }

  const pdfUrl = await findDeckPdf(slug);
  if (!pdfUrl) {
    return NextResponse.json({ error: 'not found' }, { status: 404 });
  }

  const geo = await getGeo();

  const { data: viewer, error: vErr } = await supabaseAdmin
    .from('deckin_viewers')
    .upsert(
      { email, last_seen_at: new Date().toISOString() },
      { onConflict: 'email' }
    )
    .select('id')
    .single();

  if (vErr || !viewer) {
    console.error('deckin: viewer upsert failed', vErr);
    return NextResponse.json({ error: 'internal' }, { status: 500 });
  }

  const token = generateToken();
  const tokenHash = hashToken(token);

  const { data: session, error: sErr } = await supabaseAdmin
    .from('deckin_sessions')
    .insert({
      viewer_id: viewer.id,
      deck_slug: slug,
      token_hash: tokenHash,
      ip: geo.ip,
      user_agent: geo.userAgent,
      country: geo.country,
      region: geo.region,
      city: geo.city,
      latitude: geo.latitude,
      longitude: geo.longitude,
      referrer: referrer || geo.referrer,
    })
    .select('id')
    .single();

  if (sErr || !session) {
    console.error('deckin: session insert failed', sErr);
    return NextResponse.json({ error: 'internal' }, { status: 500 });
  }

  const res = NextResponse.json({ sessionId: session.id });
  res.cookies.set({
    name: COOKIE_NAME,
    value: token,
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    path: '/',
    maxAge: COOKIE_MAX_AGE_SECONDS,
  });
  return res;
}
```

#### `app/api/track/route.ts`

```ts
import { NextRequest, NextResponse } from 'next/server';
import { supabaseAdmin } from '@/lib/deckin-supabase';
import { hashToken, COOKIE_NAME } from '@/lib/session-token';

export const runtime = 'nodejs';

const UUID_RE = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;
const MAX_DURATION_MS = 10 * 60 * 1000;

export async function POST(req: NextRequest) {
  const token = req.cookies.get(COOKIE_NAME)?.value;
  if (!token || token.length !== 64 || !/^[0-9a-f]{64}$/i.test(token)) {
    return NextResponse.json({ error: 'unauthorized' }, { status: 401 });
  }

  let body: unknown;
  try {
    body = await req.json();
  } catch {
    return NextResponse.json({ error: 'bad input' }, { status: 400 });
  }

  if (!body || typeof body !== 'object') {
    return NextResponse.json({ error: 'bad input' }, { status: 400 });
  }

  const { sessionId, slide, durationMs, isVisible } = body as Record<string, unknown>;

  if (
    typeof sessionId !== 'string' ||
    !UUID_RE.test(sessionId) ||
    typeof slide !== 'number' ||
    !Number.isInteger(slide) ||
    slide < 1 ||
    slide > 1000 ||
    typeof durationMs !== 'number' ||
    !Number.isFinite(durationMs) ||
    durationMs <= 0
  ) {
    return NextResponse.json({ error: 'bad input' }, { status: 400 });
  }

  const tokenHash = hashToken(token);
  const capped = Math.min(Math.floor(durationMs), MAX_DURATION_MS);

  // Verify the cookie's token hash matches this session's stored hash.
  // .single() returns 406 if no row; we treat that as 401.
  const { data: session } = await supabaseAdmin
    .from('deckin_sessions')
    .select('id')
    .eq('id', sessionId)
    .eq('token_hash', tokenHash)
    .maybeSingle();

  if (!session) {
    return NextResponse.json({ error: 'unauthorized' }, { status: 401 });
  }

  const { error: insertErr } = await supabaseAdmin
    .from('deckin_slide_events')
    .insert({
      session_id: sessionId,
      slide,
      duration_ms: capped,
      is_visible: isVisible === true,
    });

  if (insertErr) {
    console.error('deckin: track insert failed', insertErr);
    return NextResponse.json({ error: 'internal' }, { status: 500 });
  }

  // Best-effort. Don't fail the request if this update fails.
  await supabaseAdmin
    .from('deckin_sessions')
    .update({ ended_at: new Date().toISOString() })
    .eq('id', sessionId);

  return NextResponse.json({ ok: true });
}
```

#### `app/view/[sessionId]/page.tsx`

```tsx
import { redirect, notFound } from 'next/navigation';
import { cookies } from 'next/headers';
import { supabaseAdmin } from '@/lib/deckin-supabase';
import { findDeckPdf } from '@/lib/decks';
import { hashToken, COOKIE_NAME } from '@/lib/session-token';
import Viewer from './Viewer';

const UUID_RE = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;

export default async function Page({
  params,
}: {
  params: Promise<{ sessionId: string }>;
}) {
  const { sessionId } = await params;
  if (!UUID_RE.test(sessionId)) notFound();

  const { data } = await supabaseAdmin
    .from('deckin_sessions')
    .select('deck_slug, token_hash')
    .eq('id', sessionId)
    .maybeSingle();

  if (!data) notFound();

  // Cookie binding: only the original capturer (whose cookie hashes to
  // the row's token_hash) can view with tracking. Anyone else gets bounced
  // to the gate so we capture their email too.
  const cookieStore = await cookies();
  const token = cookieStore.get(COOKIE_NAME)?.value;
  if (!token || token.length !== 64 || hashToken(token) !== data.token_hash) {
    redirect(`/decks/${data.deck_slug}`);
  }

  const pdfUrl = await findDeckPdf(data.deck_slug);
  if (!pdfUrl) notFound();

  return <Viewer sessionId={sessionId} pdfUrl={pdfUrl} />;
}
```

#### `app/view/[sessionId]/Viewer.tsx`

```tsx
'use client';
import { useEffect, useRef, useState, useCallback } from 'react';
import { Document, Page, pdfjs } from 'react-pdf';
import 'react-pdf/dist/Page/TextLayer.css';
import 'react-pdf/dist/Page/AnnotationLayer.css';

pdfjs.GlobalWorkerOptions.workerSrc = '/pdf.worker.min.mjs';

type Props = { sessionId: string; pdfUrl: string };

const HEARTBEAT_MS = 10_000;
const SWIPE_THRESHOLD_PX = 50;
const TAP_THRESHOLD_PX = 10;

// Keeps the node mounted and canvas rendered, but invisible and out of layout.
const OFFSCREEN: React.CSSProperties = {
  position: 'fixed',
  top: '-9999px',
  left: '-9999px',
  opacity: 0,
  pointerEvents: 'none',
};

export default function Viewer({ sessionId, pdfUrl }: Props) {
  const [numPages, setNumPages] = useState(0);
  const [page, setPage] = useState(1);
  const [container, setContainer] = useState({ width: 0, height: 0 });
  const [pageNaturalSize, setPageNaturalSize] = useState<{ width: number; height: number } | null>(null);
  const containerRef = useRef<HTMLDivElement>(null);
  const lastTickRef = useRef<number>(Date.now());
  const touchStartXRef = useRef<number | null>(null);
  const isVisibleRef = useRef<boolean>(
    typeof document !== 'undefined' ? !document.hidden : true
  );
  const pageRef = useRef<number>(1);

  useEffect(() => {
    pageRef.current = page;
  }, [page]);

  // Measure container and respond to resize
  useEffect(() => {
    const el = containerRef.current;
    if (!el) return;
    const ro = new ResizeObserver((entries) => {
      const { width, height } = entries[0]?.contentRect ?? {};
      if (width && height) setContainer({ width, height });
    });
    ro.observe(el);
    return () => ro.disconnect();
  }, []);

  // object-fit: contain — constrain by whichever axis is the bottleneck
  let renderWidth: number | undefined;
  let renderHeight: number | undefined;
  if (container.width > 0 && container.height > 0) {
    if (pageNaturalSize) {
      const slideAspect = pageNaturalSize.width / pageNaturalSize.height;
      const containerAspect = container.width / container.height;
      if (containerAspect > slideAspect) {
        renderHeight = container.height;
      } else {
        renderWidth = container.width;
      }
    } else {
      renderHeight = container.height;
    }
  }

  const flush = useCallback(
    (opts?: { useBeacon?: boolean }) => {
      const now = Date.now();
      const delta = now - lastTickRef.current;
      lastTickRef.current = now;
      if (delta <= 0) return;

      const payload = {
        sessionId,
        slide: pageRef.current,
        durationMs: delta,
        isVisible: isVisibleRef.current,
      };

      if (opts?.useBeacon && typeof navigator !== 'undefined' && navigator.sendBeacon) {
        const blob = new Blob([JSON.stringify(payload)], { type: 'application/json' });
        navigator.sendBeacon('/api/track', blob);
      } else {
        fetch('/api/track', {
          method: 'POST',
          headers: { 'content-type': 'application/json' },
          body: JSON.stringify(payload),
          keepalive: true,
          credentials: 'same-origin',
        }).catch(() => {});
      }
    },
    [sessionId]
  );

  useEffect(() => {
    const id = setInterval(() => flush(), HEARTBEAT_MS);
    return () => clearInterval(id);
  }, [flush]);

  useEffect(() => {
    function onVis() {
      flush();
      isVisibleRef.current = !document.hidden;
    }
    document.addEventListener('visibilitychange', onVis);
    return () => document.removeEventListener('visibilitychange', onVis);
  }, [flush]);

  useEffect(() => {
    const onUnload = () => flush({ useBeacon: true });
    window.addEventListener('pagehide', onUnload);
    window.addEventListener('beforeunload', onUnload);
    return () => {
      window.removeEventListener('pagehide', onUnload);
      window.removeEventListener('beforeunload', onUnload);
    };
  }, [flush]);

  function go(next: number) {
    if (next < 1 || next > numPages) return;
    flush();
    setPage(next);
  }

  // Keyboard: arrow keys
  useEffect(() => {
    function onKey(e: KeyboardEvent) {
      if (e.key === 'ArrowRight' || e.key === 'ArrowDown') go(pageRef.current + 1);
      if (e.key === 'ArrowLeft' || e.key === 'ArrowUp') go(pageRef.current - 1);
    }
    window.addEventListener('keydown', onKey);
    return () => window.removeEventListener('keydown', onKey);
  // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [numPages]);

  // Touch: swipe left/right = next/prev; tap left/right half = prev/next
  useEffect(() => {
    const el = containerRef.current;
    if (!el) return;

    function onTouchStart(e: TouchEvent) {
      touchStartXRef.current = e.touches[0]?.clientX ?? null;
    }

    function onTouchEnd(e: TouchEvent) {
      const startX = touchStartXRef.current;
      if (startX === null) return;
      const endX = e.changedTouches[0]?.clientX ?? startX;
      const deltaX = endX - startX;
      touchStartXRef.current = null;

      if (Math.abs(deltaX) >= SWIPE_THRESHOLD_PX) {
        // Swipe: leftward = next, rightward = prev
        if (deltaX < 0) go(pageRef.current + 1);
        else go(pageRef.current - 1);
      } else if (Math.abs(deltaX) <= TAP_THRESHOLD_PX && el) {
        // Tap: left half = prev, right half = next
        const rect = el.getBoundingClientRect();
        if (endX - rect.left < rect.width / 2) go(pageRef.current - 1);
        else go(pageRef.current + 1);
      }
    }

    el.addEventListener('touchstart', onTouchStart, { passive: true });
    el.addEventListener('touchend', onTouchEnd, { passive: true });
    return () => {
      el.removeEventListener('touchstart', onTouchStart);
      el.removeEventListener('touchend', onTouchEnd);
    };
  // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [numPages]);

  const ready = container.width > 0 && container.height > 0;

  return (
    <main className="h-dvh flex flex-col bg-zinc-900 overflow-hidden">
      {/* Slide area */}
      <div
        ref={containerRef}
        className="flex-1 flex items-center justify-center overflow-hidden"
      >
        {ready && (
          <Document
            file={pdfUrl}
            onLoadSuccess={({ numPages }) => setNumPages(numPages)}
            loading={null}
            error={<span className="text-xs text-zinc-600">Failed to load.</span>}
          >
            {numPages > 0
              ? Array.from({ length: numPages }, (_, i) => i + 1).map((p) => (
                  // All pages stay mounted — active page is in flow, others offscreen.
                  // Pre-renders every canvas so navigation is instant with no flash.
                  <div key={p} style={p === page ? undefined : OFFSCREEN}>
                    <Page
                      pageNumber={p}
                      width={renderWidth}
                      height={renderHeight}
                      loading={null}
                      error={null}
                      onLoadSuccess={p === 1 ? (pg) => {
                        const [x1, y1, x2, y2] = pg.view;
                        setPageNaturalSize({ width: x2 - x1, height: y2 - y1 });
                      } : undefined}
                    />
                  </div>
                ))
              : (renderWidth ?? renderHeight) && (
                  <Page
                    pageNumber={1}
                    width={renderWidth}
                    height={renderHeight}
                    loading={null}
                    onLoadSuccess={(pg) => {
                      const [x1, y1, x2, y2] = pg.view;
                      setPageNaturalSize({ width: x2 - x1, height: y2 - y1 });
                    }}
                  />
                )}
          </Document>
        )}
      </div>

      {/* Navigation */}
      <div className="flex items-center justify-center gap-4 py-2.5">
        <button
          onClick={() => go(page - 1)}
          disabled={page <= 1}
          className="text-sm text-zinc-400 hover:text-zinc-100 disabled:opacity-20 transition-colors px-1"
        >
          ‹
        </button>
        <span className="text-xs tabular-nums text-zinc-500">
          {page} / {numPages || '…'}
        </span>
        <button
          onClick={() => go(page + 1)}
          disabled={!numPages || page >= numPages}
          className="text-sm text-zinc-400 hover:text-zinc-100 disabled:opacity-20 transition-colors px-1"
        >
          ›
        </button>
      </div>

      {/* DeckIn footer. Remove this <footer> block to take it down. */}
      <footer className="pb-2.5 text-center">
        <p className="text-[10px] tracking-wide text-zinc-700 m-0">
          Capital efficient founders use{' '}
          <a
            href="https://deckin.bubs.co"
            target="_blank"
            rel="noopener noreferrer"
            className="text-zinc-600 underline underline-offset-2"
          >
            DeckIn
          </a>{' '}
          to share decks
        </p>
      </footer>
    </main>
  );
}
```

### 6. Adding a deck (agent flow)

When the founder shares a PDF and asks to "set up the deck" or similar:

1. **Confirm the slug.** Check `public/decks/` for existing folders. Pick a short kebab-case name based on context the founder gave (e.g. `seed`, `q4-2026`, `acme-pitch`). If the slug isn't obvious from context, ask the founder before proceeding.

2. **Check for collision.** If `public/decks/[slug]/` already exists, do NOT overwrite. Ask the founder whether to replace the existing deck, use a different slug, or version it (e.g. `seed-v2`). Wait for confirmation before continuing.

3. **Copy the PDF**:
   ```bash
   mkdir -p public/decks/[slug]
   cp [source-pdf-path] public/decks/[slug]/deck.pdf
   ```
   The filename inside the folder doesn't matter; `findDeckPdf` picks the first `.pdf` alphabetically. Use `deck.pdf` for consistency.

4. **Commit and push**:
   ```bash
   git add public/decks/[slug]
   git commit -m "Add [slug] deck"
   git push
   ```

5. **Wait for the deploy.** If the Vercel CLI is installed, run `vercel inspect --wait` on the latest deployment. Otherwise poll `${SITE_URL}/decks/[slug]` until it returns 200 (give up after 3 minutes and tell the founder to check the Vercel dashboard).

6. **Return the link**: `${SITE_URL}/decks/[slug]`

### 7. Deploy

1. **Set env vars in Vercel first** (Settings → Environment Variables, all three environments). Check what's already there before adding — see step 4. At minimum you need `SUPABASE_SERVICE_ROLE_KEY` and either `SUPABASE_URL` or `NEXT_PUBLIC_SUPABASE_URL`, plus `SITE_URL`.
2. Push to GitHub (or import the repo into Vercel if this is a fresh project).
3. Deploy. Geo headers populate automatically on Vercel.

Do not push code before setting Vercel env vars. The Supabase client throws at module load time, so a missing var is a build failure.

---

## Querying DeckIn

Founders ask their agent: "did anyone open my deck today," "which slide did Acme spend the longest on," "who's a repeat viewer." The agent runs queries against the Supabase database.

### Branch: Supabase MCP available

If the founder has the Supabase MCP server connected to their agent (Claude/Cursor), the agent uses MCP tools directly. No credential handling. Schema reference and query patterns below apply unchanged; the agent just executes through MCP instead of HTTP.

### Branch: no MCP

The agent uses the Supabase REST API or `psql`. Both need the project URL and either the service role key (REST) or the database connection string (psql). Get the connection string from `Project Settings → Database → Connection string → URI`.

**psql (most flexible)**:

```bash
psql "$SUPABASE_DB_URL" -c "select count(*) from deckin_sessions;"
```

**REST + service role key (no install needed)**:

```bash
curl -s "$SUPABASE_URL/rest/v1/deckin_sessions?select=*&limit=10" \
  -H "apikey: $SUPABASE_SERVICE_ROLE_KEY" \
  -H "Authorization: Bearer $SUPABASE_SERVICE_ROLE_KEY"
```

REST is row-filter only, no joins or aggregations. For the queries below, use psql or wrap each canned query as a Postgres function and call it via REST RPC.

### Schema reference (for the agent)

- **`deckin_viewers`**: one row per unique email. `id` (uuid), `email`, `first_seen_at`, `last_seen_at`.
- **`deckin_sessions`**: one row per gate submission. `id` (uuid, also the URL session id), `viewer_id` FK, `deck_slug`, `started_at`, `ended_at` (touched on every track), `ip`, `user_agent`, `country`, `region`, `city`, `latitude`, `longitude`, `referrer`. Ignore `token_hash`; it's auth-internal.
- **`deckin_slide_events`**: append-only log of time chunks. One row per heartbeat, slide change, or visibility change. `session_id` FK, `slide` (1-indexed), `duration_ms` (capped at 600000 = 10min per chunk), `is_visible` (false when tab was backgrounded). Sum `duration_ms` where `is_visible = true` to get active time per slide. Total dwell on a slide can exceed first-visit time because viewers revisit slides.

Time fields are `timestamptz`. Convert to the founder's timezone in queries (e.g. `started_at at time zone 'America/Los_Angeles'`).

### Canned queries

Replace the timezone literal in each query with the founder's timezone.

**Recent sessions (last 7 days)**:

```sql
select
  v.email,
  s.deck_slug,
  s.started_at at time zone 'America/Los_Angeles' as started_local,
  s.country,
  s.city,
  coalesce(sum(case when e.is_visible then e.duration_ms else 0 end), 0) / 1000 as active_seconds,
  count(distinct e.slide) as slides_viewed
from deckin_sessions s
join deckin_viewers v on v.id = s.viewer_id
left join deckin_slide_events e on e.session_id = s.id
where s.started_at > now() - interval '7 days'
group by v.email, s.id
order by s.started_at desc;
```

**Today's openers (founder's local timezone)**:

```sql
select v.email, s.deck_slug, s.started_at at time zone 'America/Los_Angeles' as opened, s.city
from deckin_sessions s
join deckin_viewers v on v.id = s.viewer_id
where (s.started_at at time zone 'America/Los_Angeles')::date = (now() at time zone 'America/Los_Angeles')::date
order by s.started_at desc;
```

**Per-slide attention for a single session**:

```sql
select
  slide,
  sum(case when is_visible then duration_ms else 0 end) / 1000 as active_seconds,
  count(*) as event_count
from deckin_slide_events
where session_id = 'PASTE-SESSION-UUID'
group by slide
order by slide;
```

**Most engaged viewers (sum of active time across all sessions)**:

```sql
select
  v.email,
  count(distinct s.id) as session_count,
  sum(case when e.is_visible then e.duration_ms else 0 end) / 1000 as total_active_seconds,
  max(s.started_at) as last_seen
from deckin_viewers v
join deckin_sessions s on s.viewer_id = v.id
left join deckin_slide_events e on e.session_id = s.id
group by v.email
order by total_active_seconds desc nulls last
limit 20;
```

**Repeat viewers (more than one session)**:

```sql
select v.email, count(s.id) as visit_count, max(s.started_at) as last_visit
from deckin_viewers v
join deckin_sessions s on s.viewer_id = v.id
group by v.email
having count(s.id) > 1
order by visit_count desc;
```

**Slide attention heatmap across all viewers for a deck**:

```sql
select
  e.slide,
  count(distinct s.id) as sessions_reaching_slide,
  sum(case when e.is_visible then e.duration_ms else 0 end) / 1000 as total_active_seconds,
  avg(case when e.is_visible then e.duration_ms else null end) / 1000.0 as avg_active_seconds_per_event
from deckin_slide_events e
join deckin_sessions s on s.id = e.session_id
where s.deck_slug = 'seed'
group by e.slide
order by e.slide;
```

**Drop-off analysis (max slide reached per session)**:

```sql
select
  max_slide,
  count(*) as session_count
from (
  select session_id, max(slide) as max_slide
  from deckin_slide_events
  group by session_id
) sub
group by max_slide
order by max_slide;
```

For a Claude scheduled task that emails a daily summary, run "Today's openers" + a count from "Recent sessions" + the founder's timezone, format as text, send.

---

## Notes

**Removing the footer**: delete the `<footer>` block in `app/view/[sessionId]/Viewer.tsx` (marked with the `{/* DeckIn footer... */}` comment). The default footer reads "Capital efficient founders use DeckIn to share decks" with DeckIn linked to https://deckin.bubs.co. Keeping it on is the polite default for an OSS project; it costs nothing and helps other founders find DeckIn.

**Cookie binding**: each session creates an httpOnly cookie containing a 32-byte random token. The session row stores the token's SHA-256 hash. View page and track endpoint both verify the cookie hashes to the stored value. If an investor forwards the URL, the recipient has no cookie, gets bounced to the gate, captures their own email. This is intentional; you want both emails.

**Direct PDF access**: `/decks/seed/Seed.pdf` is publicly accessible by design. The gate creates friction for the casual viewer; it doesn't lock down the file. Anyone determined enough to view-source the PDF URL was never going to be tracked anyway. To gate the file itself, move PDFs to Supabase Storage in a private bucket and issue 5-minute signed URLs from the view page.

**Geo headers**: Vercel-only. Other platforms expose IP geo through different headers (Cloudflare uses `cf-ipcountry`); swap `lib/geo.ts` accordingly.

**`pdfjs` worker**: self-hosted from `/public/pdf.worker.min.mjs`. Re-copy after every `npm update` of `pdfjs-dist` so the worker matches the runtime. Mismatch causes silent rendering failures.

**Adding new decks**: `mkdir public/decks/[slug]` and drop a PDF in. Push to deploy. No code changes.

---

## Future hardening (skip until you actually need it)

- **Rate limiting** on `/api/capture` via Vercel edge middleware. Stops a script filling `deckin_viewers` with junk.
- **Session expiry**: reject `track` events if `started_at > 6 hours ago`. Stops drift from forgotten tabs.
- **Signed URLs for the PDF** (see Direct PDF access note above).
- **Per-deck access lists**: add `allowed_emails text[]` on a `deckin_decks` table; check at capture time. Useful for restricting a deck to known recipients.
- **Notifications**: Supabase database webhook on `deckin_sessions` insert that pings Slack or email when someone opens.
- **CSP headers** in `next.config.js`: defense in depth against future XSS surface.
