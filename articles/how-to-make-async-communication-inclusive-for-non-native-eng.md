---
layout: default
title: "How to Make Async Communication Inclusive for Non-Native"
description: "A practical guide to writing async communication that works for global teams with diverse language backgrounds. Includes templates, tools, and concrete"
date: 2026-03-16
last_modified_at: 2026-03-22
author: theluckystrike
permalink: /how-to-make-async-communication-inclusive-for-non-native-eng/
categories: [guides]
tags: [remote-work-tools, async-communication, remote-work, inclusion, non-native-english, global-teams]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
---

{% raw %}

Async communication has become the backbone of remote work. Written discussions, Slack messages, GitHub comments, and shared documents replace the instant feedback of office life. For teams spread across continents, this shift offers flexibility—but it also creates barriers for team members who communicate in English as a second or third language.

If you write async communication that assumes native-level English fluency, you're excluding talented teammates and losing their contributions. This guide shows you how to write async messages that work for everyone on your team, regardless of their language background.

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: The Problem with English-Centric Async Communication

When you type quickly during a busy workday, you likely use idioms, slang, and complex sentence structures without thinking. Phrases like "circle back," "deep dive," or "low-hanging fruit" make perfect sense to native speakers but create confusion for others. Compound sentences with multiple clauses, passive voice, and implicit context all increase cognitive load.

Consider this typical Slack message:

> "Hey team, can we touch base on the API refactor? I think we're kinda barking up the wrong tree with the current approach. Maybe we should table this and revisit after the sprint demo?"

This message contains three idioms ("touch base," "barking up the wrong tree," "table this") and assumes the reader knows the sprint schedule. For a non-native speaker, parsing this takes significantly more effort than for a fluent speaker.

### Step 2: Writing Clear Async Messages

The core principle is simple: write for clarity first, brevity second. Clear communication actually takes less time to produce because it reduces follow-up questions and misunderstandings.

### Use Simple, Direct Language

Replace idioms with plain language. Instead of:

> "Let's circle back on this after standup"

Write:

> "Let's discuss this after our daily standup meeting"

Instead of abstract expressions, use concrete verbs. "Table this" becomes "postpone this." "Bark up the wrong tree" becomes "take the wrong approach."

### Break Up Complex Information

Long paragraphs with multiple ideas are harder to process. Group related points together and use bullet lists:

**Instead of:**
> "The new authentication system needs to be implemented by next Thursday and it should handle OAuth2 flows for Google and GitHub, plus we need to support JWT tokens for mobile clients and I think we should also consider adding rate limiting because of the API security concerns everyone has been talking about."

**Write:**
> "The new authentication system has these requirements:
> - Implement OAuth2 for Google and GitHub
> - Support JWT tokens for mobile clients
> - Add rate limiting for API security
> - Complete by next Thursday"

### Provide Explicit Context

Non-native speakers often struggle with implied context. Include information that native speakers would infer:

**Weak:** "Check the PR for details."

**Strong:** "I've opened PR #247 that implements the user dashboard. Please review the changes by Wednesday so we can merge before the release. The main changes are in `dashboard.js` and `api-routes.js`."

### Step 3: Code Examples and Technical Writing

For developer teams, technical communication carries additional complexity. Code examples, error messages, and technical discussions need special attention.

### Use Descriptive Variable and Function Names

Bad code creates confusion even for native speakers, but it disproportionately affects non-native developers:

```javascript
// Confusing
function process(data) {
  return data.filter(x => x.active).map(x => x.val);
}

// Clear
function getActiveUserIds(users) {
  return users
    .filter(user => user.isActive)
    .map(user => user.id);
}
```

### Write Meaningful Commit Messages

Generic commit messages force reviewers to dig through code to understand changes. Good commit messages follow a consistent format and explain the "why":

```bash
# Weak
"fix bug"

# Better
"Fix race condition in user authentication flow

The previous implementation checked auth state before each request
but didn't handle concurrent requests properly. This caused intermittent
login failures when users submitted forms quickly after page load.

Fixes #142"
```

### Document Edge Cases Explicitly

When writing technical documentation, spell out scenarios that native speakers might infer:

