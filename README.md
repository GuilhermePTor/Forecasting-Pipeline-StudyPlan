# Study Schedule — Synced Checklist

A single self-contained HTML page (`study-schedule-daily.html`) with a 42-day study checklist. Progress (checked/unchecked boxes) is stored in a free Firebase Realtime Database, so checking a box on one device shows up on every other device within a second or two.

If Firebase isn't configured yet, or fails to connect, the page still works fully — the checklist renders and checkboxes still update the progress meter — it just won't save between visits until Firebase is set up.

## 1. Host it on GitHub Pages

1. Make sure the repo is **public** (required for GitHub Pages on the free plan): **Settings → General → Danger Zone → Change visibility → Public**.
2. Make sure `study-schedule-daily.html` sits at the **top level** of the repo (not in a subfolder). If you want it to load at your site's root URL with no filename in the address bar, rename it to `index.html`.
3. Go to **Settings → Pages**.
4. Under **Build and deployment → Source**, choose **Deploy from a branch**.
5. Set the branch to `main` and the folder to **`/ (root)`**, then click **Save**.
6. Wait a minute or two — check the **Actions** tab if you want to confirm the build succeeded.
7. Your site will be live at:
   ```
   https://<your-username>.github.io/<repo-name>/
   ```
   (or just `https://<your-username>.github.io/` if the repo itself is named `<your-username>.github.io`).

## 2. Create the Firebase project

