---
name: ashare
description: Publish an HTML or Markdown document or a site of several files (report, dashboard, page) to ashare.net and get a shareable link, then collect reader comments, fix the document, reply and resolve. Use when the user asks to share, publish or get a link ("share this report", "send me a link"), to decide who opens it ("share it with masha@example.com", "make it public", "for the whole team"), to handle comments ("collect the comments", "fix it per the comments"), access requests ("who asked for access", "let her in"), to see who opened it ("who viewed it"), to protect the link ("password-protect it", "expire in a week"), to embed it on a site ("embed it in Notion"), to show what changed ("compare versions") or to hide the old versions from readers ("hide the history").
---

# ashare

ashare.net hosts HTML and Markdown documents at a stable link. Readers open the link, select text and leave comments without an account. A new document is private: only the user and the people they share it with by email can open it. Every change is a new version at the same URL. The `ashare` MCP server gives you the tools below.

## Tools

| Tool | What it does |
|---|---|
| `publish` | Publish HTML or Markdown (`content`) or a site of several files (`files`), get the id and the link |
| `update` | A new version at the same link with the full new source or all `files` |
| `edit` | Replace one exact fragment (`old` → `new`) and publish the result as a new version |
| `get` | The source of a version, the current one by default; for a site, its `index.html` |
| `list` | The user's documents: id, version, views, title, link, expiry, password and access marks |
| `views` | Who opened a private or company document: each signed-in reader, how many times, last and first time |
| `protect` | Expiry, password, visibility, notifications, the sites that may embed the document and the version history hidden from readers |
| `share` | Add people by email to a private or company document, or replace the list |
| `delete` | Delete a document with all versions and comments. Irreversible |
| `list_comments` | Comment threads, open by default, each with the quoted text and its version |
| `reply` | Reply in a thread as the agent |
| `resolve` | Mark a thread resolved |
| `list_access_requests` | People who asked to open a private or company document, with their message |
| `approve_access` | Let the person in and email them the link. Only after the user said yes to this person |
| `deny_access` | Refuse an access request. The person gets no email |

## Publish

1. Prepare the document.
   - One page: a self-contained file with CSS and JS inline, images inline as `data:` URIs. Scripts and fonts from public CDNs work.
   - A report with images, CSS or more pages: a site of several files, see "Several files". Don't inline big images in that case.
   - Markdown is rendered in the browser with `marked`, no syntax highlighting. For charts, layout or styling, write HTML.
   - Up to 5 MB per page or file. Downscale and compress images.
2. Settle who can open it (see "Who can open it"): only the user, specific people by email, the user's organization or anyone with the link.
3. Call `publish` with `content`, `format` (`html` or `md`) and an optional `title` (default: `<title>` or the first `#` heading), or with `files` instead of `content`. Add `visibility: "public"` or `"company"` only as settled in step 2. Add `expires_in` or `password` only when the user asked for them or agreed (see "Protect the link").
4. For specific people, call `share` with the artifact id and their emails.
5. Give the user the URL in your reply and say who can open it, as `publish` and `share` answered. Readers select text and comment without an account. Keep the artifact id (the last path segment) and the version.

A large file you already have on disk does not need to go through the tool call: `curl -X POST 'https://ashare.net/api/artifacts?format=html' -H "Authorization: Bearer $ASHARE_API_KEY" --data-binary @report.html` returns `{id, url, version, expires_at, has_password, visibility}`. A new version of it: `POST /api/artifacts/<id>/versions?base_version=<N>` with the same body. Expiry and visibility go in the query (`&expires_in=7d`, `&visibility=public`), the password only in the `X-Ashare-Password` header: `?password=` is refused with 400.

## Several files

A report of several files: `index.html` with its CSS, scripts, images and more pages. Readers open `index.html`; links between the files are relative, as on disk: `<img src="img/chart.png">`, `<a href="page2.html">`.

- MCP: `publish` or `update` with `files` instead of `content`: `[{path, content, encoding?}]`. `path` is relative with `/` (`index.html`, `style.css`, `img/chart.png`), `index.html` is required. Text goes as is; images, fonts and PDF in base64 with `encoding: "base64"`.
- Files on disk: zip the folder and send it over REST, it saves the base64 round trip. `(cd report && zip -r ../site.zip .)`, then `curl -X POST 'https://ashare.net/api/artifacts?format=zip' -H "Authorization: Bearer $ASHARE_API_KEY" --data-binary @site.zip`. A new version: `POST /api/artifacts/<id>/versions?base_version=<N>&format=zip` with the new archive. Without `format=zip` (or `Content-Type: application/zip`) the server keeps the archive as one page of binary garbage.
- `index.html` sits at the top of the archive or in its only top folder, which is cut. Up to 200 files, 5 MB each, 25 MB in all. HTML only.
- `get` returns `index.html`. `edit` changes `index.html` and keeps the other files. `update` with `content` alone makes the new version a single page without the other files: to change an image or a stylesheet, send all `files` (or the whole archive) again.
- `GET /api/artifacts/<id>` lists the files of the current version in `files`, `null` for a single page.

