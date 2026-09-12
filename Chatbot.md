# Building a Free, Serverless AI Chatbot for a Static Portfolio Site

A from-zero, copy-paste-friendly guide: add a real AI assistant (with live
web search) to a static GitHub Pages portfolio — for $0/month, no server to
maintain, no model to host, secrets kept out of the browser.

![Architecture Diagram](./portfoliochatbot.png)

> This guide is written to be reused for teaching. Swap every placeholder
> (`your-repo`, `your-worker-name`, `YOUR_USERNAME`) with your own values.
> Never commit real API keys into any repo, public or private.

---

## 1. What we're building

A static site (HTML/CSS/JS, hosted free on GitHub Pages) gets a chat
widget. The widget doesn't run any AI itself — it calls a small backend
function that:

1. Receives the visitor's question
2. Adds your bio/facts as context
3. Optionally runs a live web search if the question needs current info
4. Sends everything to a fast, free LLM API
5. Returns the answer to the chat widget
6. Falls back to simple keyword matching if the LLM call fails, so the
   bot never goes fully silent

No model weights are downloaded or hosted anywhere. No server needs to
stay running. No PC needs to stay on once deployed.

### Why not just host a model?

- Model files are tens–hundreds of MB — awkward in a git repo, slow to load
- Free hosted inference endpoints often sleep after inactivity, causing
  cold-start delays that look like the bot is broken
- Access tokens on some platforms expire and need manual renewal
- A small self-hosted model is far weaker than a free-tier hosted LLM API
  anyway — you gain nothing for the extra complexity

---

## 2. The services used (real links)

| Service | Role | Link |
|---|---|---|
| GitHub Pages | Hosts the static frontend | https://pages.github.com |
| Cloudflare Workers | Runs the backend function, holds secrets | https://developers.cloudflare.com/workers |
| Wrangler | CLI that deploys to Cloudflare Workers | https://developers.cloudflare.com/workers/wrangler |
| Groq | Free-tier hosted LLM API | https://console.groq.com |
| Tavily | Free-tier web search API built for LLMs | https://tavily.com |
| Node.js | JS runtime needed to run the CLI locally | https://nodejs.org |

All of the above have free tiers sufficient for a personal portfolio bot
as of this writing — check each provider's current pricing page before
relying on it, since free-tier terms change over time.

---

## 3. Architecture

