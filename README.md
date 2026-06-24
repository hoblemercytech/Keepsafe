# 🔐 KeepSafe — Your Personal Digital Vault

> A secure, beautiful, and fully-featured personal digital vault web app. Store photos, documents, videos, notes, passwords, links, mails, and audio — all in one private place.

![KeepSafe Dashboard](./screenshots/dashboard.png)

---

## ✨ Features

### 🗂️ Vault Categories
- 📷 **Photos** — Store and preview images with download support
- 📄 **Documents** — Upload and download files
- 🎬 **Videos** — Store and stream videos with download
- 📝 **Notes** — Write notes with one-tap copy
- 🔑 **Passwords** — Securely store passwords with app name, username, and masked display
- 🔗 **Links** — Save URLs with one-tap open and copy
- 📧 **Mails** — Save important email content
- 🎵 **Audio** — Store and play audio files

### 🔒 Security
- **SHA-256 hashed Privacy PIN** — stored in Supabase, works across all devices
- **PIN lockout system** — 4 wrong attempts → 30 min lock, repeat → 1 hour lock with live countdown
- **PIN reset** requires verifying current PIN first
- **Supabase Auth** — email/password authentication with secure sessions

### 🗑️ Trash & Recovery
- Soft delete — items move to Trash, never lost immediately
- **30-day auto-expiry** with visual countdown per item
- Restore or permanently delete from a dedicated Trash modal
- Empty Trash button for bulk permanent deletion

### 🔔 Notifications
- Every vault action logs a notification (add, delete, restore, PIN change, profile update, login)
- Bell icon with live unread badge
- Slide-down notification panel with mark-as-read
- **Instant email notifications** via Resend + Supabase Edge Functions
- Branded HTML email template for every event

### 🔍 Search
- Global search across all categories simultaneously
- Results grouped by category with counts
- 350ms debounce for smooth typing experience

### 👤 Profile
- Update display name
- Change password from profile settings
- Update Privacy PIN with current PIN verification
- Light / Dark mode toggle — persists across sessions

### 📱 PWA — Installable App
- Works on iPhone (Add to Home Screen) and Android (Install prompt)
- Offline fallback page with auto-reconnect
- Safe area support for notch / Dynamic Island / home indicator
- Service Worker with smart caching strategy
- App shortcuts for quick access

### 🎨 Design
- Premium dark/light theme with violet + cyan palette
- Smooth modal animations with spring physics
- Ambient radial mesh background
- Glass-morphism cards with glow effects
- Fully responsive — mobile, tablet, desktop

---