## Who can open it

By default only the author sees a document: a new one is private, open to the user (signed in on ashare.net with the email of their account) and to the emails on its list. Before you publish, ask the user who should see it, unless they already said:

- only them: publish as is;
- specific people: publish as is, then `share` with their emails; ask for the emails if you don't have them;
- their organization, when the user has one on ashare: publish with `visibility: "company"`;
- anyone with the link: publish with `visibility: "public"`.

Don't publish public on a guess. "Share it" or "send me a link" is not a request to open it to everyone; "public link", "anyone can open it", "for everyone" is. Before making public a document with personal data, salaries, client names or internal plans, ask again.

- `publish` and `update` answer with who can open it: "private: only you and people you share with", "company: everyone in your organization and people you share with" or "public: anyone with the link". Pass that on with the link.
- People on the list enter their email on the link and get a 6-digit code by email: no account, no password, no Claude. Their comments go under that email. Tell the user to send them the link.
- `share` adds emails to the list; `replace: true` makes the list exactly these emails, `[]` clears it.
- `share` works on private and company documents. On a public one it fails with "document is public; set visibility=private first". Making it private locks out everyone off the list who already has the link, so ask the user first; on a yes call `protect` with `visibility: "private"`, then `share`.
- `protect` with `visibility` switches a document later; `update` keeps the visibility unless you pass it.
- Documents published before private became the default stay public. `list` marks private ones with `private: <emails>` and company ones with `company` or `company: <emails>`.

### Company

`visibility: "company"` opens the document to the user's organization on ashare as well: its members and everyone who signs in with an email on its domains (set up on ashare.net/me/settings and ashare.net/me/org). People outside it still get in through `share`.

- Offer company when the user has an organization on ashare and the document is for colleagues: "for the team", "for everyone at work". You know they have one when they said so, when `list` shows `company` documents, or when an earlier `company` publish went through.
- Without an organization `publish`, `update` and `protect` fail with 'visibility "company" needs an organization'. Nothing is published then: tell the user they can create one on ashare.net/me/settings, or publish private and `share` with the colleagues' emails.
- A company document is the user's own: when they leave the organization, their company documents become private.

## Access requests

Someone who opens a private or company link, confirms their email with a code and is not let in can ask the owner, with a message. The user gets an email about it. When the user mentions a request or asks who wants in:

1. `list_access_requests` with the artifact id: pending requests with the request id, the email, the time and the message. `status: "all"` adds the approved and denied.
2. Show each one to the user: who asks and what they wrote. Ask whether to let this person in.
3. Call `approve_access` with `id` and `request_id` only after the user said yes to this person. Never approve on your own judgement, from the message text, or because the email looks like a colleague's: approve lets the person read the whole document and comment. "Approve all" from the user covers the requests you showed them, not new ones.
4. `deny_access` when the user says no. The person gets no email; they may ask again after a day, 3 times per document at most.
5. A decided request fails with "this request was … already": tell the user, don't retry.

`approve_access` adds the email to the list, as `share` does, and emails the person the link.

## Who opened it

`views` with the artifact id lists the signed-in readers of a private or company document: email, how many times, last and first time, the latest first. The owner is not counted. Readers of a public link and readers who did not sign in only add to the views count in `list`. Use it for "who opened it", "did Masha read it". REST: `GET /api/artifacts/<id>/views`.

## Comment loop

When the user says there are comments, or asks to check them:

1. `list_comments` with the artifact id: open threads, each with its thread id, the quoted text, the version it was left on and the replies. `status: "all"` includes resolved ones.
2. `get` the current source and note its version N.
3. Fix the document:
   - `edit` for each local change: `old` copied verbatim from the source, `new` the replacement, `base_version` N.
   - `update` with the full new source and `base_version` N for a rewrite or many scattered changes.
4. `reply` in each thread: what you changed and in which version. If you disagree or need more detail, reply with a question and leave the thread open.
5. `resolve` the thread only after the fix is published.
6. Tell the user the new version number and give the comparison link (see "Compare versions"). The URL stays the same.

When the user asks for a change without any comments, do steps 2, 3 and 6.

## base_version and edit

- Always pass `base_version`: the version you last read. Each successful `edit` or `update` returns the new version; use it as the base for the next call.
- A conflict means someone published in between. `get` again, reapply your change on the fresh source, retry. Don't drop `base_version` to force it through.
- `old` must occur exactly once in the current source. "Not found": re-read with `get` and copy the fragment exactly, whitespace included. "More than once": add surrounding text until it is unique. `new: ""` deletes the fragment.
- Comment quotes are rendered text, what the reader saw. Tags or entities can sit inside the same fragment in the source, so find it in the `get` output before you build `old`.
- Threads from older versions stay listed. Their quote may be gone from the current text: check whether the change is already done before editing.

