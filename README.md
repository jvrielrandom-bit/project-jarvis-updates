# JARVIS MK1

A local-first personal AI assistant inspired by the linked Jarvis demo. The dashboard includes an animated core, conversational voice UI, removable widgets, live weather, web research, YouTube search and caption summaries, screen analysis, Google Calendar, and Gmail.

## Desktop app

On Windows, install Node.js 24 or newer, then run:

```powershell
npm install
npm run desktop:build
```

The Electron NSIS installer is created under `dist/`. Installed Jarvis keeps running in the system tray when its window closes so its global hotkeys and reminders keep working. Use the tray menu to show or quit Jarvis.

The installer does not include provider API keys. On first launch Jarvis creates `%APPDATA%\com.Jarvis.MK1\.env` from a small template. Configure an OpenRouter primary key before using chat. All installed versions using the stable `com.Jarvis.MK1` app identifier share this settings file, so updates retain provider keys and the optional Fish Audio key. Optional speech replies use Fish Audio's `s2.1-pro-free` model. Jarvis also generates its local token encryption key and bundles Python for document analysis and SQLite for private persistent memory. PowerShell uses Windows PowerShell already included with supported Windows versions; no separate language installation is needed.

For development, install Node.js 24 or newer, copy `.env.example` to `.env`, then run `npm install` and `npm run desktop:dev`. The local server and browser launch scripts are also available with `npm start`, `start.cmd`, or `run-app.cmd`; `run-app.cmd` opens your Windows default browser. `npm run build:assistant` compiles the TypeScript assistant engine into the JavaScript used by the app.

## Provider setup

### OpenRouter

Create API keys in your OpenRouter account and add them to the per-user `.env` file opened from **Settings → Connections → Open .env file**:

```dotenv
OPENROUTER_API_KEY=your-primary-key
OPENROUTER_FALLBACK_API_KEYS=your-backup-key-one,your-backup-key-two,your-backup-key-three
OPENROUTER_MODEL=nvidia/nemotron-3-ultra-550b-a55b:free
```

Set one required primary key and up to three comma-separated backups. Jarvis always uses OpenRouter. `OPENROUTER_MODEL` picks the model; the supplied template uses a free conversational model. `openrouter/free` is treated as the default model because that alias routes randomly and can select a non-chat classifier instead of a conversational model.

Every configured key is tried in order. A key that is rejected (401/403), out of credits (402), or rate-limited (429) does not stop the request; Jarvis moves to the next key and reports the limit only when all keys fail. Provider keys remain editable plaintext in `.env`, including keys saved from Settings. Anyone or any process that can read this file can use those keys, so protect access to your Windows account and configuration folder. Legacy `dpapi:v1:` values are decrypted into plaintext on first startup and require the same Windows account that encrypted them. Restart Jarvis after changing `.env`. OpenRouter credits, model/provider limits, and rate limits still apply; multiple keys do not provide unlimited usage or bypass those limits.

### Fish Audio

