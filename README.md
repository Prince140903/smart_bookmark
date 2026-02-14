# Smart Bookmark App

A real-time bookmark manager where users sign in with Google, save bookmarks, and see changes sync instantly across tabs. Built with Next.js (App Router), Supabase, and Tailwind CSS.

**Live URL**: https://smart-bookmark-dun.vercel.app/

## Tech Stack

- **Next.js 16** — App Router, Server Components, Route Handlers
- **Supabase** — Google OAuth, PostgreSQL with Row Level Security, Realtime subscriptions
- **Tailwind CSS** — utility-first styling
- **TypeScript** — end-to-end type safety

## Features

- Google OAuth authentication (no email/password)
- Add and delete bookmarks (URL + title)
- Bookmarks are private to each user via RLS policies
- Real-time sync — add a bookmark in one tab, it appears in another instantly
- Optimistic UI updates for snappy interactions
- Auto-prepend `https://` for URLs entered without a protocol
- Sticky header with glassmorphism blur, relative timestamps, hover-reveal actions

## Architecture Decisions

- **Server Components for initial data**: The dashboard page is a Server Component that fetches bookmarks on the server and passes them down. This means the page loads with data already present — no loading spinner on first render.

- **Single client-side state owner**: Instead of splitting the form and list into separate client components with no shared state, I consolidated them into one `DashboardClient` component. This was a deliberate choice — it lets the form immediately push a new bookmark into the list state after a successful insert, without waiting for the Realtime event. The Realtime subscription still runs alongside for cross-tab sync.

- **Two Supabase clients**: One for the browser (`createBrowserClient` from `@supabase/ssr`) and one for the server (`createServerClient` that wraps the Next.js `cookies()` API). This separation is necessary because the server client needs to read/write HTTP-only cookies for auth, while the browser client manages its own session from cookies automatically.

- **Middleware for session refresh**: A root-level middleware runs on every request to refresh the Supabase auth token. Without this, users would silently lose their session after the JWT expires, leading to confusing "not authenticated" errors on actions that were working moments ago.

## Problems I Ran Into

**1. Form silently refusing to submit**

My URL input initially had `type="url"`, which I assumed would just be a hint for mobile keyboards. Turns out the browser's built-in HTML5 validation blocks form submission entirely if the value doesn't match the URL spec (needs a protocol). Users typing `example.com` without `https://` would click "Add Bookmark" and nothing would happen — no error, no feedback. Took me a while to figure out because `handleSubmit` wasn't even firing. Fixed by switching to `type="text"` and adding manual `https://` prepend logic.

**2. Bookmarks not appearing after adding**

After fixing the form submission, bookmarks were being inserted into the database but not showing up in the list. The issue was architectural — my `BookmarkForm` and `BookmarkList` were separate client components with no shared state. The form would insert successfully and clear itself, but the list was only listening to Supabase Realtime events for updates. If Realtime wasn't configured properly (I hadn't enabled replication on the `bookmarks` table initially), the list just sat there empty. The fix was twofold: enable replication in Supabase, and also restructure so the form directly updates local state on successful insert via `.select().single()` chained on the insert call.

**3. Supabase SSR cookie errors in Server Components**

The server-side Supabase client needs to set cookies (for token refresh), but Next.js Server Components are read-only — you can't set cookies from them. This would throw errors during `getUser()` calls. The solution from the Supabase SSR docs is to wrap `setAll` in a try-catch and silently swallow the error. The actual cookie refresh happens in the middleware instead, so by the time a Server Component reads the session, it's already been refreshed.

**4. Realtime DELETE events not filtering by user**

For INSERT events, Supabase Realtime supports filtering by column (`filter: user_id=eq.${userId}`), so each user only receives their own bookmark inserts. But DELETE events send the `old` record, and Supabase can't filter on `old` row data. So DELETE events are unfiltered — every user's deletes come through. I handle this by just running `prev.filter(b => b.id !== deletedId)` on local state, which is harmless if the deleted ID doesn't belong to the current user (it simply won't match anything).

**5. Duplicate bookmarks from race condition**

After fixing the state issue, I had duplicate bookmarks appearing. The insert would immediately add the bookmark to local state, and then the Realtime INSERT event would fire a moment later and try to add the same bookmark again. Fixed with a simple `prev.some(b => b.id === newBookmark.id)` guard in both the insert callback and the Realtime handler.

**6. Google OAuth redirect URL mismatch on Vercel**

Locally everything worked, but after deploying to Vercel, Google sign-in would fail with a redirect mismatch error. I had to add the production callback URL (`https://my-app.vercel.app/auth/callback`) to two places: the Google Cloud Console OAuth credentials AND Supabase's Authentication > URL Configuration > Redirect URLs. Missing either one causes the flow to break.

**7. Middleware deprecation warning in Next.js 16**

Next.js 16 shows a warning that the `middleware.ts` file convention is deprecated in favor of `proxy`. The middleware still works for now, so I kept it since Supabase's SSR library is built around it. Something to migrate when `@supabase/ssr` adds proxy support.
