# AI Usage Guidelines: OneSite Help Search Agent

## Overview

The **@onesite-help** agent is a custom GitHub Copilot agent that searches the OneSite Leasing & Rents online help documentation in real-time. Instead of manually navigating the retired help site frameset, you can ask natural-language questions directly in VS Code and get answers pulled from the live help pages.

**What it does:**

- Looks up a curated topic index of ~200+ OneSite help pages
- Fetches the relevant help pages directly from the production help server
- Follows internal links for deeper detail
- Summarizes findings with direct URLs to the source pages

**Example topics it covers:** quotes, reservations, move-ins, move-outs, renewals, screening, accounts, prospects, applicants, residents, non-residents, transactions, billing, evictions, units, floor plans, dashboards, settings, and more.

---

## Setup

There are two ways to set up the agent. Choose whichever works best for your workflow.

---

### Option A: Local Repository (Recommended)

This approach places the agent files directly in your repository so it appears as a configured agent in VS Code's Copilot Chat.

**Step 1:** Create the agents folder in your repository root (if it doesn't already exist):

```
mkdir .github\agents
```

**Step 2:** Download both files from the wiki and place them in `.github\agents\`:

| File | Wiki Source | Local Path |
|---|---|---|
| Agent definition | [onesite-help.agent.md](https://newwiki.realpage.com/display/PM/onesite-help.agent.md) | `.github/agents/onesite-help.agent.md` |
| Topic index | [onesite-help-topic-index.md](https://newwiki.realpage.com/display/PM/onesite-help-topic-index.md) | `.github/agents/onesite-help-topic-index.md` |

> **Important:** The filename must be exactly `onesite-help.agent.md` — VS Code uses the part before `.agent.md` as the agent's invocation name (`@onesite-help`).

**Step 3:** Open GitHub Copilot Chat in VS Code. The `@onesite-help` agent should now appear in the agent picker (click the agent icon or type `@`).

**Folder structure:**

```
your-repo/
├── .github/
│   └── agents/
│       ├── onesite-help.agent.md        ← agent definition
│       └── onesite-help-topic-index.md  ← topic index
├── src/
└── ...
```

---

### Option B: Reference via Wiki URL

If you don't want to add files to your repository, you can paste the agent instructions directly into a Copilot Chat prompt and reference the topic index by URL.

No setup required — just use the prompt patterns shown in the **How to Use** section below.

---

## How to Use

### Using Option A (Local Agent)

Once the agent files are in `.github/agents/`, invoke it by typing `@onesite-help` followed by your question in Copilot Chat.

**Sample prompts:**

| Prompt | What it finds |
|---|---|
| `@onesite-help what are the quote statuses?` | Lists all quote statuses (Active, Expired, Reserved, Leased) with descriptions |
| `@onesite-help how do I cancel a reservation?` | Steps, prerequisites, and cancellation reasons |
| `@onesite-help what is the difference between residents and non-residents?` | Comparison of resident vs. non-resident accounts and unit types |
| `@onesite-help what are the steps for a move-in?` | Move-in process, checklist, and prerequisites |
| `@onesite-help what rights are needed to post scheduled billing?` | Required user permissions for billing operations |
| `@onesite-help what filters are available on the prospects page?` | All filter options on the Prospects list page |
| `@onesite-help what is a final account statement (FAS)?` | FAS overview, process, and related pages |

---

### Using Option B (Wiki URL Reference)

If you haven't set up the local agent, you can achieve the same result by giving Copilot the wiki context directly. Use this prompt pattern:

**Template:**

```
Using the topic index at https://newwiki.realpage.com/display/PM/onesite-help-topic-index.md
and the agent instructions at https://newwiki.realpage.com/display/PM/onesite-help.agent.md,
search the OneSite help documentation for: [YOUR QUESTION HERE]
```

**Sample prompts:**

---

**Prompt 1 — Account and resident statuses:**

```
Using the topic index at https://newwiki.realpage.com/display/PM/onesite-help-topic-index.md
and the agent instructions at https://newwiki.realpage.com/display/PM/onesite-help.agent.md,
search the OneSite help documentation for: what are the account statuses and resident statuses?
```

---

**Prompt 2 — Reservation hold period:**

```
Using the topic index at https://newwiki.realpage.com/display/PM/onesite-help-topic-index.md
and the agent instructions at https://newwiki.realpage.com/display/PM/onesite-help.agent.md,
search the OneSite help documentation for: how does the reservation hold period work?
```

---

**Prompt 3 — Eviction process:**

```
Using the topic index at https://newwiki.realpage.com/display/PM/onesite-help-topic-index.md
and the agent instructions at https://newwiki.realpage.com/display/PM/onesite-help.agent.md,
search the OneSite help documentation for: what is the eviction process?
```

> **Tip:** Option B is longer to type but useful for one-off questions or if you're working in a repo where you can't add agent files.

---

## Comparison of Approaches

| | Option A: Local Agent | Option B: Wiki URL |
|---|---|---|
| **Setup** | One-time: copy 2 files to `.github/agents/` | None |
| **Invocation** | `@onesite-help [question]` | Paste full prompt with URLs |
| **Convenience** | Short, clean prompts | Verbose but no setup |
| **Works across repos** | Only in repos with the files | Anywhere in Copilot Chat |
| **Auto-discovery** | Shows in VS Code agent picker | Manual prompt only |
| **Recommended for** | Daily use by the team | Ad-hoc / occasional use |

---

## Tips for Best Results

1. **Be specific with keywords** — "cancel reservation steps" works better than "reservations"
2. **Ask about statuses/filters** — the agent finds filter descriptions that list all status values
3. **Ask follow-ups** — after an initial answer, ask the agent to drill into a linked sub-topic
4. **Use OneSite terminology** — terms like "guest card", "FAS", "scheduled billing", "ILM" get precise matches
5. **The agent fetches live pages** — content is always current from the production help server

---

## Troubleshooting

| Issue | Solution |
|---|---|
| `@onesite-help` doesn't appear in agent list | Verify the file is at `.github/agents/onesite-help.agent.md` (exact path and name) |
| Agent returns empty/no results | Try broader keywords or ask about the parent section (e.g., "quotes" instead of a specific sub-topic) |
| "Help Center Has Moved" message | The agent handles this automatically — it bypasses the retired frameset and fetches pages directly |
| Wiki URL approach doesn't fetch pages | Ensure Copilot has web access enabled (`fetch_webpage` tool must be available) |

---

## Source Files

- **Agent definition:** [onesite-help.agent.md](https://newwiki.realpage.com/display/PM/onesite-help.agent.md)
- **Topic index:** [onesite-help-topic-index.md](https://newwiki.realpage.com/display/PM/onesite-help-topic-index.md)
- **Production help site:** `https://onesitehelp.realpage.com/OneSiteHTMLHelp/leasing%20and%20rents/40/{TOPIC_ID}.htm`

---

*Last updated: March 2026*