Create a key at [Fish Audio API Keys](https://fish.audio/app/api-keys), then save it in **Settings → Connections**. Jarvis always uses Fish Audio's `s2.1-pro-free` model and voice reference `14129c3e320149449d6bada6862f7338` for text-to-speech; there is no local speech-engine switch or fallback. Voice replies are on by default for new settings. Replies up to 500 characters are spoken after the answer is complete; longer replies remain text-only. Failed Fish Audio requests or playback are retried up to three times, then Jarvis reports the error. Voice replies can be enabled or disabled in **Settings → Voice & behavior** or by asking Jarvis.

### Speech recognition and microphone

Jarvis transcribes with the engine selected in **Settings → Voice & behavior**. The desktop app uses the bundled on-device Vosk model, because the Electron shell has no online speech service; when Jarvis runs in an ordinary browser (`npm start`, `start.cmd`, `run-app.cmd`) it can use the browser's speech recognition instead. Choosing a specific microphone always uses on-device recognition, since browser recognition can only listen to the Windows default input. On-device audio never leaves this PC.

There is no always-on listening. The microphone opens only for one question at a time, normally by pressing **Alt+Space** anywhere on the computer: Jarvis comes to the front, listens until you pause, and sends what you said. Pressing the same key again, or **Stop listening**, ends the recording without sending it. A spoken “Jarvis” at the start of the question is dropped, so “Jarvis, what's the weather” and “what's the weather” behave the same.

**Alt+J** brings Jarvis to the front from any app, **Ctrl+Alt+L** locks him, and **Ctrl+Alt+X** stops everything from anywhere. All of those shortcuts are registered system-wide by the desktop shell; Settings → Voice & behavior shows the keys actually in use. Windows reserves Alt+Space for the window menu, so if it refuses that key Jarvis falls back to **Ctrl+Alt+Space** and says so there.

The same panel lists connected microphones, provides **Test mic** for a live input level, and **Run voice check** to verify the device and engine. If the selected microphone stops sending audio, Jarvis falls back to the system default input. Voice faults that survive a retry are researched and repaired in the background, as described below. You can also ask by voice or text: “list my microphones,” “use microphone <name>,” “check your microphone,” “fix your microphone,” “switch to on-device recognition,” or “automatic.”

Press **Esc** to interrupt whatever Jarvis is doing: it aborts the answer being generated, silences spoken audio, ends a recording in progress, and cancels a background self-repair pass, then hands the app back so you can type immediately. Anything already streamed stays on screen marked “— stopped”. Esc closes an open dialog first, so it never interrupts while a review card is on screen.

### Answers, research, and images

Jarvis has live web access. Ask it to research, compare, look something up, or answer anything time-sensitive — latest releases, prices, news, scores — and it searches and reads the strongest pages itself before answering: the reply comes back with what it found and links to the exact sources. You never have to open a link, run a search, or read a page to get the answer. A URL pasted into the conversation is read directly, and a `web_search` the model runs already includes the text of its best results instead of only snippets.

Long conversations stay on track. A single request runs at most six model/tool rounds; if the model keeps calling tools, an identical call repeating, or the model sends nothing, Jarvis stops the loop and forces one final written answer instead of returning a canned line. Blank turns and very old stuck replies are dropped from the history that is sent with each new question.

### Starting Jarvis twice

Jarvis serves the window from `127.0.0.1`. Starting a second copy no longer crashes: it prints **“Jarvis is already running at http://localhost:PORT”**, exits cleanly, and leaves the running instance untouched. If the port belongs to some other program, it says so and points at `PORT` in the local `.env` file instead of throwing an `EADDRINUSE` stack trace. The desktop shell also passes its process ID to the backend, so a window that crashes or is force-closed takes its local service with it rather than leaving an orphan that blocks the next launch. 1.2.7 also ignores inherited junk in `PORT` (some terminals export `PORT=0`): only a real port is honoured, and the desktop shell always pins its backend to 4180, so the window can no longer wait on a service that came up somewhere else.

**Speed:** a plain question carries a compact instruction set (about 1,600 tokens, roughly half the old prompt) and only the ten read-only tools it might need; anything that can open, write, schedule, email, research or look at the screen gets all 55 tools and full instructions. Each request also tells the model to send independent tool calls together in one reply, so a task needs fewer model round trips. Answer length is the biggest cost on a free model, so replies stay short unless detail is wanted.

Images you ask for appear in the chat and can be seen full size: click a generated image (or any image Jarvis shows) to open the large viewer. **Save to my PC** in the viewer or on the image card writes the file into your `Downloads` folder and tells you the exact path. Asking Jarvis to “save that image” saves it the same way, so no download link is needed. Version 1.2.7 generates images through Pollinations first, which answers in seconds, and falls back to AI Horde only if that service fails.

### The 1.2.7 assistant upgrades

Jarvis now keeps three kinds of long-term local state: a rolling conversation memory (so a cleared chat or a restart still remembers what was already discussed), a private **knowledge graph** of projects, contacts, snippets, decisions and documents, and **long-running goals** broken into dated steps that can be re-planned when something blocks them. Ask “what did I decide about X last month?” and Jarvis searches the graph; ask for a week-sized objective and it creates a tracked goal.

An **autonomous research agent** plans several searches, reads the strongest pages, compares independent sources and returns a cited brief. It runs locally and records every step. The same research is available from the Tools page and from ordinary “research X” questions, so no link ever needs to be opened.

**Automations** run multi-step workflows on a daily time, an interval, a one-off time, or when a watched file changes. A workflow can build the daily brief, run research, check the weather, read device status, or simply leave a notification. **The daily brief** reads today’s calendar, open tasks, active goals, weather, battery and connection, highlights overlapping events, and can be spoken as a short summary. Jarvis greets you on your first message of the day and offers it instead of dumping it unasked.

**Device awareness:** the top-left of the top bar shows battery and connection state, and Jarvis knows the remembered city and time zone for weather and local time. It mentions low power once and explains when a missing connection limits live answers. **Full access** in Settings → Local tools lets file writes, commands and batches run immediately, while Windows system folders, disk formatting, boot settings and security policy remain protected.

**Screen and control:** “look at my screen” captures the primary display directly, with no picker, and Jarvis answers from it. “Take over” lets Jarvis move the pointer, click, type and send key combinations to navigate for you; it hides first and is reopened from the tray. A camera question captures one still and sends it to a free vision model instead of running continuous detection.

**Local plugins:** drop a folder into `%LOCALAPPDATA%\Jarvis MK1\plugins` with a `plugin.json` (name, version, tools) and an `index.js` defining `async function run(tool, args, ctx)`. Tools appear to Jarvis as `plugin_<name>` and execute in a worker sandbox with no filesystem, process, or timer access, a timeout, and HTTPS-only fetches that cannot reach private addresses. Reload them with **Refresh plugins** in the API or restart Jarvis.

**Git and GitHub:** Jarvis can search public repositories, read repository details and text files, and inspect a local repository read-only with `status`, `log`, `diff`, `branch`, `show`, `remote` or `shortlog`. **Live widgets:** stock and crypto widgets refresh themselves every minute. **Serious mode** fades the whole console — core, panels, chat, apps and controls — into a red alert palette over a darker room, with a direct, professional tone, and is remembered between launches; toggle it from the top-left of the top bar, Settings, or by asking.

### Friend mode and persona memory

Jarvis keeps a small, private record of your world — the people in it, what you are building, how you like to be spoken to, and the rituals you share. It lives in `jarvis-persona.sqlite` in the app data folder, is never uploaded, and reaches a prompt only as a short, relevant line. Ask “remember that Anna is my sister” in conversation and he saves it himself through the `remember_persona` tool; **Settings → Jarvis** lets you add, review and forget entries, and shows the mood and the greeting he would use right now.

**Friend mode** (on by default) makes the tone personal rather than servile: contractions, dry humour, honest pushback, and callbacks to what you both remember. His mood shifts with the time of day and with serious mode, low battery or a dropped connection, and the first message of the day opens with a greeting that knows your name and the project currently in focus before offering the brief. With **Proactive remarks** on, those events arrive as one in-character line instead of a dry system notice — once per kind per day. Turning friend mode off restores a plain, professional tone.

### Phone link and privacy posture

**Settings → Phone** turns your phone into a remote control for the Jarvis on this PC. Turn remote access on and Jarvis listens on your private network only (`0.0.0.0`) while every request needs the pairing token; the pane shows a QR code and the exact link. The link opens `/remote` — a compact remote page with a talk button, a growing text field and a stop button — never the desktop console. Everything you say is sent to the PC, which does the thinking, shows the turn in its own transcript, speaks the reply when voice replies are on, and carries out the actions there. The phone only sends and displays, so there is one assistant and one memory, not two. A locked Jarvis covers the whole remote page with the unlock screen, so nothing can be typed, spoken or run from the phone until it is unlocked.

Phone voice and Face ID both need the page to be served over **https**: phones never give a page the microphone or a biometric prompt on a plain `http://192.168.x.x` link. Jarvis therefore makes its own local certificate authority and serves a second, private https listener one port above the normal one (`4181` by default, or `JARVIS_HTTPS_PORT`). **Download the trust certificate** in Settings → Phone, install/trust it once on the phone, then open the secure address shown there: no warning, and Face ID plus phone dictation work. If Tailscale is installed and connected, the pane also shows its `https://…ts.net` address, which works away from home with nothing published to the internet. On the plain link the remote says what is unavailable up front and points at the keyboard's dictation button, the PC camera unlock and the passcode, all of which work there.

Each phone now registers its own token on first use, listed under **Paired devices**. Revoking one phone leaves the others working; **New token** still invalidates every device at once. Optional **auto-lock** locks Jarvis after a chosen idle time, when Windows locks, or when the phone disconnects. While remote access is on Jarvis also keeps the PC awake (unless that switch is off), because a sleeping computer cannot answer the phone and a browser cannot wake it. Questions sent while the phone is offline, or while the PC is asleep, are kept on the phone and sent automatically when it is reachable again — the remote says how many are waiting.

### Speed, reliability and the new foundations

**Model tiering** sends a short conversational turn to a fast model and keeps the larger model for code, research and multi-step work. Set `JARVIS_FAST_MODEL` and `JARVIS_DEEP_MODEL` in `.env` (or fill the two fields in Settings → Jarvis) to use different models; left blank, both tiers use your configured model and its backups. Each answer reports which tier it used.

**Windows notifications** can tell you when a brief is ready, an automation finishes, a meeting summary is done, a condition fires or an update is found — even with the window closed. They are off until you turn them on in Settings → Access & automation, honour quiet hours that wrap past midnight, and never repeat the same notice twice in a row. **Send a test** proves the toast path first.

**Backup & restore** writes one encrypted file (AES-256-GCM with a scrypt key) containing conversations, memories, tasks, the knowledge graph, automations, goals, workflows, the lock and the persona. A wrong passphrase never yields partial data. A restore is checked, staged, and applied on the next start so an open SQLite database is never overwritten underneath itself.

**Health, watchdog and diagnostics:** `/api/health` reports database, model, speech, remote and plugin checks with timings and the last error; the settings panel shows the same thing in plain words. If the backend stops answering three health checks in a row, the desktop shell restarts it (five-minute cooldown, at most three times) and reloads the window.

**Search everything** indexes conversations, notes, meeting notes, tasks, knowledge, automations and workflows in SQLite FTS5. **Ctrl+K** opens the command palette over that index, every app, and the settings sections; type any words from a decision you made months ago and open the result directly. The composer's **⧉** button drops whatever you copied into the question.

**Meeting mode** (Apps → Meeting mode, or the palette) listens with the phone-style recogniser or accepts a pasted transcript, then turns it into a summary, decisions, action items with owners and open questions. Action items can become tasks, and the notes stay in the local Meetings memory.

**Conditions** extend automations beyond the clock: a file changing, the battery dropping below or rising above a level, the network going away, an app opening or closing, the PC sitting idle, or text appearing on the clipboard. Each fires once per transition, not on every sweep. **Workflows** are ordered tool calls with a preview: describe `tool | JSON arguments` per line, preview every step, save it, and run it — it stops at the first failure and records the result. Steps can reuse an earlier result with `$name`.

**This computer:** start Jarvis with Windows, start minimized, and **Ctrl+Alt+X** as a kill switch that stops the answer being written, the spoken reply, a recording and any phone turn immediately. The tray menu adds **Brief me**, **Meeting mode**, **Lock Jarvis** and **Stop everything**.

**Updates:** set `JARVIS_REPO` (for example `owner/jarvis-mk1`) or `JARVIS_RELEASES_URL` in `.env` to let **Check for updates** compare this build with the release feed and show its highlights. Nothing is downloaded or installed automatically. **Plugins** remain worker-sandboxed with HTTPS-only fetches, no filesystem or process access and a hard timeout; the diagnostics panel lists each loaded plugin, its version and its tools.

Keep the Jarvis window open while using the remote; if it is closed the page says so, because actions need the desktop session. **■** on the phone stops whatever the PC is doing. The token is stored in `jarvis-remote.json` in the app data folder, **New token** invalidates every paired device, and a paired phone stays linked until you turn remote access off or press **New token**, so there is no second QR scan. Away from home, install free Tailscale on the PC and the phone and connect both to the same tailnet — the link stays the same and nothing is exposed to the internet. Every request from another device is logged locally (time, address, path, allowed or refused) and listed in the pane.

### Locking Jarvis and unlocking it with your face

**Settings → Privacy → Lock Jarvis** asks for an unlock before Jarvis answers anything — the window, the API and the phone alike. Set a passcode, then turn the lock on; **Lock now** locks immediately, as do the **Lock** button in the header, **Ctrl+Alt+L** from any app, and **Lock** on the phone remote. A locked Jarvis returns 423 to everything except the unlock calls themselves, and the window watches the state every few seconds, so an unlock from the phone clears the computer's lock screen on its own.

Face ID is enrolled on the phone, never on the PC: while Jarvis is unlocked, open the remote over **HTTPS** (your Tailscale address, for example) and choose **Add Face ID to this phone**. That registers a WebAuthn platform credential — your phone's real Face ID or fingerprint, matched by the phone's secure hardware — and the PC only ever sees a signature it verifies against the stored public key. When Jarvis is locked, the remote offers **Unlock with Face ID**, and a successful check unlocks the PC as well as the phone page.

When the phone cannot do Face ID — a plain `http://192.168.x.x` link is not a secure origin, or no credential has been enrolled yet — the remote shows **Unlock with the PC camera** instead. That asks the Jarvis window on the computer to run Windows Hello: the camera on the PC looks at you, Windows does the matching, and only a signed yes is handed back to the local service. The passcode always works as the last resort, and five wrong attempts pause unlocking for thirty seconds.

Privacy is unchanged by any of this: there is no Jarvis server, no analytics, and no upload. The app data folder holds the databases, the remote token and the audit log; provider keys stay in the local `.env` file, and Google tokens stay encrypted. A phone loads only the shell and its own token — live data is never cached by the service worker, and the shell itself is fetched fresh each time so a new build reaches the phone immediately.

### Self-repair and background research

When a subsystem keeps failing after its own retries — speech recognition or spoken replies — Jarvis runs a self-repair pass in the background. It searches the public web with DuckDuckGo, reads up to two public pages, and sends a redacted problem description plus those excerpts to OpenRouter for a diagnosis. No window opens for this research, and nothing on screen changes while it happens.

Repairs are limited to a fixed allowlist of reversible, Jarvis-internal actions: reload the speech model, reset the microphone choice, switch recognition engine, restart the local voice worker, and restart listening. Jarvis will not edit files, run shell commands, install or download software, change Windows or security settings, or handle credentials on its own. If the fix needs you, it lists at most three concrete steps instead. If a repair genuinely requires the mouse or keyboard, Jarvis shows a **Hands off the screen** alert and asks you to leave the computer alone until it finishes.

Each subsystem gets at most three research passes per hour, and a fault that was just researched is retried quietly for two minutes before it earns another pass. If the cloud planner cannot answer with a valid plan — a weak or offline model — known faults (a missing, blocked, or disabled microphone, and a recognizer that will not start) still get a deterministic diagnosis and repair from a built-in table, so the research is never wasted entirely. Diagnostic reports are scrubbed of API keys, tokens, email addresses, and your Windows user name before they leave this PC.

### Google Calendar and Gmail

The app publisher configures a Google **Desktop** OAuth client once for the installer. The build reads its public client ID from `JARVIS_GOOGLE_CLIENT_ID`; the installed app uses PKCE and does not require users to paste a client ID or client secret. To build a release, create a Desktop OAuth client in the Google Cloud project, enable Calendar and Gmail APIs, configure the consent screen, then set the client ID once in the build environment (PowerShell: `$env:JARVIS_GOOGLE_CLIENT_ID = 'your-desktop-client-id'`) before `npm run desktop:build`. Each user then chooses **Connect Google** in Settings and grants access in their browser. If the app is in testing mode, add the account as a test user.

Jarvis requests Calendar event access, Gmail read/send access, and Google Drive read access plus permission to create Drive files. Reconnect Google after upgrading from an earlier version so the new Drive permission is granted. Google asks you to grant those scopes during account linking. The app encrypts refresh tokens in your OS app-data folder (`%LOCALAPPDATA%\Jarvis MK1` on Windows); generated credentials are not stored in the project folder.

### Built-in apps and Commands

Open **Apps** to land on **My Apps** with your installed tools such as Calculator, Notes, Browser, Weather, Files, Calendar, Mail, the Command Library, Local Tools, and Voice Studio. The redesigned **App Store** offers 16 entries with one-click **Add to My Apps**, **Open** and **Remove** actions; deleted entries can be added again at any time. The Apps page has a Dashboard exit button, and every app panel has its own minimize and close buttons. Browser includes DuckDuckGo search; sites such as Google and YouTube open in an isolated Electron browser view when their iframe policies block the embedded frame. Use the full-page button for other sites that refuse frames; while that native view is open, **← Exit full page** in the browser toolbar returns to the app. Notes are stored in this app's local browser storage. Files can browse local drives and network folders accessible to the signed-in Windows account, search a selected location, and preview UTF-8 text locally. File names and contents stay local unless you choose **Share with Jarvis**. The Files preview can also produce local key points, likely action items, and key terms without sending a file to an AI service. Jarvis can open documents and launch selected `.exe` apps. With **Settings → Local tools → Act without asking** enabled (the default), opening folders and files, launching apps, browsing drives, searching files, and volume or window shortcuts happen immediately; file writes and shell commands still show their exact target or command before you confirm. These actions run as the current Windows user and do not elevate to administrator. The TypeScript engine improves command search with typo-tolerant matching and suggests likely corrections for uncertain speech results. Weather uses Open-Meteo. The Commands page contains 2,500 searchable prompt examples across security, programming, data, home, travel, learning, accessibility, and other topics. Use **Settings → Connections → Open .env file** to open the local configuration file in Notepad.

Jarvis bundles the official Windows x64 Python embeddable runtime for its local file analyzer, so installed copies do not depend on a separate Python installation. The build pins the Python release and verifies its SHA-256 checksum before staging it. The analyzer uses only Python's standard library; Jarvis still falls back to its equivalent TypeScript analyzer if the bundled runtime cannot start. The separate local voice engine installs its Python and model dependencies only after the user starts setup.

**Local tools:** SQL features use SQLite in the bundled Node.js runtime to store user-approved memories on this PC. Jarvis only searches those memories when asked to recall them, and it will not save obvious passwords, API keys, tokens, or payment details. If enabled in Settings, only memories categorized as “Preferences” are sent with chat requests so Jarvis can apply relevant saved preferences; other memories remain local unless explicitly recalled. Set preferences by asking Jarvis, for example “turn serious mode on,” “call me Alex,” “keep answers concise,” or “use saved preferences only when I ask.” Windows PowerShell commands run as the signed-in user, require an explicit review click, and never elevate.

### Coding workspace

**Apps → Code** supports creating, browsing, editing, and saving text/source files for JavaScript, TypeScript, Python, C#, Go, Rust, C, C++, SQL, PowerShell, HTML, CSS, Bash, Elixir, and Java. Select a project folder with the Windows folder picker; the workspace only accepts relative file paths beneath that selected folder, skips symbolic links and common dependency/build directories, and limits editable files to 1 MB. Saving is a deliberate action and shows the target path. Quick checks report text-file basics and validate JSON syntax without executing code. **Ask Jarvis to review** shares the current file with OpenRouter only after a separate confirmation.

This is an editing and AI-review workspace, not an IDE/compiler: it does not build or run project code and does not install the language toolchains. This keeps the installer lean and avoids silently running code with access to your Windows account. Use your own development tools to build or run saved files, and review their permissions and behavior first.

## Privacy and approvals

- An optional Fish Audio key entered in Settings is stored in local per-user configuration; the browser never receives or reads its saved value. Fish Audio processes text for spoken replies.
- The local voice runtime and pinned voice-model files are downloaded only after choosing **Set up local voice clone**. Large model weights are SHA-256 verified before use.
- Messages you send to Jarvis are processed by OpenRouter. Your message, conversation history, submitted screen captures, and any Google data you explicitly allow are sent to OpenRouter and handled under its account/provider policies. Google results are shared only when **Allow Calendar and Gmail results in AI conversations** is enabled in Settings.
- When you ask Jarvis to research something, or ask anything time-sensitive, your query goes to DuckDuckGo and Jarvis reads up to three of the strongest public pages. Result titles, snippets, extracted page text, and the URLs are sent to OpenRouter as context. Page scripts are not run, and private/local addresses, non-HTTPS URLs, and redirects to them are blocked.
- When you use the built-in Browser, the selected website receives its ordinary web request from your device. The embedded iframe is sandboxed and does not send a referrer. Sites that block iframes, including Google and YouTube, use a separate isolated Electron browser view inside the Browser panel; the site is not given Jarvis's local command permissions. Some sites may still restrict embedded app views or sign-in.
- Coding workspace file names and contents stay on this PC unless you explicitly choose **Ask Jarvis to review** and confirm; then the selected file contents and question are sent to OpenRouter. The workspace does not compile or execute those files.
- Image prompts you ask Jarvis to create go to Pollinations, a free no-account image service, and fall back to AI Horde's community workers if Pollinations fails. Both receive the prompt; treat prompts as public. Saving an image writes it only to your own `Downloads` folder on this PC.
- Voice input uses the engine chosen in **Settings → Voice & behavior**. The desktop app transcribes locally with the bundled Vosk model. In a web browser Jarvis may use the built-in `SpeechRecognition` service, which can send microphone audio to its provider; it switches to the local model when that service is unavailable. Manual dictation accumulates recognized phrases across pauses and sends after about 2.8 seconds of silence; use **Stop listening** to submit sooner. The microphone opens only for one question per hotkey press. Low-confidence results are shown for review. When voice replies are enabled, short answers are sent to Fish Audio, with up to three attempts if speech synthesis or playback fails; Jarvis never falls back to computer-generated speech. The assistant is instructed to review its reasoning and correct errors it notices, but no AI system can guarantee error-free answers.
- Local document insights run in the TypeScript engine or optional Python helper. The selected text stays on this device and does not go to OpenRouter unless you separately choose **Share with Jarvis**.
- When Jarvis self-repairs, it sends a redacted failure description to DuckDuckGo (as a search query) and reads up to two public pages. That description and the page excerpts are sent to OpenRouter for a repair plan. Keys, tokens, email addresses, and your Windows user name are removed first. Only reversible Jarvis-internal repairs are applied automatically; anything else is reported to you as steps.
- A screen capture is sent to the active AI provider only after you select a screen and submit your question. Image understanding requires a vision-capable model; OpenRouter routes the image according to the selected model/provider.
- Calendar events and email messages are never created or sent by a model tool call. Jarvis prepares a review card and waits for you to confirm. You can say or type **confirm all** to approve the displayed Gmail and Calendar edits/deletions together; creating calendar events and sending email still require individual review.
- Jarvis can browse and search locations accessible to the signed-in Windows account, preview local UTF-8 text, open files and folders, launch selected apps, control volume, send common keyboard shortcuts to the active window, and collect basic CPU/RAM/disk telemetry. The app verifies that it is not elevated before local computer actions; protected paths remain protected by Windows.
- Safe computer actions run immediately while autonomy is on: opening folders and documents, launching apps, browsing, file search, telemetry, volume, keyboard shortcuts, screen capture, and mouse/keyboard takeover. File writes and PowerShell commands still show their exact path or command and wait for a separate user click in the default **Safe** access level. Choosing **Full** access runs writes, commands and batch writes immediately as well; Windows system folders, disk and boot commands, and security policy stays blocked either way. Turn autonomy off in **Settings → Local tools** to review every computer action. Shell execution has a 30-second limit, runs as the current user, and does not persistently change PowerShell's execution policy.
- Jarvis can suggest a click on a monitor capture, but the user must review the marked screenshot and confirm the click. Screen clicks require one connected monitor.
- Settings, Jarvis page navigation, chat visibility, and widget removal are reversible in-app actions. Custom widgets are stored locally in the browser profile.
- Disconnect Google in Privacy settings to revoke the token and remove the local encrypted token file.

Google OAuth apps using Gmail scopes may need to be kept in testing mode with approved test users until the project completes Google’s verification requirements.