See `portfoliochatbot.png` above for the visual, or `architecture.drawio`
(editable at https://app.diagrams.net) for the source diagram.

```
Visitor's Browser
      │  loads static files
      ▼
GitHub Pages  (index.html, CSS, JS)
      │  fetch() POST on chat message
      ▼
Cloudflare Worker  (worker.js — backend logic + secrets)
      ├── static bio/facts (hardcoded context)
      ├── IF question needs current info → Tavily web search
      ├── → Groq LLM call with all context
      └── IF Groq call fails → keyword-matching fallback
      ▼
Answer returned to the chat widget
```

---

## 4. Repository structure

```
your-portfolio-repo/            (public — the actual site)
├── index.html
└── assets/...

your-backend-repo/               (private recommended)
├── worker.js                    (the serverless function)
├── wrangler.toml                 (deployment config)
├── README.md                     (this file)
├── architecture.drawio           (editable diagram source)
└── portfoliochatbot.png          (diagram image, referenced above)
```

Keep these as two separate repos/folders. The frontend repo needs no
secrets and can be fully public. The backend code also contains no
secrets (keys are stored separately as encrypted platform secrets, never
in the file) — many people still keep it private simply to avoid exposing
personal bio content or making their exact setup trivially cloneable.

---

## 5. Prerequisites

- A code editor (VS Code or similar)
- [Node.js](https://nodejs.org) installed (LTS version) — verify with:

```bash
node -v
npm -v
```

- A free [Cloudflare](https://dash.cloudflare.com/sign-up) account
- A free [Groq](https://console.groq.com) account + API key
- A free [Tavily](https://tavily.com) account + API key

---

## 6. Step-by-step setup (copy-paste commands)

### 6.1 Install the Cloudflare CLI

```bash
npm install -g wrangler
```

### 6.2 Log the CLI into your Cloudflare account

```bash
wrangler login
```

This opens your browser to authorize the CLI — approve it, then return to
the terminal.

### 6.3 Create your project folder

```bash
mkdir portfolio-bot
cd portfolio-bot
```

Place `worker.js` and `wrangler.toml` inside this folder.

### 6.4 Store your API keys as encrypted secrets

Never hard-code keys in `worker.js`. Store them via the CLI instead:

```bash
wrangler secret put GROQ_API_KEY
```
*(paste your Groq key when prompted, press Enter)*

```bash
wrangler secret put TAVILY_API_KEY
```
*(paste your Tavily key when prompted, press Enter)*

### 6.5 Deploy

```bash
wrangler deploy
```

This prints a live URL, e.g.:
```
https://your-worker-name.your-subdomain.workers.dev
```
Copy this — it's your `BOT_API_URL`.

### 6.6 Wire the frontend to the backend

In your site's `index.html`, inside the chat widget's `<script>` tag, set:

```javascript
const BOT_API_URL = "https://your-worker-name.your-subdomain.workers.dev";
```

Your chat widget's send function should `fetch()` this URL with the
visitor's message and render whatever `reply` comes back — no other
frontend changes are needed for anything that happens on the backend.

### 6.7 Verify secrets were saved

```bash
wrangler secret list
```

Should list `GROQ_API_KEY` and `TAVILY_API_KEY` (values hidden).

### 6.8 Watch live logs while testing

```bash
wrangler tail
```

Leave this running in a terminal, then use the chat widget on your live
site in another window. Any backend error (bad API key, rate limit,
etc.) will print here in real time — this is the only way to see backend
errors, since they never reach the browser console.

### 6.9 Redeploying after any code change

Every time you edit `worker.js`:

```bash
wrangler deploy
```

Secrets don't need to be re-entered on redeploy — only code changes.

---

## 7. Security checklist

- [ ] API keys are stored only via `wrangler secret put`, never typed into
      `worker.js` directly
- [ ] CORS in `worker.js` (`ALLOWED_ORIGIN`) is locked to your exact site
      URL, not `*`
- [ ] Input length is validated/capped before being sent to the LLM
- [ ] The bot's system prompt explicitly instructs it to say "not sure"
      rather than invent facts
- [ ] `.env` / `.dev.vars` files (if used for local testing) are listed in
      `.gitignore` and never committed

---

## 8. Common errors and fixes

| Symptom | Cause | Fix |
|---|---|---|
| `node` not recognized | Node.js not installed, or terminal opened before install | Install from nodejs.org, open a **fresh** terminal |
| `respond is not defined` | Old and new `<script>` code left in the same file | Delete the entire old block, don't append new code alongside it |
| `Cannot set properties of undefined` | A helper function doesn't `return` its result | Ensure `addMessage()` (or similar) returns the created element |
| Bot always says "trouble reaching my brain" with no console error | Backend error is being swallowed silently | Run `wrangler tail` while reproducing — backend errors never reach the browser console unless logged |
| `401 Invalid API Key` | Key mistyped, or revoked/rotated on the provider's dashboard | Generate a fresh key, `wrangler secret put` again, redeploy |
| `429 rate_limit_exceeded` | Free-tier daily token quota reached | Wait for it to roll off (see the error message's retry time), or switch to a lighter model with a separate quota, or add a fallback (see §9) |

---

## 9. Keeping the bot useful even when the LLM quota runs out

Free-tier LLM APIs have daily token limits. Rather than showing visitors
an error when the limit is hit, keep a small static keyword-matching
fallback in `worker.js`:

1. Try the LLM call first (always the priority).
2. If it fails for any reason (`429`, outage, bad key), fall back to
   matching the visitor's message against a small hardcoded
   `KNOWLEDGE_BASE` object of your key facts.
3. If no keyword matches either, return a friendly "running on backup
   mode" message pointing to the site's own sections/buttons.

This requires zero frontend changes — the frontend only ever sees a
`reply` string and has no idea whether it came from the LLM or the
fallback.

---

## 10. Mistakes made along the way (real debugging log)

Documenting actual failure modes, not a "how it should go" fantasy.

**Confusing the terminal prompt with the command itself.**
Pasting `C:\Users\you>node -v` instead of just `node -v`. The prompt text
shows *where* you are — it is never part of what you type.

**Runtime not installed yet.**
Running the CLI before Node.js was installed. Fix: install from
[nodejs.org](https://nodejs.org), then verify in a fresh terminal window
(old windows don't pick up newly installed programs).

**Confusing two separate projects.**
Assuming the backend folder had to live inside the site's GitHub repo, or
that deploying it required downloading the site's repo first. They're
unrelated — the site repo only ever holds frontend files; the backend
folder deploys independently via the CLI and never touches GitHub.

**Leaving old code next to new code.**
Replacing a hardcoded response function with a real API call, but not
deleting the old function — JavaScript's last-declared function wins, so
the *old, broken* version silently overrode the new one. Symptom:
`X is not defined`, pointing at code that looked deleted. Fix: delete the
entire old block, don't append new code beside it.

**A helper function not returning its result.**
A function creating a chat bubble needs to `return` that element if other
code updates it later (e.g. showing "thinking..." then swapping in the
real answer). Omitting the `return` caused a
"cannot set property on undefined" error two functions downstream — a
reminder that JS errors often surface far from their real cause.

**A failure hidden by a friendly fallback message.**
The backend caught API failures and returned a generic "having trouble"
message with a normal success status — good for visitors, bad for
debugging, since **no error ever reached the browser console**. The real
cause only became visible via `wrangler tail` while reproducing it.

**An API key that worked, then suddenly didn't.**
After several successful calls, the same key started failing with
"invalid API key" — provider-side (superseded/flagged), not a typo.
Confirmed by checking the provider's dashboard directly rather than
re-debugging the deployed code.

**Hitting the free daily token quota mid-testing.**
Heavy back-and-forth testing burned through the day's free LLM quota,
surfacing as `429 rate_limit_exceeded` in `wrangler tail` logs. Solved by
switching to a lighter model with a separate quota pool, trimming
`max_tokens` per response, and adding the keyword-matching fallback in
§9 so visitors are never left with a dead end.

### How AI assistance was used

An AI assistant (Claude) was used throughout as a **pair-debugger**:

- Proposed the architecture (serverless function + hosted LLM + hosted
  search, instead of self-hosting a model) once the constraints were
  explained (free, no localhost, portfolio use case)
- Generated the initial backend code and explained each function's role
- When errors were pasted in verbatim, explained *why* each occurred
  (e.g. JS's last-declared-function-wins behavior) rather than just
  supplying a fix, so the concept transferred
- Added diagnostic logging specifically because the original design's
  silent error-catching made real failures invisible
- Distinguished "your code has a bug" from "a third-party service's state
  changed," which determined where to actually look
- Designed the keyword-matching fallback once quota exhaustion became a
  real, observed failure mode — not preemptively, but in response to an
  actual `wrangler tail` log showing it happening

**The pattern worth teaching:** paste the *exact* error text (console
errors, terminal output, log lines) rather than a paraphrase — precise
error messages are what make targeted debugging possible instead of
guesswork.

---

## 11. How to build your own version

1. Have a static portfolio already (plain HTML/CSS/JS is enough).
2. Write down 10–20 facts about yourself/your work for the bot to know.
3. Get free API keys: [Groq](https://console.groq.com),
   [Tavily](https://tavily.com), [Cloudflare](https://dash.cloudflare.com/sign-up).
4. Write the backend function (facts + optional live search + LLM call +
   keyword fallback). Get "it responds at all" working before adding
   search.
5. Follow §6 above to deploy, store secrets, and wire up the frontend.
6. Test with `wrangler tail` open. When something fails silently, that's
   exactly what it's for.
7. Iterate: refine the bot's instructions, expand its facts, add rate
   limiting.

---

## 12. License / usage note

Written to be reused for teaching. Replace all placeholders with your own
values before using; never commit real keys or personal URLs into a
public copy of this guide.

---

## 13. Author

**Showkot Hosen**
XAI in Cybersecurity Researcher

- MSc in Electronics and Telecommunication Engineering (AI & Cybersecurity), Chittagong University of Engineering and Technology (CUET) — *in progress*
- BSc in Electronics and Telecommunication Engineering, CUET — CGPA 3.63/4.00
- (ISC)² Certified in Cybersecurity (CC), Cisco Ethical Hacker
- Invited Reviewer, *Cyber Security and Applications* (Elsevier); Reviewer Panel, *HKIE Transactions*

📧 showkothosen10@gmail.com
🔗 [LinkedIn](https://www.linkedin.com/in/showkot-hosen10) · [GitHub](https://github.com/Showkot-Hosen-10) · [Portfolio](https://showkot-hosen-10.github.io/showkotsfolio)
