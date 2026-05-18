---
title: "CRGenerator: automating Change Request documentation from git history"
date: 2024-09-20T10:00:00+12:00
draft: false
tags: ["Go", "JIRA", "Git", "DevOps", "Automation"]
readingTime: 3
customSummary: A small Go CLI that walks your git commit range, extracts JIRA ticket numbers, and pulls the issue details — so you can stop writing Change Requests by hand.
---

## The problem

Most regulated software environments require a Change Request before any production deployment. The CR needs a list of every JIRA issue included in the release — what changed, why, and who approved it.

In practice, that means someone opens the JIRA board, opens the git log, and manually cross-references the two. Commit messages contain ticket numbers like `PROJ-1234 fix null pointer in payment service`. There might be 30 of them across a two-week sprint. You copy each one, look it up, paste the summary into a doc, repeat. It takes 20–30 minutes and is exactly the kind of task where humans make mistakes — missing a ticket, duplicating one, or pulling the wrong summary.

[`crgenerator`](https://github.com/GaaraZhu/crgenerator) automates this entirely.

## How it works

Point it at two git refs — commits, tags, or branch names — and it does the rest:

```bash
# Between two commits
crgenerator abc1234 def5678

# Between two tags (common for release-to-release)
crgenerator v1.3.0 v1.4.0

# Between a branch and HEAD
crgenerator origin/main HEAD
```

It walks every commit in that range, extracts all JIRA ticket numbers from the commit messages, calls the JIRA API to fetch the issue details, and prints a formatted summary ready to paste into your CR.

![crgenerator output showing JIRA issues between commits](https://github.com/user-attachments/assets/7914f4e2-936d-4148-9a0b-bdd38b78643b)

## Setup

Install via Homebrew and configure your JIRA credentials once:

```bash
brew install crgenerator
```

```bash
echo 'export JIRA_BASE_URL=https://yourorg.atlassian.net' >> ~/.zshrc
echo 'export JIRA_USER_NAME=you@yourorg.com' >> ~/.zshrc
echo 'export JIRA_API_TOKEN=your-api-token' >> ~/.zshrc
source ~/.zshrc
```

Your JIRA API token is generated from your Atlassian account settings. After that, run it from any git repository.

## Why this matters

The manual CR process is one of those tasks that looks simple but accumulates real cost at scale. A team shipping weekly to a regulated environment spends hours per quarter on this. More importantly, a missed ticket in a CR is a compliance gap — the kind that causes audit findings.

Automating it removes both the cost and the risk. The tool is deterministic: same git range, same JIRA state, same output. It's also fast enough to run as part of a release checklist without slowing anyone down.

The source is on [GitHub](https://github.com/GaaraZhu/crgenerator), MIT-licensed.
