# Smart Bookmark App

A simple bookmark manager built with Next.js (App Router), Supabase, and Tailwind CSS. Users can sign in with Google, save bookmarks, and see updates in real-time across tabs.

## Features

- Google OAuth sign-in (no email/password)
- Add bookmarks (URL + title)
- Delete bookmarks
- Bookmarks are private per user (Row Level Security)
- Real-time updates across tabs via Supabase Realtime

## Tech Stack

- **Next.js 15** (App Router)
- **Supabase** (Auth, PostgreSQL Database, Realtime)
- **Tailwind CSS**

## Setup

### 1. Clone and install

```bash
git clone <your-repo-url>
cd bookmark-app
npm install
```

### 2. Create a Supabase project

1. Go to [supabase.com](https://supabase.com) and create a new project.
2. Copy your **Project URL** and **anon public key** from **Settings > API**.

### 3. Configure Google OAuth

1. Go to [Google Cloud Console](https://console.cloud.google.com/).
2. Create a new project (or use an existing one).
3. Go to **APIs & Services > Credentials > Create Credentials > OAuth 2.0 Client ID**.
4. Set the application type to **Web application**.
5. Add the following authorized redirect URI:
   ```
   https://<your-supabase-project-ref>.supabase.co/auth/v1/callback
   ```
6. Copy the **Client ID** and **Client Secret**.
7. In your Supabase dashboard, go to **Authentication > Providers > Google**.
8. Enable Google and paste the Client ID and Client Secret.

### 4. Create the database table

Run this SQL in the Supabase **SQL Editor**:

```sql
CREATE TABLE bookmarks (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
  title TEXT NOT NULL,
  url TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now() NOT NULL
);

-- Enable Row Level Security
ALTER TABLE bookmarks ENABLE ROW LEVEL SECURITY;

-- Users can only see their own bookmarks
CREATE POLICY "Users can view own bookmarks"
  ON bookmarks FOR SELECT
  USING (auth.uid() = user_id);

-- Users can only insert their own bookmarks
CREATE POLICY "Users can insert own bookmarks"
  ON bookmarks FOR INSERT
  WITH CHECK (auth.uid() = user_id);

-- Users can only delete their own bookmarks
CREATE POLICY "Users can delete own bookmarks"
  ON bookmarks FOR DELETE
  USING (auth.uid() = user_id);
```

### 5. Enable Realtime

In your Supabase dashboard, go to **Database > Replication** and add the `bookmarks` table to the publication. This enables real-time updates.

### 6. Set environment variables

Copy the example file and fill in your values:

```bash
cp .env.local.example .env.local
```

Edit `.env.local`:

```
NEXT_PUBLIC_SUPABASE_URL=https://<your-project-ref>.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=<your-anon-key>
```

### 7. Run locally

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

## Deploy to Vercel

1. Push your code to a GitHub repository.
2. Go to [vercel.com](https://vercel.com) and import the repository.
3. Add the environment variables (`NEXT_PUBLIC_SUPABASE_URL` and `NEXT_PUBLIC_SUPABASE_ANON_KEY`) in the Vercel project settings.
4. Deploy.
5. **Important**: Add your Vercel deployment URL to the allowed redirect URLs in Supabase:
   - Go to **Authentication > URL Configuration**.
   - Add `https://your-app.vercel.app/auth/callback` to the **Redirect URLs**.

## Problems Encountered and Solutions

- **Supabase SSR cookie handling**: Used `@supabase/ssr` package with proper cookie getter/setter implementations for both browser and server contexts. The server client wraps Next.js `cookies()` API and silently catches errors when `setAll` is called from Server Components (since cookies can only be set in Server Actions, Route Handlers, or Middleware).

- **Middleware for session refresh**: Implemented Next.js middleware that refreshes the Supabase auth session on every request, ensuring users don't get unexpectedly logged out. The middleware also protects the `/dashboard` route by redirecting unauthenticated users to the home page.

- **Real-time filtering by user**: Used Supabase Realtime's filter parameter (`filter: user_id=eq.${userId}`) on INSERT events to only receive the current user's bookmark changes. For DELETE events, we listen to all deletes (since the filter can't access the deleted row's data) and remove matching IDs from local state.

- **Duplicate prevention**: Added a check in the real-time INSERT handler to prevent duplicate bookmarks from appearing when both the optimistic local insert and the real-time event fire.
