## LSPatch 1.1 — what came back from the device

1.0 rebuilt LSPatch on Vector. 1.1 is what running that rebuild on real devices sent back: managers
reaped in the background, a log panel showing everything except the app you had patched, a patch
reading an apk the system had moved out from under it, and a Shizuku channel that answers "granted"
long after it was revoked.

> [!IMPORTANT]
> The loader inside a patched app changed. Apps patched with 1.0 keep running as they are, but
> **Update loader** on an app's detail page — or a re-patch — is what gives them the manager
> independence and the full log below.

### 🔗 A patched app no longer needs the manager to be alive
A manager-backed patch asked the manager for its modules the instant the app started, and the manager
is an ordinary app: on a device that reaps background processes, or after a force-stop, that ask went
unanswered and the app started with no modules, no explanation, and five seconds of its own start-up
already spent waiting.
- **📋 It keeps its own list.** Which modules it runs and where their APKs are, read from that copy
  when the manager does not answer — the code still loaded from the module's installed APK by the same
  step the served path uses, so a restored module and a served one are the same dex from the same file.
  Because a miss is no longer fatal, the bind budget falls from five seconds to one and a half.
- **🔄 The binding is kept, not made once.** A manager reaped for memory returns on the binding that
  still exists; a force-stop breaks it for good and is rebound with a growing delay. Reconnecting
  restores the hot-reload channel and a live service per module.
- **🐕 Uptime, asked for last.** A foreground presence that stands down when nothing on the device is
  patched and returns after a reboot or a manager update; and where Shizuku is granted, an opt-in
  watchdog in the shell service starts the manager again when it finds it gone.

### 🪵 Logs that contain the app you patched
The framework stream was routed by a set of uids built before the package scan that fills it had
finished, so it held the manager's uid alone for the life of that collector — and everything a patched
app and its modules wrote fell to the branch that keeps only `AndroidRuntime` warnings and fatals.
- **🎯 Routed by uid, not by tag.** The set is built after the scan and pushed to the running collector
  on every tick, so an app patched later joins in place, without a restart and without a gap. No tag
  list survives anywhere in it: a module names its own tag, and no list written in advance can know it.
- **📣 The release loader keeps its own lines.** It was discarding everything the signature bypass, the
  module loader and the service client wrote — exactly the half of a patched app's log that explains
  the patch.
- **🔎 Filter by writer.** The scan counts uids alongside tags and levels, so the log's processes are a
  filter of their own, and a native crash dump reads as one foldable entry rather than a page of
  fragments.

### 🌐 The Store resolves over HTTPS
Every remote fetch — the catalogue, per-module detail, downloads, the GitHub release feed and
self-update — goes through Vector's shared OkHttp client with its DoH resolver, so a network that
blocks or tampers with DNS can no longer keep the Store quietly empty. A menu button in the Store
header opens the switch and a line saying what the last lookup actually did: resolved through
Cloudflare, bypassed by a proxy, or fallen back to the system resolver.

### 🩹 A patch reads the file you chose
- **📍 Target apks resolved when the patch runs (#104).** A request records its target's paths when the
  target is picked, then outlives that reading — saved to disk, re-entered after the process is killed,
  re-run by a retry. Where an installed app keeps its apks is the system's choice, so a recorded path
  is treated as a name and resolved again as the job starts.
- **🗃️ A picked apk kept out of the cache.** A file chosen from storage is copied into the app, and that
  copy lived in the cache directory, which the platform may evict under storage pressure — taking the
  only copy of the source with it. It lives in no-backup storage now, for the life of the request.
- **🧩 An apk that ships an apk is not a bundle.** A zip holding a nested `*.apk` was read as an app
  bundle, so an app carrying one as an asset — a skin, a plugin, an embedded installer — had its own
  apk discarded and the asset patched under a package of its own. A file whose manifest sits at the
  root of the zip is an apk, and is used as it is.

### 📦 Installing tells you what happened
- **⏱️ No unbounded waits.** A callback from a system service may never arrive, and waiting on one
  presents as a hang: no result, no error, nothing written for anyone to read afterwards. Every wait
  has a deadline now, and expiry is not read as failure on its own — the package manager is asked what
  the device actually has, so an install whose status was lost is recognised rather than repeated.
  Where a route is shut, the artifact is offered both ways: handed to an installer the device has, or
  saved wherever you ask.
- **✍️ The uninstall prompt only when the signatures differ.** Whether Android would take the new apk as
  an update was decided by asking a different question — whether the installed build carried LSPatch's
  own metadata. Sign with a keystore that matches the installed app and the update would have
  succeeded, yet you were sent to the uninstall prompt and told to give up your data first. The
  signing certificates themselves are compared now.

### 🔐 Shizuku checked, not assumed
- **🚦 One guard per call.** Every entry point funnels through a guard that turns anything thrown into a
  typed failure naming the operation and the state that stopped it — kept in a bounded history for the
  report, surfaced once per distinct problem, and carrying the remedy and, where one exists, the trace.
- **♻️ Authority re-read.** The client library answers a permission check from a static it stops
  refreshing once the answer is affirmative, and the server offers no revocation callback, so the
  server itself is asked at each point of use and whenever the app returns to the foreground.
- **🧷 The user service stays reachable.** Keep rules pin the service and its generated AIDL, without
  which a minified build could not start it at all; and a binder that stops answering without its
  connection callback firing is cleared by the binder's own death, instead of sitting in place for
  every later call to route into.

### 🧾 A bug report that survives the failure
A report gathered entirely through a privileged channel loses precisely the evidence it exists for
whenever that channel is the fault. What an unprivileged process can reach is read directly — its own
log, which the platform grants any app for its own uid, and its own entries under `/proc` — and the
channel is consulted once for the whole archive. Files are streamed whole through a ranged read rather
than truncated, since a truncated crash dump is not a smaller report but a wrong one. The archive also
records what this device can install with at all.

### 🧹 Also
- **💥 Manage no longer crashes while listing an integrated patch's modules.** Two of the screen's
  subscriptions unpacked the same embedded module at once, and one wrote the apk the other had handed
  to the platform to read — which ends the process with a bus error, not an exception it could catch.
- **🏷️ The loader version left the icon column.** It printed the word "Rolling" for a manager-backed
  patch, which all 18 locales read as the verb — French told you your patched app had been *cancelled* —
  and at three digits it measured wider than the icon, so rows drifted out of line. It is a pill beside
  the patch mode now, shown only where the recorded number is the one that runs.
- **🔔 The log-monitoring notification speaks your language.** It was the one piece of user-facing text
  shipping with no translation at all.