1. Go to [console.firebase.google.com](https://console.firebase.google.com) and sign in with any Google account.
2. Click **Create a Firebase project** (or **Add project**).
3. Type a project name — anything, e.g. `study-schedule`. Firebase auto-generates a project ID from it; you can edit that ID once via the pencil icon, but not after creation.
4. It may ask about Google Analytics or "Gemini in Firebase" — you can skip/disable both, they're not needed here.
5. Click **Continue** / **Create project** and wait ~30 seconds while it provisions.

## 3. Turn on the Realtime Database

1. In the left sidebar of your new project, under **Build**, click **Realtime Database**.
2. Click **Create Database**.
3. Pick a location (closest region to you is fine — this can't be changed later, but it barely matters at this scale).
4. Choose **Start in test mode** — this opens read/write access for 30 days so you can get it working immediately. Step 5 below replaces this with a permanent rule.
5. Click **Enable**. You'll land on an empty data viewer — note the URL shown at the top of that page (something like `https://study-schedule-default-rtdb.REGION.firebasedatabase.app/` or `...firebaseio.com/`). This needs to match exactly what you paste into the file below — Firebase uses different URL formats depending on the region you picked, so don't assume it matches the placeholder pattern in the file.

## 4. Register a web app and get the config object

1. Click the **gear icon** near the top-left (next to "Project Overview") → **Project settings**.
2. Scroll to **Your apps**, click the **`</>`** (web) icon to register a new app.
3. Give it a nickname (just for your reference in the console — visitors never see it, e.g. `study-schedule-site`).
4. If it offers to set up Firebase Hosting, skip it — you're using GitHub Pages instead.
5. Click **Register app**. Firebase shows you a code block — the part between the curly braces is your config object. That's what you need.
6. You can always get back to this later: **Project settings → General → Your apps → click the app → "SDK setup and configuration" → Config**.

What each field actually does:

| Field | What it's for |
|---|---|
| `apiKey` | Identifies your project to Google's servers. Despite the name, it's not secret — it's normal and fine for it to be visible in your page's source. Actual security comes from the database rules, not from hiding this. |
| `authDomain` | Used for login flows. Not used by this checklist since there's no login, but harmless to leave in. |
| `databaseURL` | **The one field this project actually needs.** Points to your Realtime Database. |
| `projectId` | Your project's unique ID. |
| `storageBucket`, `messagingSenderId`, `appId` | Used by other Firebase features (file storage, push notifications). Not used here — fine to leave in, ignored by the script. |

## 5. Where to paste the config in the HTML file

Open `study-schedule-daily.html` and search for `YOUR_API_KEY` — it's near the **end** of the `<script>` block, inside a `try { }` block labeled `--- Firebase (optional layer) ---`:

```javascript
  // --- Firebase (optional layer) ---
  try {
    const firebaseConfig = {
      apiKey: "YOUR_API_KEY",
      authDomain: "YOUR_PROJECT.firebaseapp.com",
      databaseURL: "https://YOUR_PROJECT-default-rtdb.firebaseio.com",
      projectId: "YOUR_PROJECT",
    };
    firebase.initializeApp(firebaseConfig);
    ...
```

Delete the placeholder object (the four lines between `const firebaseConfig = {` and `};`) and paste in the whole object Firebase gave you in step 4 — even if it has extra fields (like `storageBucket` or `measurementId`) that weren't in the placeholder. That's fine, the extra fields are simply unused. Leave everything else in the file — the `try { ... } catch (e) { ... }` wrapper, the rest of the script — untouched.

## 6. Lock down the database rules

This replaces the 30-day test mode from step 3. In the Firebase console: **Realtime Database → Rules** tab, replace the contents with:

```json
{
  "rules": {
    ".read": true,
    ".write": true
  }
}
```

Click **Publish**. This keeps it working indefinitely instead of expiring in 30 days. It's still technically open to anyone who knows your database URL — fine for a personal checklist nobody else will stumble onto. See [Adding authentication](#adding-authentication-later) below if you want it locked to just you.

## 7. Commit and push

```bash
git add study-schedule-daily.html
git commit -m "Add Firebase sync"
git push
```

GitHub Pages rebuilds automatically, usually within a minute. Once it's live, checking a box on one device should show up on any other device that loads the page within a second or two.

## Adding authentication later

Right now the database rules (`.read: true, .write: true`) allow anyone who has the database URL to read or edit progress. Adding sign-in locks that down to just you. Pick one of the two options below — neither is implemented in the file yet, this just documents how.

### Option A — Email/Password

1. Firebase console → **Build → Authentication → Get started → Sign-in method** tab → enable **Email/Password**.
2. Go to the **Users** tab → **Add user** → set an email + password. This is a Firebase-only login (separate from any Google account) used just for this checklist — one account, used from every device.
3. Add one more SDK script tag in `<head>`, next to the existing two:
   ```html
   <script src="https://www.gstatic.com/firebasejs/10.12.2/firebase-auth-compat.js"></script>
   ```
4. Wrap the existing `<div class="wrap">...</div>` content with an id (e.g. `id="app"`) and add a small login form above it:
   ```html
   <div id="login-gate">
     <input type="email" id="login-email" placeholder="Email">
     <input type="password" id="login-password" placeholder="Password">
     <button id="login-btn" type="button">Sign in</button>
   </div>
   ```
5. In the script, gate the app behind sign-in and wire up the button:
   ```javascript
   firebase.auth().onAuthStateChanged(user => {
     document.getElementById('login-gate').style.display = user ? 'none' : 'block';
     document.getElementById('app').style.display = user ? 'block' : 'none';
   });

   document.getElementById('login-btn').addEventListener('click', () => {
     const email = document.getElementById('login-email').value;
     const password = document.getElementById('login-password').value;
     firebase.auth().signInWithEmailAndPassword(email, password)
       .catch(err => alert(err.message));
   });
   ```
6. Update the database rules:
   ```json
   {
     "rules": {
       ".read": "auth != null",
       ".write": "auth != null"
     }
   }
   ```

Don't hardcode the password anywhere in the file — anyone could view-source and read it. The point of the form is that the password lives only in your head; Firebase keeps you signed in on each browser after the first login, so you won't retype it every visit.

### Option B — Google Sign-In

Same effect as Option A, but a single click instead of a password to manage.

1. Firebase console → **Build → Authentication → Sign-in method** → enable **Google**.
2. **Authentication → Settings → Authorized domains** → add your GitHub Pages domain (e.g. `your-username.github.io`). Without this, the sign-in popup gets rejected.
3. Add the same `firebase-auth-compat.js` script tag as Option A, step 3.
4. Replace the login form with a single button:
   ```html
   <button id="google-signin-btn" type="button">Sign in with Google</button>
   ```
5. Wire it up:
   ```javascript
   firebase.auth().onAuthStateChanged(user => {
     document.getElementById('google-signin-btn').style.display = user ? 'none' : 'block';
     document.getElementById('app').style.display = user ? 'block' : 'none';
   });

   document.getElementById('google-signin-btn').addEventListener('click', () => {
     const provider = new firebase.auth.GoogleAuthProvider();
     firebase.auth().signInWithPopup(provider).catch(err => alert(err.message));
   });
   ```
6. Same rules change as Option A, step 6.

### Optional: signing out

Either option, add a button anywhere and bind it to:
```javascript
document.getElementById('signout-btn').addEventListener('click', () => firebase.auth().signOut());
```


