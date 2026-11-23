---
name: business-setup
description: Set up a Sentasity tenant's business context in a short conversation. Use when the user wants to describe what their business (or an MSP customer's business) does, its industry, or the compliance frameworks it answers to, or to mark which AWS accounts are production, staging or development.
---

# Business setup

Sentasity shapes its findings around what a business does: its industry and product tags, the compliance frameworks it answers to, and which AWS accounts run production. This skill turns a short conversation into that configuration through the Sentasity MCP tools. It covers the same fields as the portal's business settings, so it's another way in, not a second copy of the data. The portal's setup wizard still works the same way; use whichever the user prefers.

The server enforces the important parts on its own: every business-context write needs a confirmation token from a proposal the user saw. This skill adds the conversation around it.

## 1. Find the tenant

Start with `whoami`. A direct customer configures its own tenant (the `home_tenant`). An MSP may be setting up one of its customers: take the customer's id from `tenants_list` or `customers_get_setup`, and ask which customer when the user's words fit more than one.

## 2. See what's set now

No tool reads the current tags and frameworks directly. To show the user what's already there, call `business_context_propose` with the values you're about to suggest and read each diff entry's `before` side. Its response also carries the `catalog` of business tags and the `framework_catalog` with its `framework_statuses`. Treat those catalogs as the only valid values. Never invent a tag or a framework, and map the user's words onto the closest catalog entries, saying which ones you picked.

## 3. Ask a short set of questions

Ask these in plain language, a couple at a time, and skip anything the user already answered:

1. What does the business do, and who are its customers?
2. Which industry is it in? Map this to industry tags from the `catalog`.
3. Does it handle regulated data, such as health records, card payments, or personal data of EU residents?
4. Which compliance frameworks does it answer to (SOC 2, HIPAA, PCI DSS, ISO 27001 and so on), and for each one, is it maintaining it, pursuing it, aspiring to it, or not in scope? Those are the four `framework_statuses`.
5. Which AWS accounts are production, and which are staging, development, sandbox, shared or management?

Offer the user a choice before researching anything: they can answer themselves, or you can look the company up (its website, LinkedIn). If you research, confirm the company's identity with the user first ("is this the company at example.com?"), and treat what you find as a draft for them to correct.

## 4. Propose, show, then write

1. Call `business_context_propose` with the `tenant_id`, the `tag_ids` and the `compliance_frameworks` (each a `framework_id` with a `status`).
2. Show the user the whole `diff`: every tag and framework that changes, before and after.
3. Wait for an explicit yes. A changed answer means a new proposal.
4. On yes, write with the token: `business_context_set_tags` takes the `confirmation_token` for tags, and `business_context_set_frameworks` takes it for frameworks. When the proposal covers both, `confirmation_tokens` holds one token for each (`business_tags` and `business_info`).

A token expires after 10 minutes. If the write reports a missing, expired or mismatched token, propose again and get a new yes.

## 5. Mark the AWS accounts

List the tenant's accounts with `accounts_list` and read each `workload_classification`. For the accounts the user describes, set the workload with `accounts_set_workload` (one `account_id` at a time). This tool has no proposal step, so read the planned classifications back to the user and get a yes before setting them. Leave an account the user isn't sure about unclassified.

## For MSPs: many customers at once

An MSP can describe each customer once and apply it across that customer's accounts:

1. `customers_get_setup` lists the customers and their `account_ids`.
2. For each customer the user describes, run steps 2 to 5 against that customer's tenant: one proposal, one yes, one set of writes per customer.
3. Keep a running list of what's done and what's left, so the user can stop and pick up later.

## When the portal and this conversation disagree

Both write the same fields through the same API, so the last write wins. The proposal's `before` side always shows the current value, including one set in the portal, so the user sees what they're replacing before they say yes.