## 🛠️ Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Vanilla HTML, CSS, JavaScript |
| Backend / Database | [Supabase](https://supabase.com) (PostgreSQL) |
| Authentication | Supabase Auth |
| File Storage | Supabase Storage (`keep_vault` bucket) |
| Email | [Resend](https://resend.com) via Supabase Edge Functions |
| Fonts | Cabinet Grotesk + Syncopate (Google Fonts) |
| Icons | Font Awesome 6 |
| PWA | Service Worker + Web App Manifest |

---

## 🗄️ Database Schema

### `keep_profile`
| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID | References `auth.users` |
| `full_name` | TEXT | User's display name |
| `email` | TEXT | User's email |
| `privacy_pin_hash` | TEXT | SHA-256 hash of 4-digit PIN |
| `updated_at` | TIMESTAMPTZ | Last update timestamp |

### `keep_items`
| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID | Primary key |
| `user_id` | UUID | References `auth.users` |
| `title` | TEXT | Item title |
| `content` | TEXT | Note content / URL / JSON (passwords) |
| `category` | TEXT | `photos\|documents\|videos\|notes\|passwords\|links\|mails\|audio` |
| `file_url` | TEXT | Public URL from Supabase Storage |
| `file_name` | TEXT | Original filename |
| `file_size` | BIGINT | File size in bytes |
| `is_favourite` | BOOLEAN | Starred item |
| `is_deleted` | BOOLEAN | Soft delete flag |
| `deleted_at` | TIMESTAMPTZ | When item was trashed |
| `created_at` | TIMESTAMPTZ | Creation timestamp |
| `updated_at` | TIMESTAMPTZ | Last edit timestamp |

### `keep_notifications`
| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID | Primary key |
| `user_id` | UUID | References `auth.users` |
| `type` | TEXT | `item_added\|item_deleted\|item_restored\|pin_changed\|profile_updated\|login` |
| `title` | TEXT | Notification title |
| `message` | TEXT | Notification body |
| `is_read` | BOOLEAN | Read state |
| `metadata` | JSONB | Extra data (device, time, category, etc.) |
| `created_at` | TIMESTAMPTZ | Timestamp |

---

## 🚀 Setup Guide

### 1. Clone the repo
```bash
git clone https://github.com/yourusername/keepsafe.git
cd keepsafe
```

### 2. Create a Supabase project
1. Go to [supabase.com](https://supabase.com) and create a new project
2. Copy your **Project URL** and **Anon Key** from Settings → API

### 3. Run the database migrations
In your Supabase SQL Editor, run these in order:

```sql
-- Profiles table
CREATE TABLE public.keep_profile (
  id               UUID REFERENCES auth.users(id) ON DELETE CASCADE PRIMARY KEY,
  full_name        TEXT,
  email            TEXT,
  privacy_pin_hash TEXT,
  updated_at       TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE public.keep_profile ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can read own profile"   ON public.keep_profile FOR SELECT TO authenticated USING (auth.uid() = id);
CREATE POLICY "Users can update own profile" ON public.keep_profile FOR UPDATE TO authenticated USING (auth.uid() = id);

-- Auto-create profile on signup
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO public.keep_profile (id, full_name, email, privacy_pin_hash, updated_at)
  VALUES (NEW.id, COALESCE(NEW.raw_user_meta_data->>'full_name', ''), NEW.email, NULL, NOW())
  ON CONFLICT (id) DO NOTHING;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
AFTER INSERT ON auth.users
FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();

-- Items table
CREATE TABLE public.keep_items (
  id           UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id      UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
  title        TEXT NOT NULL,
  content      TEXT,
  category     TEXT NOT NULL,
  file_url     TEXT,
  file_name    TEXT,
  file_size    BIGINT DEFAULT 0,
  is_favourite BOOLEAN DEFAULT FALSE,
  is_deleted   BOOLEAN DEFAULT FALSE,
  deleted_at   TIMESTAMPTZ,
  created_at   TIMESTAMPTZ DEFAULT NOW(),
  updated_at   TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE public.keep_items ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can manage own items" ON public.keep_items FOR ALL TO authenticated USING (auth.uid() = user_id) WITH CHECK (auth.uid() = user_id);

-- Notifications table
CREATE TABLE public.keep_notifications (
  id         UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id    UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
  type       TEXT NOT NULL,
  title      TEXT NOT NULL,
  message    TEXT NOT NULL,
  is_read    BOOLEAN DEFAULT FALSE,
  metadata   JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE public.keep_notifications ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can manage own notifications" ON public.keep_notifications FOR ALL TO authenticated USING (auth.uid() = user_id) WITH CHECK (auth.uid() = user_id);
```

### 4. Create the storage bucket
1. Go to **Storage → New Bucket**
2. Name: `keep_vault`
3. Enable **Public bucket**
4. Run in SQL Editor:

```sql
CREATE POLICY "Users can upload own files"   ON storage.objects FOR INSERT TO authenticated WITH CHECK (bucket_id = 'keep_vault' AND (storage.foldername(name))[1] = auth.uid()::text);
CREATE POLICY "Users can read own files"     ON storage.objects FOR SELECT TO authenticated USING (bucket_id = 'keep_vault' AND (storage.foldername(name))[1] = auth.uid()::text);
CREATE POLICY "Users can delete own files"   ON storage.objects FOR DELETE TO authenticated USING (bucket_id = 'keep_vault' AND (storage.foldername(name))[1] = auth.uid()::text);
CREATE POLICY "Public can view vault files"  ON storage.objects FOR SELECT TO public USING (bucket_id = 'keep_vault');
```

### 5. Deploy the Edge Function
1. Go to **Edge Functions → New Function** → name it `send-notification-email`
2. Paste the contents of `edge-functions/send-notification-email.ts`
3. Go to **Secrets** and add:
   ```
   RESEND_API_KEY = your_resend_api_key
   ```

### 6. Configure your credentials
In `dashboard.html` and `auth.html`, update:
```js
const SUPABASE_URL      = 'your_supabase_project_url';
const SUPABASE_ANON_KEY = 'your_supabase_anon_key';
```

### 7. Generate PWA icons
1. Open `icon-generator.html` in your browser
2. Upload your `logo.png`
3. Click **Generate** then **Download All**
4. Place the `icons/` folder in your project root

### 8. Deploy
Deploy to any static host — **Vercel**, **Netlify**, or **GitHub Pages**:

```bash
# Vercel
npx vercel

# Netlify
npx netlify deploy --prod

# GitHub Pages — just push to main, enable Pages in repo settings
git add .
git commit -m "Initial KeepSafe deployment"
git push origin main
```

---

## 📁 Project Structure

```
keepsafe/
├── auth.html                  # Login & signup page
├── dashboard.html             # Main app (all vault features)
├── reset-password.html        # Password reset page
├── offline.html               # PWA offline fallback
├── manifest.json              # PWA manifest
├── sw.js                      # Service Worker
├── logo.png                   # App logo
├── icons/                     # PWA icons (generated)
│   ├── icon-72.png
│   ├── icon-96.png
│   ├── icon-128.png
│   ├── icon-144.png
│   ├── icon-152.png
│   ├── icon-192.png
│   ├── icon-384.png
│   └── icon-512.png
├── screenshots/               # App screenshots (optional)
│   ├── dashboard.png
│   └── mobile.png
└── edge-functions/
    └── send-notification-email.ts
```

---

## 🔐 Security Notes

- All user data is protected by Supabase **Row Level Security (RLS)** — users can only access their own data
- Privacy PINs are **SHA-256 hashed** before storage — never stored in plain text
- File uploads are scoped to user-specific folders in Supabase Storage
- Sessions are managed by Supabase Auth — JWTs expire and rotate automatically
- PIN lockout after 4 failed attempts prevents brute-force access

---

## 🌍 Environment

This project requires no build step — it runs as plain HTML/CSS/JS. All dependencies are loaded via CDN:

- `@supabase/supabase-js@2` — Supabase client
- `Font Awesome 6.5.0` — icons
- `Cabinet Grotesk + Syncopate` — typography

---

## 📸 Screenshots

| Dashboard | Categories | Item Viewer |
|-----------|-----------|-------------|
| ![Dashboard](./screenshots/dashboard.png) | ![Categories](./screenshots/categories.png) | ![Viewer](./screenshots/viewer.png) |

---

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/amazing-feature`
3. Commit your changes: `git commit -m 'Add amazing feature'`
4. Push to the branch: `git push origin feature/amazing-feature`
5. Open a Pull Request

---

## 📄 License

MIT License — see [LICENSE](./LICENSE) for details.

---

## 👤 Author

**Hoblemercy Tech**
- GitHub: [@hoblemercytech](https://github.com/hoblemercytech)

---

<div align="center">
  <p>Built with ❤️ using Supabase, Resend, and vanilla JavaScript</p>
  <p>🔐 <strong>KeepSafe</strong> — Everything safe. Always yours. Always here.</p>
</div>
