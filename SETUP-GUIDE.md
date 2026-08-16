# UBW NEWS — Setup Guide (Self-Hosted, No Claude Branding)

This turns `ubw-news-selfhost.html` into a real public website with your
own link, no `claude.ai` anywhere. It uses **Firebase** (a free Google
service) to store posts, likes, comments and reposts so every visitor
sees the same live data — exactly like the Claude-hosted version did,
just without Claude in the loop.

Total time: ~10 minutes. No coding required, just copy-pasting values.

---

## Part 1 — Create your Firebase project (free)

1. Go to https://console.firebase.google.com and sign in with any Google account.
2. Click **Add project**. Name it anything (e.g. `ubw-news`). You can skip Google Analytics — not needed.
3. Once the project is created, click the **`</>`** (web) icon on the project overview page to register a web app.
4. Give it a nickname (e.g. `ubw-news-web`) and click **Register app**. You do **not** need Firebase Hosting checked at this step.
5. Firebase will show you a code block that looks like this:

   ```js
   const firebaseConfig = {
     apiKey: "AIza...",
     authDomain: "ubw-news-xxxxx.firebaseapp.com",
     projectId: "ubw-news-xxxxx",
     storageBucket: "ubw-news-xxxxx.appspot.com",
     messagingSenderId: "123456789",
     appId: "1:123456789:web:abcdef123456"
   };
   ```

   Copy this whole object.

## Part 2 — Paste your config into the site

1. Open `ubw-news-selfhost.html` in any text editor (Notepad, VS Code, etc.).
2. Find this block near the top of the `<script>` tag:

   ```js
   const firebaseConfig = {
     apiKey: "YOUR_API_KEY",
     ...
   };
   ```

3. Replace it entirely with the config you copied from Firebase in Part 1.
4. Save the file.

## Part 3 — Turn on Firestore (the database) and lock it down properly

1. In the Firebase console, open your project and go to **Build → Firestore Database** in the left sidebar.
2. Click **Create database**. Choose any location close to your readers (e.g. `asia-south1` for India). Click **Next**.
3. Choose **Start in production mode** (not test mode — production mode starts locked, and we'll open it up carefully with the rules below). Click **Create**.
4. Go to the **Rules** tab in Firestore and replace the contents with:

   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {

       match /posts/{postId} {
         allow read: if true;

         allow create: if request.auth != null
           && request.resource.data.uid == request.auth.uid
           && request.resource.data.author is string && request.resource.data.author.size() <= 30
           && request.resource.data.category in ['world','offtopic','opinion']
           && request.resource.data.title is string && request.resource.data.title.size() <= 140
           && request.resource.data.body is string && request.resource.data.body.size() <= 3000
           && request.resource.data.likes == []
           && request.resource.data.reposts == []
           && request.resource.data.commentCount == 0;

         allow update: if request.auth != null
           && request.resource.data.diff(resource.data).affectedKeys().hasOnly(['likes','reposts']);

         allow update: if request.auth != null
           && request.resource.data.diff(resource.data).affectedKeys().hasOnly(['commentCount']);

         allow delete: if false;

         match /comments/{commentId} {
           allow read: if true;
           allow create: if request.auth != null
             && request.resource.data.uid == request.auth.uid
             && request.resource.data.author is string && request.resource.data.author.size() <= 30
             && request.resource.data.text is string && request.resource.data.text.size() <= 500;
           allow update, delete: if false;
         }
       }
     }
   }
   ```

   Click **Publish**.

5. Now go to **Build → Authentication** in the left sidebar, click **Get started**, then under **Sign-in method** enable **Anonymous**. Click **Save**.

   This is the key piece: every visitor is silently signed in the moment
   the page loads — they never see a login screen, nothing changes for
   them — but it gives Firestore a real, verified identity to check
   against. Combined with the rules above:

   - **Reading stays fully open** to everyone, no exceptions.
   - **Writing requires a real authenticated session** from your actual site — a script hitting the database directly, without ever loading your page through a browser, can't sign in and can't write.
   - **Nobody can delete anything**, ever, from the client — `allow delete: if false` blocks it outright.
   - **Nobody can edit a story's title, body, author or category** after it's posted — updates are only allowed to touch `likes`, `reposts`, or `commentCount`, and only in the shape the app itself produces.
   - **Comments can never be edited or deleted** once posted, by anyone, from the client.

   This isn't unbreakable — a determined person could still write a fake
   "browser" that signs in anonymously and then posts spam through the
   allowed fields, same as anyone could spam a real comment section. But
   the "open the database directly and wipe everything" risk you were
   worried about is closed.

## Part 4 — Put the site online with a real link

You have two easy, free options:

### Option A: Firebase Hosting (simplest, same ecosystem)

1. Install Node.js if you don't have it (https://nodejs.org).
2. Open a terminal and run:
   ```
   npm install -g firebase-tools
   firebase login
   ```
3. In the folder containing `ubw-news-selfhost.html`, run:
   ```
   firebase init hosting
   ```
   - Select your project.
   - Public directory: press Enter for `public`, then move your HTML file into that folder and rename it `index.html`.
   - Configure as single-page app: **No**.
4. Deploy:
   ```
   firebase deploy
   ```
5. Firebase gives you a live link like `https://ubw-news-xxxxx.web.app` — no Claude branding, works for anyone.

