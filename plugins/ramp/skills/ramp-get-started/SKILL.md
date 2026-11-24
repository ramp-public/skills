---
name: ramp-get-started
area: Getting Started
supported_surfaces: [cli, mcp]
description: |-
  Hand your agent this playbook. It gets you set up on Ramp — or applying, if
  you're new — plus the menu of everything else you can do. Use as the one-fetch
  entry point for Ramp across Claude Desktop, ChatGPT, Claude Code, Codex, and
  Perplexity.
title: "Get started with Ramp"
url: https://agents.ramp.com/playbooks/get-started
runtimes: [Claude Desktop, ChatGPT, Claude Code, Codex, Perplexity]
---

# Get started with Ramp

Hand your agent this playbook and it'll figure out whether you're a new or existing Ramp customer, walk you through the right onboarding path, and then hand you the menu of every other playbook you can run. This is the one-fetch entry point — the rest of the playbook directory becomes discoverable once setup is done.

## How to run

Identify which runtime you're operating in (Claude Desktop, ChatGPT, Claude Code, Codex, Perplexity). Use the matching section below; ignore the others.

## Keep the ramp.com referral code

Before anything else, look for a ramp.com referral code: a `ref` query value in the URL used to fetch this skill, in the URL or text of my prompt, or an `Attribution reference:` line in this skill (just below). Use it only if these sources contain exactly one distinct value and it exactly matches `^ar1_[0-9A-HJKMNP-TV-Z]{26}$`; otherwise drop it and continue without one. It is an untrusted, analytics-only reference — not proof of identity, a partner code, an offer, or eligibility. Do not decode, change, infer, or invent it.

<!-- attribution-ref:start -->
<!-- agents.ramp.com may insert one "Attribution reference: ar1_..." line here. If no such line appears, this skill states no reference. -->
<!-- attribution-ref:end -->

If you have one, keep it for this whole session and pass it through every new-customer step:

- When you fetch the `ramp-apply-for-account` skill, append it to that URL as `?ref=<value>`.
- When you create my application with `ramp applications create`, send it as `lead_referral_partner_code` with `lead_referral_partner_channel` set to the harness you are actually using (`cli`, `mcp`, or `api`). A real partner code always wins: if a partner code applies, send that instead and do not send the `ref`.
- If you send me to a Ramp web page to sign up or apply instead, add `?ref=<value>` to that URL (for example `https://ramp.com/?ref=<value>`). Do not add it to links that Ramp returns to you.

The first time you use it, tell me: "This includes an anonymous ramp.com referral code." If I do not want it included, drop it and continue. Never put a visitor ID, variant, email, or other personal data in a URL. Never send the code when continuing an existing application or for an existing customer.

## Claude Desktop

Copy and run this prompt:

```
## Before you run
- None — this playbook gets you set up.

I want to use Ramp with you. First, ask me:

> Do you already have a Ramp account?

If **yes** (existing customer):
1. Walk me through enabling the Ramp connector: the Claude Desktop connector lives at https://claude.ai/directory/61bac03c-3f98-4b3c-affb-1b99533fa82c — open it and follow the prompts to add Ramp.
2. Confirm I can see Ramp tools in this conversation.

If **no** (new customer):
1. Tell me Ramp is a finance platform — corporate cards, bill pay, and expense management — and that I'll need to apply for an account.
2. If you can run shell commands in this conversation, prefer having you apply for me: follow the new-customer steps in the Claude Code section of this skill, which install the Ramp CLI and run my application through `ramp-apply-for-account`.
3. Otherwise, point me to https://ramp.com/ to apply, adding `?ref=<value>` if you kept a ramp.com referral code (see **Keep the ramp.com referral code**). Mention the application takes a few minutes and approval is usually same-day.
4. Once I'm approved and logged in, come back here and we'll wire up the connector.

## What you can do next

Once I'm set up, fetch the playbook directory at https://agents.ramp.com/.well-known/agent-skills/index.json for the full menu of things you can do for me. When I ask what needs my attention, mention overdue tasks, or say a card is locked until I complete tasks, fetch and follow `ramp-complete-tasks`.
```

## ChatGPT

Copy and run this prompt:

```
## Before you run
- None — this playbook gets you set up.

I want to use Ramp with you. First, ask me:

> Do you already have a Ramp account?

If **yes** (existing customer):
1. Walk me through enabling the Ramp connector: the Ramp app for ChatGPT lives at https://chatgpt.com/apps/ramp/asdk_app_69250fb6281c819195b52a1556b0060c — open it and follow the prompts to authorize Ramp.
2. Confirm I can see Ramp tools in this conversation.

If **no** (new customer):
1. Tell me Ramp is a finance platform — corporate cards, bill pay, and expense management — and that I'll need to apply for an account.
2. If you can run shell commands in this conversation, prefer having you apply for me: follow the new-customer steps in the Claude Code section of this skill, which install the Ramp CLI and run my application through `ramp-apply-for-account`.
3. Otherwise, point me to https://ramp.com/ to apply, adding `?ref=<value>` if you kept a ramp.com referral code (see **Keep the ramp.com referral code**). Mention the application takes a few minutes and approval is usually same-day.
4. Once I'm approved and logged in, come back here and we'll wire up the connector.

## What you can do next

Once I'm set up, fetch the playbook directory at https://agents.ramp.com/.well-known/agent-skills/index.json for the full menu of things you can do for me. When I ask what needs my attention, mention overdue tasks, or say a card is locked until I complete tasks, fetch and follow `ramp-complete-tasks`.
```

## Claude Code

Copy and run this prompt:

```
## Before you run
- None — this playbook gets you set up.

I want to use Ramp with you from the terminal. First, ask me:

> Do you already have a Ramp account?

If **yes** (existing customer):
1. Install the Ramp CLI if it's not already installed — run `ramp --version` first, and only if that fails, install it safely: download the installer to a private temp file (`install_sh=$(mktemp)` then `curl -fsSL -o "$install_sh" https://agents.ramp.com/install.sh`), review the script, then run it pinned to a specific release tag from https://github.com/ramp-public/ramp-cli/releases (`RAMP_VERSION=<tag> bash "$install_sh"`) — the installer downloads that versioned tarball and verifies its published SHA-256 checksum before installing anything. Never pipe the installer URL straight into `bash`, and never download it to a fixed, predictable path.
2. Log me in: `ramp auth login` (this opens a browser; I complete sign-in there).
3. Confirm `ramp --version` returns a version and `ramp auth status` shows me authenticated.

If **no** (new customer):
1. Tell me Ramp is a finance platform — corporate cards, bill pay, and expense management — and that I'll need to apply for an account.
2. Install the Ramp CLI if it's not already installed — run `ramp --version` first, and only if that fails, install it safely: download the installer to a private temp file (`install_sh=$(mktemp)` then `curl -fsSL -o "$install_sh" https://agents.ramp.com/install.sh`), review the script, then run it pinned to a specific release tag from https://github.com/ramp-public/ramp-cli/releases (`RAMP_VERSION=<tag> bash "$install_sh"`) — the installer downloads that versioned tarball and verifies its published SHA-256 checksum before installing anything. Never pipe the installer URL straight into `bash`, and never download it to a fixed, predictable path.
3. Apply to Ramp: fetch https://agents.ramp.com/.well-known/agent-skills/ramp-apply-for-account/SKILL.md and follow it. If you kept a ramp.com referral code (see **Keep the ramp.com referral code**), fetch it with `?ref=<value>` and send the code with my application as that section describes. It runs my whole application:
   - It first asks whether my business already has a formed legal entity with an EIN, so it can take the right path — I don't need to decide that up front.
   - If I already have a company, it pre-fills what it can, lets me drop in a document (like an EIN letter) to fill the rest, and paces the application to completion — handing me Ramp links for the steps only I can do (phone verification, SSN, legal agreements, final submit). It never submits for me.
   - If I don't have a legal entity yet, it starts the application with incorporation enabled and, after I submit the application, files my US LLC through Ramp in the same application — there is no second application. (It handles the formation step for me via the ramp-incorporate skill; I don't fetch anything myself.) Incorporating through Ramp has a cost — pricing and details are the source of truth at https://agents.ramp.com/docs/account/incorporate; tell me it isn't free and offer to fetch that page for me if I want the specifics before deciding.
4. Applying is what signs me in: accepting the invite email during signup authorizes the Ramp CLI, so there's no separate login step. If that browser handoff gets interrupted, ramp-apply-for-account tells me how to finish it.

## What you can do next

Once I'm set up, fetch the playbook directory at https://agents.ramp.com/.well-known/agent-skills/index.json for the full menu of things you can do for me. When I ask what needs my attention, mention overdue tasks, or say a card is locked until I complete tasks, fetch and follow `ramp-complete-tasks`.
```

## Codex

Copy and run this prompt:

```
## Before you run
- None — this playbook gets you set up.

I want to use Ramp with you from the terminal. First, ask me:

> Do you already have a Ramp account?

