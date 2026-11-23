---
name: ramp-submit-plugin
area: Developers
supported_surfaces: [cli]
description: >-
  Publish a plugin for your product to the Ramp plugin marketplace, so it is
  available in Ramp products: sign up for a Developer Console
  account, save a plugin draft, request a test sandbox, submit the draft for
  review, and check the review result. Use when a developer asks to list,
  submit, or publish their MCP server or plugin on Ramp, or asks whether their
  Ramp plugin was approved.
compatibility: Requires an up-to-date Ramp CLI, browser access for sign-in, and a work email. The user must be an administrator of a Ramp Developer Console.
---

# Submit a plugin to the Ramp plugin marketplace

Take the user's product from their project to a plugin submitted to the Ramp
plugin marketplace. A plugin is built around the product's MCP server.
A plugin has a draft that you can save as often as needed. Submitting freezes the
draft and sends it to Ramp for review.

## Rules

- Run commands with `--agent`. Pass the request as `--json` and put a `rationale`
  in the body that says why you are making this call.
- Run `ramp plugins <command> --help` before the first use of a command and
  follow the fields it lists.
- Fill the draft from the user's project and what the user tells you. Never
  guess a URL, such as a privacy policy or support page. Ask for it.
- When changing an existing plugin, send only the fields that change. The rest
  of the draft is kept. To clear a field, send it as `null`, or as an empty list.
- Save only what the user asked for, and tell them what you saved. Do not edit
  the draft when the user only asked a question, such as whether it was reviewed.
- Every response returns the plugin's `revision`. Pass the latest one on the
  next save or submit. If a call says the plugin changed, read the plugin again
  and show the user what changed. Anything the user confirmed before no longer
  counts: get their confirmation again before you retry a submit.
- Submitting cannot be undone. Show the user the saved draft and get explicit
  confirmation first.
- Never ask the user for a password or sign-in code. Sign-in happens in the
  browser.

## 1. Connect to Ramp

```bash
ramp auth status --agent
```

If the user has no Ramp Developer Console account, sign up with their work
email. Personal email addresses are not accepted.

```bash
ramp auth signup --email <work email> --agent
```

Then stop and ask the user to accept the invite email and set a password. When
they have, they sign in:

```bash
ramp auth login
```

If signup says the company already has a Ramp account, the user must ask their
Ramp administrator for Developer Console access. Do not retry.

Refresh the tools and check that the `plugins` commands are listed. If they are
not, plugin publishing is not available for this account yet; tell the user and
stop.

```bash
ramp tools refresh --agent
ramp plugins --help
```

## 2. Check for an existing plugin

```bash
ramp plugins status --agent --json '{"rationale": "<why>"}'
```

If the plugin already exists, read it in full and continue from its current
state instead of creating a second one:

```bash
ramp plugins status --agent --json '{"rationale": "<why>", "plugin_id": "<plugin id>"}'
```

## 3. Fill in and save the draft

Read the user's project to find the plugin's name, what it does, the MCP server
URL, how the server authenticates (`none`, `bearer`, or `oauth`), and its OAuth
redirect URIs. Ask the user for anything you cannot find, including links to
their website, support page, privacy policy, and terms.

Add instructions a Ramp reviewer can follow to test the plugin, and a few
example prompts with the result each should produce.

Create the plugin with a `slug`. The slug is permanent, so confirm it with the
user:

```bash
ramp plugins draft --agent --json '{
  "rationale": "<why>",
  "slug": "acme-expenses",
  "name": "Acme Expenses",
  "tagline": "...",
  "description": "...",
  "mcp_url": "https://mcp.acme.com/mcp",
  "auth_type": "oauth",
  "oauth_redirect_uris": ["https://app.acme.com/oauth/callback"],
  "website_url": "https://acme.com",
  "support_url": "https://acme.com/support",
  "privacy_policy_url": "https://acme.com/privacy",
  "terms_url": "https://acme.com/terms",
  "reviewer_instructions": "..."
}'
```

To change an existing plugin, send `plugin_id` and the latest `revision`
instead of `slug`, with only the fields that change:

```bash
ramp plugins draft --agent --json '{
  "rationale": "<why>",
  "plugin_id": "<plugin id>",
  "revision": <latest revision>,
  "description": "..."
}'
```

If a save is rejected, fix the field it names and save again. Keep the
`plugin_id`, the draft's version ID, and the `revision` from the response.

## 4. Request a sandbox

The sandbox is a separate Glass test account for trying the plugin before
review. Each Developer Console administrator has one of their own, used for
all of the business's plugins. To share one with a teammate instead, its owner
invites them in Glass.

Requesting one asks Ramp staff to create it, so ask the user before you do:

```bash
ramp plugins sandbox --agent --json '{"rationale": "<why>"}'
```

- `requested`: Ramp staff create the sandbox. This is not instant. Tell the user
  to watch for an email titled "Your Ramp plugin sandbox is ready", and do not
  wait or poll.
- `completed`: give the user the login URL and these steps.
  1. Set up the login with the link in that email. The link works for 48 hours;
     after that, open the login URL and choose Reset password. If the page asks
     them to sign in instead, their email already has a Ramp login there: they
     sign in with it, or choose Reset password.
  2. Add the plugin's MCP server in Glass: Settings → Organization settings →
     Plugins, then the + button.
  3. Try the plugin the way a user would.

Testing in the sandbox is recommended before submitting, but the user decides.

## 5. Submit for review

Before submitting, check that the draft has an MCP server URL, an
authentication type, a description, and reviewer instructions. Ramp accepts an
incomplete draft, but a reviewer cannot approve one.

Show the user the saved draft. After they confirm, submit it:

```bash
ramp plugins submit --agent --json '{
  "rationale": "<why>",
  "version_id": "<draft version id>",
  "revision": <latest revision>
}'
```

Tell the user the plugin is submitted and that review is done by Ramp.

## 6. Check the review

On a later request, read the plugin again with `ramp plugins status` and its
`plugin_id`. The latest version shows the status and, once reviewed, the
reviewer's decision and feedback.

- `submitted` or `in_review`: still with Ramp. Nothing to do.
- `changes_requested`: show the feedback and ask the user how they want to
  address it. Change the draft as in step 3 only when they ask, and submit
  again only after they confirm.
- `approved` or `published`: tell the user.
- `rejected`: show the feedback. Do not resubmit without the user's direction.