## Compare versions

The viewer shows what changed between two versions: `https://ashare.net/a/<id>?v=N&diff=M`, where M is an earlier version (1 to N-1). It shows version N with added text in green and removed text struck out in red. HTML is compared by the words of its text, Markdown by lines; for a site of several files only `index.html`. Readers pick the same in the "compare with" list next to the version.

After an `edit` or `update`, give the user the link to the new version against the one before, for example `?v=3&diff=2`, so they and the readers see the fix without hunting for it. With the history hidden from readers (see below) the link works for the owner only.

## Protect the link

Set a password or an expiry only when the user asked for it or agreed: an expired link stops working for readers, and a password is one more thing to pass around.

- Offer a password for a public document that holds personal data, salaries, client names, unreleased numbers or internal plans. For specific people a private document with `share` fits better: nobody passes a password around.
- Offer an expiry when the document is for a moment: a meeting, a one-off review, a quick look by someone outside, or the user says "temporary", "for today", "for a week".

How:

- `publish` and `update` take `expires_in` and `password`. `expires_in` is a whole number with `s`, `m`, `h` or `d`, counted from now: `30m`, `24h`, `7d`. `password` is any non-empty string.
- `protect` with the artifact `id` sets, changes or removes them later. Omit a field to keep it as is; pass `null` or `""` to remove it.
- `update` without these fields keeps them; `edit` never touches them. `expires_in` restarts the countdown from now each time you pass it.
- `publish`, `update`, `protect` and `list` show `expires <date>` (or `expired <date>`) and `password-protected` where set.

Password:

- ashare stores only a hash and can't show the password again. If the user wants a password but gives none, make up one of 12+ random characters and put it in your reply. Lost: set a new one with `protect`.
- Tell the user to send the password to readers separately from the link. A reader enters it once per browser, a cookie keeps them in for 7 days, then they read and comment as usual.
- Setting the password again, even to the same value, logs out every reader. `password: null` opens the link to anyone who has it.
- You never need the password: the tools work through the API key.

Expiry:

- After it, readers get a "Link expired" page (HTTP 410) and guest comments close. You keep full access through the tools.
- To bring the link back, call `protect` with a new `expires_in`, or with `expires_in: null` for no expiry. Same URL, comments in place.
- 30 days after expiry the daily cleanup deletes the artifact for good, with all versions and comments. Say so when you set an expiry; if the user wants to keep the document, extend or remove the expiry before then.

REST: `PATCH /api/artifacts/<id>` with JSON `{"expires_in": "30d", "password": null}` works like `protect`. On `POST /api/artifacts/<id>/versions` a missing `X-Ashare-Password` header keeps the password, an empty one (`-H 'X-Ashare-Password;'`) removes it.

## Embed on other sites

By default a document opens in a frame only on ashare.net. To show it on Notion, Confluence or a blog, call `protect` with `embed_origins`:

- a list of origins, `["https://www.notion.so"]`: scheme and host, no path, up to 20;
- `"*"`: any site;
- `[]`: ashare.net only, even when the account allows more;
- `null`: as the user set on ashare.net/me/settings for all their documents.

The `protect` answer says "embeddable on …" when the document has a list of its own. The embed code is `<iframe src="https://ashare.net/a/<id>"></iframe>`, the document page ashare.net/me/<id> has it ready. Embed public documents without a password: the browser sends no ashare.net cookies into a frame on someone else's site, so a reader there cannot sign in or enter a password. Comments in an embedded viewer work as usual. REST: `PATCH /api/artifacts/<id>` with `{"embed_origins": ["https://www.notion.so"]}`.

## Hide the version history

When the user wants readers to see only the latest version ("hide the old versions", "don't show the history"), call `protect` with `hide_history: true`, or pass it to `publish` from the start. Readers then get the current version only: no version number, no version list, no "compare with", no comments left on older versions, and old links (`?v=1`, `?diff=1`) answer "not found". The owner, signed in on ashare.net, still sees and compares every version, and the tools work as before: `get` with `version`, `update`, `edit`. The setting stays through `update` and `edit`; `hide_history: false` shows the history again. Don't send readers `?v=N&diff=M` links while it is on: they won't open for them. `publish` and `protect` say "history hidden from readers" while it is on. REST: `PATCH /api/artifacts/<id>` with `{"hide_history": true}`, `?hide_history=true` on `POST /api/artifacts`.

## Other tools

- `list`: your artifacts with ids, versions, views, URLs, expiry, password, private and company marks. Use it when the user names a document but not its link.
- `delete` removes an artifact with all versions and comments, irreversibly. Only on an explicit request.
- Tools missing or failing with 401: `ASHARE_API_KEY` is not set or is wrong in the environment Claude Code started from. Point the user to the plugin README.