If **yes** (existing customer):
1. Install the Ramp CLI if it's not already installed — run `ramp --version` first, and only if that fails, install it safely: download the installer to a private temp file (`install_sh=$(mktemp)` then `curl -fsSL -o "$install_sh" https://agents.ramp.com/install.sh`), review the script, then run it pinned to a specific release tag from https://github.com/ramp-public/ramp-cli/releases (`RAMP_VERSION=<tag> bash "$install_sh"`) — the installer downloads that versioned tarball and verifies its published SHA-256 checksum before installing anything. Never pipe the installer URL straight into `bash`, and never download it to a fixed, predictable path.
2. Log me in: `ramp auth login` (this opens a browser; I complete sign-in there).
3. Confirm `ramp --version` returns a version and `ramp auth status` shows me authenticated.

If **no** (new customer):
1. Tell me Ramp is a finance platform — corporate cards, bill pay, and expense management — and that I'll need to apply for an account.
2. Install the Ramp CLI if it's not already installed — run `ramp --version` first, and only if that fails, install it safely: download the installer to a private temp file (`install_sh=$(mktemp)` then `curl -fsSL -o "$install_sh" https://agents.ramp.com/install.sh`), review the script, then run it pinned to a specific release tag from https://github.com/ramp-public/ramp-cli/releases (`RAMP_VERSION=<tag> bash "$install_sh"`) — the installer downloads that versioned tarball and verifies its published SHA-256 checksum before installing anything. Never pipe the installer URL straight into `bash`, and never download it to a fixed, predictable path.
3. Apply to Ramp: fetch https://agents.ramp.com/.well-known/agent-skills/ramp-apply-for-account/SKILL.md and follow it. If you kept a ramp.com referral code (see **Keep the ramp.com referral code**), fetch it with `?ref=<value>` and send the code with my application as that section describes. It runs my whole application:
   - It first asks whether my business already has a formed legal entity with an EIN, so it can take the right path — I don't need to decide that up front.
   - If I already have a company, it pre-fills what it can, lets me drop in a document (like an EIN letter) to fill the rest, and paces the application to completion — handing me Ramp links for the steps only I can do (phone verification, SSN, legal agreements, final submit). It never submits for me.
   - If I don't have a legal entity yet, it starts the application with incorporation enabled and, after I submit the application, files my US LLC through Ramp in the same application — there is no second application. (It handles the formation step for me via the ramp-incorporate skill; I don't fetch anything myself.) Incorporating through Ramp has a cost — pricing and details are the source of truth at https://agents.ramp.com/docs/account/incorporate; tell me it isn't free and offer to fetch that page for me if I want the specifics before deciding.
4. Applying is what signs me in: accepting the invite email during signup authorizes the Ramp CLI, so there's no separate login step. If that browser handoff gets interrupted, ramp-apply-for-account tells me how to finish it.

## What you can do next

Once I'm set up, fetch the playbook directory at https://agents.ramp.com/.well-known/agent-skills/index.json for the full menu of things you can do for me. When I ask what needs my attention, mention overdue tasks, or say a card is locked until I complete tasks, fetch and follow `ramp-complete-tasks`.
```

## Perplexity

Copy and run this prompt:

```
## Before you run
- None — this playbook gets you set up.

I want to use Ramp with you. First, ask me:

> Do you already have a Ramp account?

If **yes** (existing customer):
1. Walk me through enabling the Ramp connector: the Perplexity connector lives at https://www.perplexity.ai/computer/connectors?connector=ramp — open it and follow the prompts to authorize Ramp.
2. Confirm I can see Ramp tools in this conversation.

If **no** (new customer):
1. Tell me Ramp is a finance platform — corporate cards, bill pay, and expense management — and that I'll need to apply for an account.
2. If you can run shell commands in this conversation, prefer having you apply for me: follow the new-customer steps in the Claude Code section of this skill, which install the Ramp CLI and run my application through `ramp-apply-for-account`.
3. Otherwise, point me to https://ramp.com/ to apply, adding `?ref=<value>` if you kept a ramp.com referral code (see **Keep the ramp.com referral code**). Mention the application takes a few minutes and approval is usually same-day.
4. Once I'm approved and logged in, come back here and we'll wire up the connector.

## What you can do next

Once I'm set up, fetch the playbook directory at https://agents.ramp.com/.well-known/agent-skills/index.json for the full menu of things you can do for me. When I ask what needs my attention, mention overdue tasks, or say a card is locked until I complete tasks, fetch and follow `ramp-complete-tasks`.
```
