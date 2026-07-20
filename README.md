# IGIMS ID Card System

A static site (deployable anywhere — GitHub Pages, Vercel, Cloudflare Pages,
etc.) backed by **Supabase** (free tier) for real database storage, real
image file storage, and real admin authentication.

## Files

| File                  | Purpose                                                    |
|-----------------------|-------------------------------------------------------------|
| `index.html`          | Landing / home page                                         |
| `login.html`          | Admin login (real Supabase Auth — no more hardcoded password)|
| `admin.html`          | The ID card generator + Employee Submissions panel (admin only) |
| `user-form.html`      | Public employee details + photo/signature submission form   |
| `thank-you.html`      | Confirmation page after a user submits the form              |
| `supabase-config.js`  | Your Supabase project URL + anon key (fill this in once)    |

No build step, no framework — plain HTML/CSS/JS, plus the `supabase-js`
library loaded from a CDN.

---

## One-time setup: create your Supabase project

1. Go to https://supabase.com and sign up (free — no credit card required).
2. Create a new project (pick any name/region/password for the database).
3. Once it's ready, go to **Project Settings → API**. You'll need two values
   from that page:
   - **Project URL**
   - **anon / public key** (NOT the `service_role` key — that one must never
     be used in browser code)
4. Open `supabase-config.js` in this folder and paste them in:
   ```js
   const SUPABASE_URL = 'https://your-project-ref.supabase.co';
   const SUPABASE_ANON_KEY = 'your-anon-public-key';
   ```

### Create the database table

Go to the **SQL Editor** in your Supabase dashboard, paste this in, and run it:

```sql
create table public.submissions (
  id uuid primary key default gen_random_uuid(),
  created_at timestamptz not null default now(),
  full_name text not null,
  designation text not null,
  department text not null,
  blood_group text not null,
  mobile_no text not null,
  address text not null,
  photo_url text,
  signature_url text
);

alter table public.submissions enable row level security;

-- Anyone can submit a new entry (the public employee form needs this)
create policy "Allow public insert"
  on public.submissions for insert
  to anon
  with check (true);

-- Only logged-in admins can view or delete submissions
create policy "Allow authenticated read"
  on public.submissions for select
  to authenticated
  using (true);

create policy "Allow authenticated delete"
  on public.submissions for delete
  to authenticated
  using (true);
```

### Create the storage bucket (for photos & signatures)

1. In the dashboard, go to **Storage** → **Create a new bucket**.
2. Name it exactly: `employee-uploads`
3. Toggle **Public bucket** → ON (this lets the admin panel display the
   images directly via URL).
4. Then go back to **SQL Editor** and run this to allow the public form to
   upload files:
   ```sql
   create policy "Public can upload to employee-uploads"
     on storage.objects for insert
     to anon
     with check (bucket_id = 'employee-uploads');

   create policy "Public can view employee-uploads"
     on storage.objects for select
     to public
     using (bucket_id = 'employee-uploads');
   ```

### Create your admin account

1. In the dashboard, go to **Authentication → Users → Add User**.
2. Enter your email and a password.
3. Check **"Auto Confirm User"** (so you don't need to click an email
   confirmation link) → Create.
4. That's it — this is what you'll log in with at `login.html`.

You can add more admin users the same way later if needed.

---

## Deploy the site

This is a completely static site — deploy the whole folder to any of these
(all free):

| Host | How |
|---|---|
| **GitHub Pages** | Push to a repo → Settings → Pages → deploy from branch |
| **Vercel** | `vercel` CLI, or drag-and-drop at vercel.com/new |
| **Cloudflare Pages** | Connect a repo, or `wrangler pages deploy` |
| **Render (Static Site)** | Connect a repo, publish directory `.` |
| **Surge.sh** | `npm install -g surge` then run `surge` in this folder |
| Your own server | Upload via FTP/File Manager |

No environment variables or server config needed — `supabase-config.js`
carries everything the site needs.

---

## How it all works together

1. **Employee** opens `user-form.html`, fills in their details, and uploads
   a photo + signature. The browser compresses both images, uploads them to
   the `employee-uploads` Supabase Storage bucket, and inserts a row into the
   `submissions` table with their details + image URLs.
2. **Admin** logs in at `login.html` using their real Supabase account
   (email + password) — this is now genuine authentication, not just a
   front-end check.
3. Once inside `admin.html`, the **"Pending Employee Submissions"** panel at
   the top lists every submission with a thumbnail of their photo/signature.
4. Clicking **"Load into Card"** on a submission instantly fills the Name,
   Designation, Department, Blood Group, Photo, and Signature fields in the
   ID card generator below — ready to review, tweak, and download as PNG.
5. **"Delete"** removes a submission once it's been processed.

## Notes & limits

- Supabase's free tier includes 500MB database storage, 1GB file storage,
  and 50,000 monthly active users for Auth — more than enough for this use
  case. Check https://supabase.com/pricing for current limits.
- `admin.html` also loads `html2canvas` (for PNG download) and `qrcodejs`
  (for the back-side QR code) from a CDN — these need internet access in the
  visitor's browser, same as before.
- Photos/signatures are stored as real files in Supabase Storage (not
  base64 text), so there's no size bloat in the database and images load
  like normal `<img>` sources.
- The `employee-uploads` bucket is public, meaning anyone with a direct
  image URL could view that specific photo/signature (the URLs aren't
  guessable, but they also aren't access-controlled). This is a common,
  acceptable tradeoff for this kind of use case — let me know if you'd
  rather lock that down further with signed URLs.
