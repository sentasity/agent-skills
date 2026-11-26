---
name: business-setup
description: Set up a Sentasity tenant's business context in a short conversation. Use when the user wants to describe what their business (or an MSP customer's business) does, its industry, or the compliance frameworks it answers to, or to mark which AWS accounts are production, staging or development.
---

# Business setup

Sentasity shapes its findings around what a business does: its industry and product tags, the compliance frameworks it answers to, and which AWS accounts run production. This skill turns a short conversation into that configuration through the Sentasity MCP tools. It covers the same fields as the portal's business settings, so it's another way in, not a second copy of the data. The portal's setup wizard still works the same way; use whichever the user prefers.

The server enforces the important parts on its own: every business-context write needs a confirmation token from a proposal the user saw. This skill adds the conversation around it.

## 1. Find the tenant

Start with `whoami`. A direct customer configures its own tenant (the `home_tenant`). An MSP may be setting up one of its customers: take the customer's id from `tenants_list` or `customers_get_setup`, and ask which customer when the user's words fit more than one.

## 2. Read what's set, then ask a short set of questions

Call `business_context_get` with the `tenant_id` first. It returns the tenant's current industry tags and compliance frameworks (with their statuses), including anything set in the portal. Tell the user what's already set, and ask only about what's missing or what they want to change.

Ask these in plain language, a couple at a time, and skip anything the user or the current context already answered:

1. What does the business do, and who are its customers?
2. Which industry is it in? Step 3 maps this to industry tags from the catalog.
3. Does it handle regulated data, such as health records, card payments, or personal data of EU residents?
4. Which compliance frameworks does it answer to (SOC 2, HIPAA, PCI DSS, ISO 27001 and so on), and for each one, is it maintaining it, pursuing it, aspiring to it, or not in scope?
5. Which AWS accounts are production, and which are staging, development, sandbox, shared or management?

Offer the user a choice before researching anything: they can answer themselves, or you can look the company up (its website, LinkedIn). If you research, confirm the company's identity with the user first ("is this the company at example.com?"), and treat what you find as a draft for them to correct.

## 3. Propose, show, then write

1. Call `business_context_propose` with the `tenant_id`, the `tag_ids` you'd suggest from the answers and the `compliance_frameworks` (each a `framework_id` with a `status`). It changes nothing.
2. Its response carries the `catalog` of business tags and the `framework_catalog` with its `framework_statuses` (the four statuses in question 4). Treat those catalogs as the only valid values. Never invent a tag or a framework: map the user's words onto the closest catalog entries, say which ones you picked, and propose again if the mapping changed.
3. Show the user the whole `diff`: every tag and framework that changes, its `before` side (what's set now, including anything set in the portal) and its `after` side.
4. Wait for an explicit yes. A changed answer means a new proposal.
5. On yes, write with the proposal's token. A proposal that covers one write returns one `confirmation_token`. One that covers both returns `confirmation_tokens`, and the two are not interchangeable: the `business_tags` token goes with `business_context_set_tags`, and the `business_info` token goes with `business_context_set_frameworks`.

A token expires after 10 minutes. If the write reports a missing, expired or mismatched token, propose again and get a new yes.

## 4. Mark the AWS accounts

List the tenant's accounts with `accounts_list` and read each `workload_classification`. For the accounts the user describes, set the workload with `accounts_set_workload` (one `account_id` at a time). This tool has no proposal step, so read the planned classifications back to the user and get a yes before setting them. Leave an account the user isn't sure about unclassified.

## For MSPs: many customers at once

An MSP can describe each customer once and apply it across that customer's accounts:

1. `customers_get_setup` lists the customers and their `account_ids`.
2. For each customer the user describes, run steps 2 to 4 against that customer's tenant: one proposal, one yes, one set of writes per customer.
3. Keep a running list of what's done and what's left, so the user can stop and pick up later.

## When the portal and this conversation disagree

Both write the same fields through the same API, so the last write wins. The proposal's `before` side always shows the current value, including one set in the portal, so the user sees what they're replacing before they say yes.