```markdown
### Step 4: API Rate Limiting

The API allows 100 requests per minute per API key.

Edge cases:
- If a client exceeds the limit, they receive HTTP 429
- The rate limit resets at the start of each minute
- Failed requests (4xx/5xx) still count toward the limit
- There's no burst allowance—clients must space requests evenly
```

### Step 5: Tools That Help

Several tools can help your team write more inclusive async communication:

**Language Tools:**
- Grammarly detects complex sentences and suggests simpler alternatives
- Hemingway Editor highlights difficult phrasing in real-time
- LanguageTool offers open-source grammar checking with support for multiple languages

**Team Practices:**
- Establish a team style guide with plain language requirements
- Use translation tools like DeepL for important announcements
- Create async video recordings (Loom) for complex topics—this lets people watch at their own pace and replay sections

**Process Adjustments:**
- Allow longer response windows for async discussions
- Designate "clarification champions" who help rephrase confusing messages
- Review past communications for patterns that cause confusion

### Step 6: Build Inclusive Async Habits

Making async communication inclusive requires ongoing attention, not one-time fixes. Start by auditing your recent written communications:

1. Count idioms and slang terms
2. Measure average sentence length
3. Check for implicit context (assumed knowledge)
4. Identify technical jargon without explanations

Then pick one improvement to focus on for two weeks. Small changes compound—using clear language consistently will improve comprehension for everyone on your team, native speakers included.

The goal isn’t to dumb down your communication. It’s to remove unnecessary barriers that have nothing to do with intelligence or capability. When you write async messages that work for non-native English speakers, you build a more inclusive team where everyone can contribute their best ideas.

### Step 7: Tools That Help with Inclusive Communication

Several tools can help you write more clearly:

**Real-Time Writing Tools**
- Grammarly Premium ($12/month): Suggests simpler alternatives for complex sentences
- Hemingway Editor (free web version): Highlights difficult phrasing in real-time
- LanguageTool (free and paid): Open-source grammar checker, works in browser and editors

**Translation and Clarity Tools**
- DeepL Translator (free-$7.99/month): Better quality than Google Translate for nuance
- Readable (free-$60/month): Analyzes readability, calculates readability score
- Simplish (free): Simplifies English text with one click

**Documentation Tools**
- Plain language checklist: Google "plain language checklist," use NIST standard
- Accessible writing guide: W3C has free guides for technical writing
- Screenshot + annotation tools: Loom for async video explanations

**Team Practices**
- Create a "clarity standard" document with your team examples
- Review past confusing messages and rewrite them clearly
- Celebrate clear writing when you see it

### Step 8: Communication Standards Template

Create a team document with these standards:

```markdown
# Our Communication Standards for Clarity

### Step 9: Sentence Structure
- Average sentence: 15-20 words maximum
- Use active voice (developer creates code, not code is created)
- One main idea per sentence

### Step 10: Vocabulary
- Avoid industry jargon without explanation
- Avoid idioms ("circle back" → "discuss again later")
- Avoid complex words when simple words work

### Step 11: Organization
- Use numbered lists for steps
- Use bullet points for related items
- Use headers to break up long text
- Lead with the main point, not background

### Step 12: Examples of Our Standards

Bad:
"We should probably touch base regarding the API refactor since I think we’re barking up the wrong tree."

Good:
"Let’s discuss the API refactor approach. I have concerns about the current direction."

Bad:
"The deployment process entails multiple steps which necessitate adherence to proper sequencing."

Good:
"Follow these deployment steps in order:
1. Check CI/CD pipeline status
2. Run smoke tests
3. Deploy to staging
4. Get approval from tech lead
5. Deploy to production"

Bad:
"Consider implementing optional enhancement to the notification system."

Good:
"Feature request: Add email notifications to alerts (currently only Slack)."
```

Share this with your team, update with real examples from your Slack history.

### Step 13: Measuring Inclusive Communication

Track whether your communication changes are working:

**Before/After Metrics**

Before clarity improvements:
- Average follow-up questions per Slack thread: 4-5
- Rework due to misunderstanding: 2-3 per sprint
- Time to understand async docs: 30-45 minutes

