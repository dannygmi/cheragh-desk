# Cheragh Desk

A single-page operations desk for [cheragh.co.uk](https://cheragh.co.uk): orders and
what each one really made, running costs, per-variant margins, campaign spend.

One HTML file. No framework, no build step, no bundler, no dependencies. It talks to
Supabase PostgREST directly with the signed-in user's JWT.

## Why this repo is public

It holds the page and nothing else. The two values in it, the project URL and the
Supabase **publishable** key, are the pair every static front end must ship to the
browser. They grant nothing on their own.

Access is decided by row level security in the database, not by this file. Verified
2026-09-10: with only that key and no session, every table and every view returns
empty or refuses. Views are `security_invoker`, so RLS follows through them rather
than being bypassed by the view owner's privileges.

No service key, no password and no customer data lives here.

## Running it locally

    python3 -m http.server 8800
