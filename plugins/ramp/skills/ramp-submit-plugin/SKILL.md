---
name: ramp-submit-plugin
area: Developers
supported_surfaces: [cli, mcp]
description: |-
  Publish a plugin for your product to the Ramp plugin marketplace, so it is
  available in Ramp products: sign up for a Developer Console
  account, save a plugin draft, request a test sandbox, add a demo video, submit
  the draft for review, and check the review result. Use when a developer asks to list,
  submit, or publish their MCP server or plugin on Ramp, or asks whether their
  Ramp plugin was approved.
compatibility: Requires an up-to-date Ramp CLI or a Ramp MCP connection, browser access for sign-in, and a work email. The user must be an administrator of a Ramp Developer Console.
---

# Submit a plugin to the Ramp plugin marketplace

Take the user's product from their project to a plugin submitted to the Ramp
plugin marketplace. A plugin is built around the product's MCP server.
A plugin has a draft that you can save as often as needed. Submitting freezes the
draft and sends it to Ramp for review.

## Surfaces

Each step shows the Ramp MCP tool and the Ramp CLI command. They do the same
thing and take the same fields. Use the one you have.

- **MCP:** call the tool by its name with the fields as arguments. The tool's
  description and input schema list the fields.
- **CLI:** run the command with `--agent` and pass the fields as `--json`. Run
  `ramp plugins <command> --help` before the first use of a command and follow
  the fields it lists.

## Rules

- Put a `rationale` in every call that says why you are making it.
- Fill the draft from the user's project and what the user tells you. Never
  guess a URL, such as a privacy policy, support page or demo video. Ask for it.
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

The user needs a Ramp Developer Console account. Signing up takes a work email.
Personal email addresses are not accepted.

**MCP:** the user signs in when they connect Ramp MCP. If they have no Developer
Console account, they sign up on the Ramp Developer Console website first, or
with the Ramp CLI as below. Check that the four tools in this skill are
available. If they are not, plugin publishing is not available for this account
yet; tell the user and stop.

**CLI:**

```bash
ramp auth status --agent
```

If the user has no account, sign up:

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

MCP:

```text
ramp_get_plugin_marketplace_status(rationale="<why>")
```

CLI:

```bash
ramp plugins status --agent --json '{"rationale": "<why>"}'
```

If the plugin already exists, read it in full and continue from its current
state instead of creating a second one.

MCP:

```text
ramp_get_plugin_marketplace_status(rationale="<why>", plugin_id="<plugin id>")
```

CLI:

```bash
ramp plugins status --agent --json '{"rationale": "<why>", "plugin_id": "<plugin id>"}'
```

In the result, each plugin's ID is `plugin_id` and each version's ID is
`version_id`. Every row of `versions` also carries the plugin's current
`revision`.

## 3. Fill in and save the draft

Read the user's project to find the plugin's name, what it does, the MCP server
URL, how the server authenticates (`none`, `bearer`, or `oauth`), and its OAuth
redirect URIs. Ask the user for anything you cannot find, including links to
their website, support page, privacy policy, and terms.

Add instructions a Ramp reviewer can follow to test the plugin, and a few
example prompts with the result each should produce.

Before you create or change a draft, show the user the fields you will save and
get their confirmation. If they say no, ask why and do not save.

Create the plugin with a `slug`. The slug is permanent, so confirm it with the
user. The fields are the same on both surfaces.

MCP: call `ramp_save_plugin_marketplace_draft` with these fields as arguments.

CLI:

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
instead of `slug`, with only the fields that change.

MCP:

```text
ramp_save_plugin_marketplace_draft(rationale="<why>", plugin_id="<plugin id>", revision=<latest revision>, description="...")
```

CLI:

```bash
ramp plugins draft --agent --json '{
  "rationale": "<why>",
  "plugin_id": "<plugin id>",
  "revision": <latest revision>,
  "description": "..."
}'
```

If a save is rejected, fix the field it names and save again. Every URL field
must be a full `https://` link. Keep the plugin's `plugin_id` and `revision`,
and the draft's `version_id`, from the response.

## 4. Request a sandbox

The sandbox is a separate Glass test account for trying the plugin before
review. Each Developer Console administrator has one of their own, used for
all of the business's plugins. To share one with a teammate instead, its owner
invites them in Glass.

Requesting one asks Ramp staff to create it, so ask the user before you do.

MCP:

```text
ramp_request_plugin_marketplace_sandbox(rationale="<why>")
```

CLI:

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
  2. In a Glass chat, ask Glass to add the plugin's MCP server and give it the
     server URL. Follow the prompts to finish setup.
  3. Try the plugin the way a user would.
  4. Record a short video of the plugin working in the sandbox. Upload it
     anywhere that gives a link anyone can open, such as an unlisted YouTube
     video or a Loom.

A submission needs this demo video, so the user has to test in the sandbox
first. The link must start with `https://`. When they give you the link, show
them the plugin name and the link, and ask whether to save it on the draft. If
they say no, ask why and do not save it. After they confirm, save it.

MCP:

```text
ramp_save_plugin_marketplace_draft(rationale="<why>", plugin_id="<plugin id>", revision=<latest revision>, demo_video_url="https://...")
```

CLI:

```bash
ramp plugins draft --agent --json '{
  "rationale": "<why>",
  "plugin_id": "<plugin id>",
  "revision": <latest revision>,
  "demo_video_url": "https://..."
}'
```

## 5. Submit for review

Before submitting, check that the draft has a demo video link, an MCP server
URL, an authentication type, a description, and reviewer instructions. Ramp
refuses a submission without a demo video. It accepts a draft that is missing
the others, but a reviewer cannot approve one.

If submit fails with `MARKETPLACE_DEMO_VIDEO_REQUIRED`, the draft has no demo
video. Go back to step 4.

Show the user the saved draft. After they confirm, submit it.

MCP:

```text
ramp_submit_plugin_to_marketplace(rationale="<why>", version_id="<draft version id>", revision=<latest revision>)
```

CLI:

```bash
ramp plugins submit --agent --json '{
  "rationale": "<why>",
  "version_id": "<draft version id>",
  "revision": <latest revision>
}'
```

Tell the user the plugin is submitted and that review is done by Ramp.

## 6. Check the review

On a later request, read the plugin again with its `plugin_id`, using
`ramp_get_plugin_marketplace_status` (MCP) or `ramp plugins status` (CLI). The latest
version shows the status and, once reviewed, the reviewer's decision and
feedback.

- `submitted` or `in_review`: still with Ramp. Nothing to do.
- `changes_requested`: show the feedback and ask the user how they want to
  address it. Change the draft as in step 3 only when they ask, and submit
  again only after they confirm.
- `approved` or `published`: tell the user.
- `rejected`: show the feedback. Do not resubmit without the user's direction.
