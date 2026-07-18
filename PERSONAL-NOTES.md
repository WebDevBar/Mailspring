# Personal fork notes (WebDevBar/Mailspring)

Tracking our `personal:` customizations and future improvements. Companion to the
`personal:` commits (window icon, themed `.desktop` icon, Fedora bundle-only build,
Linux Unity LauncherEntry badge over DBus).

## Shipped customizations

- **Unified-inbox badge + tray count** (2026-06-25). The dock/LauncherEntry badge and the
  system-tray unread icon now always reflect total unread across ALL accounts, regardless of
  which folder/perspective is selected. Previously they counted only the focused perspective's
  accounts, so new mail in a non-selected account was invisible unless the unified inbox was
  selected (and with a single account, switching folders changed the count).
  - File: `app/src/flux/stores/badge-store.ts`.
  - Change: count `AccountStore.accountIds()` (all accounts) instead of
    `FocusedPerspectiveStore.current().accountIds`; added a direct `CategoryStore` listener so the
    count still recomputes once inbox categories load asynchronously at startup (the old
    `FocusedPerspectiveStore` listener had provided that wakeup indirectly). `AccountStore` is a
    named export (`import { AccountStore }`).
  - Single-account safe: `accountIds()` returns `[oneId]`, so it just counts that inbox. No
    "unified inbox" perspective object is required (that UI only appears with 2+ accounts).
  - Peer-reviewed (Codex, 3 rounds): caught the missing `CategoryStore` listener and the
    named-import before the build; round 3 clean.
  - Tray reads `BadgeStore.unread()` (`internal_packages/system-tray/lib/system-tray-icon-store.ts`),
    so this one store fix covers both badge and tray.
  - **Requires rebuild**: `npm run build` -> `cp -a app/dist/mailspring-linux-x64/. ~/.local/share/mailspring/`.

- **Linux password-store backend detection** (2026-07-16). Backport of upstream **PR #2660**
  (open / unmerged as of this date). On Linux, before app-ready, if the user hasn't passed
  `--password-store`, query D-Bus `NameHasOwner` for `org.freedesktop.secrets` and, if present,
  force `--password-store=gnome-libsecret`. Fixes the "Mailspring could not store your password
  securely / encryption not available" dialog that appeared after the Fedora KDE
  `kf6-kwallet-6.28.0` update (2026-07-14), when Electron's default `kwallet6` backend went
  unavailable.
  - File: `app/src/browser/main.js` (27-line block after the `js-flags` switch, before
    `parseCommandLine`). Kept close to upstream so the eventual merge is near-clean.
  - Root cause (grounded + Codex-reviewed): Electron selects a keyring backend from the desktop
    session (`kwallet6` on KDE). After the kwallet 6.28 update that backend reported encryption
    unavailable (`isEncryptionAvailable=false`, wallet `isOpen=false`); the libsecret path (via
    the new `ksecretd` daemon) still worked, so forcing `gnome-libsecret` bypasses the failing
    path. The precise reason kwallet6 failed is inferred, not proven - only the working bypass is.
  - Not an Electron bug: choosing the backend is the app's responsibility via `--password-store`;
    Electron already supports both `kwallet6` and `gnome-libsecret`. Upstream's own fix (#2660)
    does exactly this in-app.
  - One-time cost after switching backends: accounts re-auth once, and old KWallet entries are
    left stale (harmless).
  - **Requires rebuild**: `npm run build` -> `cp -a app/dist/mailspring-linux-x64/. ~/.local/share/mailspring/`.
  - Rebase note: keep until upstream ships #2660 in a tagged release, then drop as redundant.

## Future improvements / TODO

- **Remove Pro upsell messages across the board** (noted 2026-06-22). A new `personal:` set of
  commits + rebuild. Surfaces found in the source:
  - `app/internal_packages/participant-profile/lib/sidebar-participant-profile.tsx` — contact-sidebar
    "Try Mailspring Pro" upsell (the visible one).
  - `app/src/components/feature-used-up-modal.tsx` + `app/src/flux/stores/feature-usage-store.tsx` —
    "feature used up / upgrade" limit modal (snooze, send-later, read-receipts, link-tracking, mail-merge).
  - `app/internal_packages/notifications/lib/items/please-subscribe-notif.tsx` — "please subscribe" notif.
  - `app/internal_packages/onboarding/lib/page-initial-subscription.tsx` — onboarding subscription/trial page.
  - Scattered pro-feature toggle buttons (e.g. `app/src/components/metadata-composer-toggle-button.tsx`).


## Update log

- **2026-07-16** - Backported upstream PR #2660 (Linux password-store detection) into
  `app/src/browser/main.js` to fix the post-`kf6-kwallet-6.28` "could not store your password
  securely" error on Fedora KDE. See Shipped customizations. Requires a rebuild to take effect.
- **2026-07-04** - Checked upstream (`Foundry376/Mailspring`). Latest tagged release is still
  **1.22.0** (our base). Upstream `main` is 2 untagged commits ahead, both null-crash IPC bugfixes:
  #2757 (SendDraftTask / thread-sharing / send-and-archive) and #2758 (three IPC handlers when
  `BrowserWindow.fromWebContents()` returns null). **Decision: HOLD** - not worth a full Mailspring
  rebuild for 2 untagged commits. Rebase our `personal:` patches and rebuild when a tagged **1.22.1+**
  release ships.