After clarity improvements (target):
- Average follow-up questions: 1-2
- Rework due to misunderstanding: 0-1 per sprint
- Time to understand async docs: 10-15 minutes

Track these for one month before changes, then one month after. You should see measurable improvement.

**Qualitative Feedback**

Ask non-native English speakers:
- Do you understand Slack messages on first read?
- Do you feel comfortable asking for clarification?
- Are there terms or patterns you find confusing?

Act on specific feedback. If multiple people say "circle back" is confusing, the team should stop using it.

### Step 14: Build Inclusive Technical Communication

For engineering teams, technical clarity is extra important:

**Clear API Documentation**
Instead of:
```javascript
// Manages user state operations with optional configuration overrides
function handleUserState(state, config) {
  // ...
}
```

Write:
```javascript
/**
 * Updates user state and saves to database
 *
 * @param {Object} state - The user object with updated fields
 * @param {Object} config - Optional settings
 *   - config.notify (boolean): Send email notification. Default: true
 *   - config.validate (boolean): Validate user data first. Default: true
 *
 * @returns {Promise<User>} Updated user object
 *
 * @example
 * const user = await handleUserState(
 *   { name: ‘Alice’, email: ‘alice@example.com’ },
 *   { notify: false }  // Don’t email about this update
 * )
 */
```

**Clear Error Messages**
Instead of:
```
ERROR: CONN_POOL_EXHAUSTED_MAX_CLIENTS_EXCEEDED
```

Write:
```
Error: Cannot connect to database

The database connection pool is full (50 active connections).
This means 50 other processes are using connections.

To fix:
1. Check if other services are running: `ps aux | grep postgres`
2. Kill unused connections: `SELECT * FROM pg_stat_activity`
3. Increase pool size in config.yaml if this is expected

Documentation: [link to connection pooling guide]
```

**Clear Code Review Comments**
Instead of:
```
This is inefficient.
```

Write:
```
Suggestion for performance improvement:
The current code queries the database in a loop (N+1 problem).
Each item triggers one query, so 100 items = 101 queries.

Instead, could we fetch all items once?

Current approach:
```python
items = get_items() # Query 1
for item in items:
 price = get_price(item.id) # Query 2, 3, 4... 101
```

Suggested approach:
```python
items = get_items()
prices = get_all_prices(item_ids) # Query 1
# match items with prices
```

This reduces database queries from 101 to 2.
```

### Step 15: Leadership Actions for Inclusive Communication

As a manager or senior engineer, you set the tone:

1. **Model clear writing**: Write clear Slack messages, clear commit messages, clear PRs
2. **Reward clarity**: "This RFC was really well written, thank you for the clarity"
3. **Fix unclear communication**: "I’m not sure I understood that, could you rephrase?"
4. **Create safe space for questions**: "I don’t understand X, can you explain?"
5. **Never mock language mistakes**: People should feel safe trying

Your behavior creates psychological safety around communication. When you ask for clarification without judgment, the whole team does.

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Frequently Asked Questions

**How long does it take to make async communication inclusive for non-native?**

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

**What are the most common mistakes to avoid?**

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

**Do I need prior experience to follow this guide?**

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

**Can I adapt this for a different tech stack?**

Yes, the underlying concepts transfer to other stacks, though the specific implementation details will differ. Look for equivalent libraries and patterns in your target stack. The architecture and workflow design remain similar even when the syntax changes.

**Where can I get help if I run into issues?**

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

## Related Articles

- [Best Voice Memo Apps for Quick Async Communication Remote](/remote-work-tools/a99-best-voice-memo-apps-for-quick-async-communication-remote-teams/)
- [Best Screen Recording Tools for Async Communication](/remote-work-tools/best-screen-recording-async-communication/)
- [How to Preserve Async Communication Culture When Team Moves](/remote-work-tools/how-to-preserve-async-communication-culture-when-team-moves-/)
- [#eng-announcements Channel Guidelines](/remote-work-tools/best-practice-for-remote-team-announcement-channel-keeping-s/)
- [Best Tool for Remote Team Capacity Planning When Scaling](/remote-work-tools/best-tool-for-remote-team-capacity-planning-when-scaling-eng/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
