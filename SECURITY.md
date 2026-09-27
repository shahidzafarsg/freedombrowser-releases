# FreedomBrowser security model

This document explains what FreedomBrowser protects, how, and where the limits are. If you find a
weakness, please report it privately through
[GitHub security advisories](https://github.com/shahidzafarsg/freedombrowser-releases/security/advisories/new).

## What is protected

When FreedomBrowser is **locked or closed**, everything it stores on disk is encrypted:

| Data | How it is stored |
| --- | --- |
| History, bookmarks, settings, open tabs | Encrypted documents |
| Cookies, site storage (localStorage, IndexedDB), cache, saved site permissions | Sealed snapshot of the Chromium profile |
| Installed extensions and their data | Inside the sealed profile |
| Downloads | Encrypted as soon as they finish |

The only unencrypted file is `vault.json`, which holds the master key in wrapped (encrypted) form,
the key-derivation settings and the failed-attempt counter. It contains no browsing data.

Downloaded files never leave the vault on their own. They can be **viewed** inside FreedomBrowser
(pictures, PDFs, video, audio and text are decrypted in memory, chunk by chunk, and shown in a tab)
or **exported**, which writes an unencrypted copy to a place you choose. No other app can open a
download until you export it.

## How it works

* A random 256-bit **master key** (MK) is created on first run. All data keys are derived from it
  with HKDF-SHA256.
* With a **password**, the MK is encrypted with a key derived from the password by **Argon2id**
  (128 MiB of memory and 3 passes).
* A **recovery key** (160 random bits, shown once) wraps the MK a second time, so a forgotten
  password is not fatal.
* **Two-step verification** uses standard TOTP (RFC 6238), compatible with Google Authenticator,
  Microsoft Authenticator, Aegis and others. The TOTP secret is stored inside the password-encrypted
  payload, and used codes cannot be replayed.
* Without a password, the MK is protected by the operating system: DPAPI on Windows and the Keychain on
  macOS.
* Files use the **FBEF** format: AES-256-GCM in 64 KiB chunks with a per-file key and the STREAM
  construction (chunk index and a "last chunk" flag in the nonce), so tampering, reordering and
  truncation are all detected. The header is authenticated with every chunk.
* The browser engine's profile is **sealed** after the engine has stopped, to the vault's X25519
  public key. This means sealing needs no password, so it also works after a crash: the next start
  seals whatever was left behind before it shows the lock screen. The matching private key is itself
  encrypted under the MK.

## Locking, auto-lock and panic wipe

* **Lock** closes every window, stops the engine, seals the profile, shreds the working copy and
  restarts into the lock screen. Tabs reopen after unlocking.
* **Auto-lock** triggers after a chosen idle time, when the system locks or sleeps, and optionally
  when a window is minimised.
* **Panic wipe** (toolbar button, menu, or `Ctrl+Shift+Alt+X`, even in the background)
  overwrites and deletes the vault header first, which destroys the wrapped keys, then deletes every
  encrypted file.
* **Erase after wrong passwords** can wipe everything after 5, 10 or 20 failed attempts in a row.
  Failed attempts are also slowed down: after five, each further attempt waits longer (30 seconds,
  doubling, up to an hour).

## Limits you should know about

* **While FreedomBrowser is unlocked**, the engine needs its profile as ordinary files. That working
  copy sits in `%LOCALAPPDATA%\FreedomBrowser\work` (Windows) or
  `~/Library/Application Support/FreedomBrowser/work` (macOS), protected by your account's file
  permissions. It is shredded when the session ends.
  Downloads, history and bookmarks are never written there in plaintext.
* **After a crash or power loss**, the working copy stays on disk until FreedomBrowser next starts,
  when it is sealed and shredded before anything else happens.
* **Shredding is best effort.** Overwriting files does not guarantee erasure on SSDs and flash
  storage, which remap blocks. FreedomBrowser therefore relies on encryption and on destroying keys,
  not on overwriting. Full-disk encryption (BitLocker or FileVault)
  remains strongly recommended.
* **Two-step verification is enforced by the app.** It stops someone who learns your password from
  opening FreedomBrowser. Someone who has your password, a copy of your files and the skill to modify
  the program could bypass the code check, because an offline app has to store the TOTP secret
  somewhere. The recovery key opens the vault without the second factor, by design.
* **Keys are in memory while unlocked.** Malware running as your user, or anyone with access to your
  unlocked computer, can read what the browser can read.
* **Extensions** run inside your profile and can see the pages you allow them to. Only install
  extensions you trust.
* **This is not an anonymity tool.** Websites, your network and your DNS provider can still see your
  IP address and which sites you connect to. Use a trustworthy VPN or Tor for anonymity.
* **Installers are not yet code-signed.** Windows SmartScreen and macOS Gatekeeper will warn the
  first time. Check that you downloaded from the official GitHub releases page.

## Privacy defaults

* Ad and tracker blocking with EasyList and EasyPrivacy through Ghostery's engine.
* HTTPS-only mode, third-party cookie blocking, Global Privacy Control and Do Not Track.
* No telemetry, no crash reports, no accounts. The only network requests FreedomBrowser makes on its
  own are filter-list updates, extension updates and (if enabled) a daily check of the GitHub
  releases API for a newer version.
