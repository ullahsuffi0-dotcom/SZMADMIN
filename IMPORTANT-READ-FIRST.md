# SZM School Admin System — Critical Fix Notes

## What was wrong & what was fixed

### Issue 1 — Recycle bin was not capturing all deletions
**Root cause:** Only 3 collections were being tracked. Most delete operations were permanent.
**Fix:** Expanded tracked collections to cover students, teachers, complaints, infraction forms, visit forms, schools, sessions, custom pages, etc. Added recycle entries for `delSession`, `customPageDel`, and `cmDelPage` which used `_raw_set` and bypassed the hook.

### Issue 2 — Changes only saved locally, not syncing live across the world
**Root cause:** Firebase sync was disabled if `bootFirebaseSync()` failed for any reason. The 4-second SDK load timeout was too short. The `__SYNC_ON` flag was the **only** gate for cross-device sync, and it was set only at the END of a fragile boot sequence.

**Fix:** Three architectural changes:
1. The localStorage interceptor now syncs **as soon as `window.__FB.db` is available**, not waiting for `__SYNC_ON`
2. `bootFirebaseSync` now waits up to **30 seconds** for Firebase, and retries indefinitely in the background if Firebase is still loading
3. `__SYNC_ON` is set **before** the initial sync-down, so user writes always reach Firebase

### Issue 3 — Login/password changes not propagating across devices
**Root cause:** Three different places in the code were unconditionally overwriting the incoming backend admin password with the local one — so password changes never propagated.
**Fix:** Added `_pwdChangedAt` timestamps to every password change. All three sync paths now compare timestamps and keep the **newer** password.

---

## ⚠️ MUST-DO STEPS for the fixes to work live worldwide

### Step 1 — Deploy the updated rules to Firebase

The Firebase Realtime Database rules in `firebase-rules.json` are **not deployed automatically**. You MUST paste them into the Firebase Console:

1. Go to https://console.firebase.google.com
2. Open your project (`szm-admin-system`)
3. Click **Realtime Database** → **Rules** tab
4. **Replace the entire rules text** with the content of `firebase-rules.json`
5. Click **Publish**

The new rules are:
```json
{
  "rules": {
    "data": { ".read": true, ".write": true },
    "broadcasts": { ".read": true, ".write": true },
    ".read": false,
    ".write": false
  }
}
```

These rules allow anyone to read/write to `/data` and `/broadcasts` paths — required for the global multi-school sync. Lock these down later only with full Firebase Auth integration.

### Step 2 — Redeploy the HTML files

Whichever service you use (Netlify, Vercel, Firebase Hosting), **redeploy** so your live URL serves the fixed code. The fixes are in `index.html`, `backend.html`, `client.html`, and the `deploy/` folder copies.

**Firebase Hosting:**
```
firebase deploy --only hosting
```

**Netlify:** drag the SZM folder into app.netlify.com/drop, or push to your connected repo.

**Vercel:**
```
npx vercel --prod
```

### Step 3 — Clear browser caches everywhere

Old code might be cached on devices. On each device:
- Open the site, press **Ctrl+Shift+R** (Windows) or **Cmd+Shift+R** (Mac) to force-reload
- Or open DevTools → Network tab → check "Disable cache" → reload

---

## How to verify it's working

After redeploying and clearing caches:

1. **Sync indicator** — Top-right corner should show a green "Synced" dot. If it shows "Offline" or "Local", Firebase isn't connecting. Check the browser console (F12) for Firebase errors.

2. **Test cross-device sync** — Log in on Device A. Add a student. Open the same school on Device B. Within ~1 second the student should appear.

3. **Test recycle bin** — On Device A, delete any student/teacher/complaint. Open the Backend Admin portal on Device B. Go to Recycle Bin. The deleted item should appear with the deleter's name and timestamp.

4. **Test password sync** — Change a staff password on Device A. Try to log in with the new password on Device B. It should work immediately.

---

## If still broken — debugging checklist

Open the browser console (F12) on the affected device and look for:

- **`[FB] Firebase SDK slow to load`** — Network blocked Firebase. Check firewall/extensions.
- **`[FB] Write failed: ... permission denied`** — Rules not updated. Redo Step 1.
- **`[FB] Bulk upload failed`** — Same as above, rules issue.
- **Green "Synced" dot but data not propagating** — Other device hasn't loaded the new code; redo Step 3.

You can also force-pull the recycle bin from Firebase by opening it and clicking the new **🔄 Refresh** button at the top.
