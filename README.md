# ashare plugin for Claude Code

[ashare.net](https://ashare.net) hosts the HTML and Markdown documents your agent makes and opens each one only to the people you choose: everyone on your company's email domain, specific people by email, or anyone with a password. Readers sign in with a code from their email, with no account to create. They can select text and comment; the agent picks up the comments, fixes the document, replies and resolves the threads. Every change is a new version at the same link.

The plugin gives Claude Code two things:

- the `ashare` MCP server (`https://ashare.net/mcp`) with 15 tools, listed below;
- the `ashare` skill, which walks the agent through the loop: publish, send you the link, collect comments, edit or update, reply, resolve. Before publishing it asks you who should see the document, it asks you before letting anyone in on an access request, and it knows when to offer a password, an expiry, company visibility or embedding, and how to change or remove them.

## Tools

| Tool | What it does |
|---|---|
| `publish` | Publish HTML or Markdown, or a site of several files, and get the link |
| `update` | A new version at the same link with the full new source |
| `edit` | Replace one exact fragment and publish the result as a new version |
| `get` | The source of a version, the current one by default |
| `list` | Your documents: id, version, views, title, link, expiry, password and access marks |
| `views` | Who opened a private or company document: each signed-in reader, how many times, last and first time |
| `protect` | Expiry, password, visibility, notifications, the sites that may embed the document and the version history hidden from readers |
| `share` | Add people by email to a private or company document |
| `delete` | Delete a document with all versions and comments |
| `list_comments` | Comment threads with the quoted text and the version |
| `reply` | Reply in a thread as the agent |
| `resolve` | Mark a thread resolved |
| `list_access_requests` | People who asked to open a private or company document, with their message |
| `approve_access` | Let the person in and email them the link |
| `deny_access` | Refuse an access request |

## Install

```bash
claude plugin marketplace add Magir/ashare-plugin
claude plugin install ashare@ashare
```

Or inside a session: `/plugin marketplace add Magir/ashare-plugin`, then `/plugin install ashare@ashare`.

## API key

Sign in at [ashare.net/login](https://ashare.net/login) with your email (a 6-digit code comes by mail, no password) and create a key on the account page. The key is shown once; lost it, revoke it there and create a new one. The key goes to the server in the `Authorization: Bearer` header and is never stored in the plugin. With your key the agent sees and changes only the artifacts you published.

Put it in the environment Claude Code starts from:

```bash
echo 'export ASHARE_API_KEY=<your key>' >> ~/.zshrc   # or ~/.bashrc
```

Open a new terminal and restart Claude Code. Without the variable the server answers 401 and the ashare tools don't show up.

## Check

In a new session run `/mcp`: `plugin:ashare:ashare` should be connected. Then ask "share this report" about any HTML or Markdown file, and the agent replies with an `https://ashare.net/a/...` link.

If you added ashare by hand before (`claude mcp add ... ashare`), Claude Code connects that copy instead of the plugin's, because both point at the same URL. The skill works with either. To switch to the plugin's server and the key from `ASHARE_API_KEY`, run `claude mcp remove -s user ashare`.

## Who can open it

A new document is private: only you open it, signed in on ashare.net with your email. Say who else should see it: "share it with masha@example.com" lets her in by email (she gets a 6-digit code, no account needed), "for the whole team" opens it to your organization on ashare, "make it public" opens it to anyone with the link. The agent asks before publishing and does not make a document public unless you say so.

- Company: everyone in your organization on ashare, its members and anyone who signs in with an email on its domains, plus the emails you shared it with. Create the organization on [ashare.net/me/settings](https://ashare.net/me/settings) first; without one the agent gets an error and nothing is published.
- Access requests: someone off the list who opens the link can ask you in, and you get an email. Ask the agent "who asked for access": it shows each request with its message and lets the person in only after you say yes to them. Refusing sends them nothing.
- Who opened it: "who viewed the report" lists the signed-in readers of a private or company document with how many times and when. Readers of a public link only add to the view count.

## Several files

A report with images, stylesheets or more pages goes up as a site: `index.html` plus the other files, relative links work as on disk. The agent passes them in `files`, or zips the folder and sends the archive over the REST API. Up to 200 files, 5 MB each, 25 MB in all.

## Compare versions

After a fix the agent gives you a link like `https://ashare.net/a/<id>?v=3&diff=2`: version 3 with what changed since version 2, added text in green and removed text struck out. Readers get the same from the "compare with" list next to the version.

Say "hide the history" and readers see only the current version: no version number, list, comparison or comments on older versions. You still see every version on ashare.net, signed in.

## Password, expiry and embedding

Ask for them in plain words: "share it with a password", "make the link expire in a week", "extend the link", "remove the password", "embed it in Notion". The agent passes `password` and `expires_in` (`30m`, `24h`, `7d`) to `publish` or `update`, or calls `protect` later.

- Password: each reader enters it once per browser, a cookie keeps them in for 7 days. ashare stores only a hash, so keep the password yourself; a new one logs every reader out.
- Expiry: after it the link shows "Link expired", you included. The agent keeps access through the API key and can extend or remove the expiry. 30 days after expiry the document is deleted for good with all versions and comments.
- Embedding: by default the document shows in a frame only on ashare.net. The agent can allow your sites (`https://www.notion.so`), any site, or go back to your account default from [ashare.net/me/settings](https://ashare.net/me/settings). Embed public documents without a password: readers cannot sign in inside someone else's site.

## Update and remove

```bash
claude plugin marketplace update ashare
claude plugin update ashare@ashare
claude plugin uninstall ashare@ashare
```

## Limits

- One page per version is a single file with CSS, JS and images inline, up to 5 MB; a site is up to 200 files, 5 MB each, 25 MB in all. Scripts from public CDNs work.
- A private document opens only to you and the emails you shared it with, a company one also to your organization, a public one to anyone with the link, unless it has a password. Links are unguessable and not indexed.

## Docs

Other agents (claude.ai, Codex, Cursor, Gemini CLI and more), the CLI, the REST API and every tool: [ashare.net/docs](https://ashare.net/docs).

## Privacy

What ashare stores and for how long: [ashare.net/privacy](https://ashare.net/privacy). Terms: [ashare.net/terms](https://ashare.net/terms).