### Option B: Netlify (drag-and-drop, no terminal)

1. Go to https://app.netlify.com/drop
2. Rename your file to `index.html` and drag the folder containing it onto the page.
3. Netlify gives you an instant public link (e.g. `https://random-name.netlify.app`), fully yours.
4. You can later add a custom domain (like `ubwnews.com`) for free in Netlify's site settings if you buy one from any domain registrar.

---

## Once it's live

- Anyone with the link can read the wire instantly, no login.
- The first time someone posts, likes, comments, or reposts, they're asked for a name — saved in their own browser after that.
- All of it updates live for everyone, because Firestore pushes changes instantly to every open tab.

If anything breaks after deploying (blank page, "Setup needed" message
still showing, etc.), the most common cause is a typo in the pasted
config — double-check it matches exactly what Firebase showed you.

---

## Part 5 — Make it findable on Google (without sharing the link)

Deploying gives the site a real address. It does **not** automatically
make Google show it in search results — you have to tell Google it
exists, and then wait. Here's the real process, honestly:

1. **Replace the placeholder domain.** Three files — `ubw-news-selfhost.html`, `robots.txt`, and `sitemap.xml` — have `https://YOUR-DOMAIN-HERE/` written in them. Once you know your real live link (from Part 4, e.g. `https://ubw-news-xxxxx.web.app` or your Netlify link), open all three files and replace every `https://YOUR-DOMAIN-HERE/` with your real link. Re-deploy after this change.
2. **Put `robots.txt` and `sitemap.xml` at the root of your site** — same folder as `index.html` — so they're reachable at `yoursite.com/robots.txt` and `yoursite.com/sitemap.xml`.
3. **Go to Google Search Console:** https://search.google.com/search-console
4. Add your site as a property (paste your live URL), and verify ownership — Search Console will walk you through this (usually a meta tag or DNS check, depending on your host).
5. In the left menu, go to **Sitemaps**, and submit `sitemap.xml`.
6. Under **URL Inspection**, paste your homepage URL and click **Request Indexing**.

**What to actually expect:**
- Google usually takes anywhere from a few days to a couple of weeks to crawl and index a brand-new site — there's no way to speed this up beyond what's above.
- Once indexed, searching your exact brand name — `UBW NEWS` — is what's realistically likely to surface it. Ranking for broad terms like "latest news" against large, established outlets is not something a new site can expect to win, regardless of setup.
- The individual stories people post won't reliably show up in Google search results — they're loaded dynamically from Firestore after the page loads, and search engines mostly see the empty page shell, not the live feed content. If you specifically want individual stories to be searchable and indexable one day, that needs a different setup (server-rendered pages), which is a bigger project than this file — let me know if you ever want to go there.
