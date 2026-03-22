---
layout: default
title: "Best Invoicing Workflow for Solo Developer"
description: "Learn how to build an efficient invoicing workflow tailored for solo developers working with international clients. Includes practical examples"
date: 2026-03-16
author: theluckystrike
permalink: /best-invoicing-workflow-for-solo-developer-with-international-clients/
categories: [guides]
tags: [remote-work-tools, invoicing, freelance, international, payments, finance, best-of, workflow]
reviewed: true
score: 9
voice-checked: true
intent-checked: true
---


{% raw %}

Use Wise as your primary payment account, invoice in your home currency with automated templates, and track everything in a single spreadsheet or Notion database. That combination handles currency conversion, minimizes fees, and keeps international tax documentation organized. This guide walks through the full invoicing workflow designed specifically for solo developers managing international client relationships.

## The Core Challenge

International invoicing introduces complexity that domestic work avoids. You need to convert your rates into client currencies, handle VAT or GST registration in some regions, track which payments have cleared, and follow up on invoices across time zones. A disorganized approach means lost hours on administrative work and delayed payments that hurt your business operations.

The solution is building a systematic workflow that handles the variations while keeping your time investment minimal. This means standardizing your processes, using appropriate tools, and setting clear expectations with clients from the start.

## Structuring Your Invoicing System

### Client Information Collection

Before sending your first invoice, gather the necessary information from each client. International clients typically require:

- Company name and registered address
- Tax identification number (VAT, GST, or local equivalent)
- Preferred currency for payment
- Preferred payment method and bank details
- Purchase order number if applicable

Create a client intake form that captures this information. Store it in a consistent location, whether that's a simple folder system or a dedicated CRM tool. You'll reference this information for every invoice, so organizing it upfront saves repeated requests later.

```markdown
## Client Information Template

- Client Name: [Company/Individual Name]
- Address: [Full registered address]
- Tax ID: [VAT/GST/Local tax number]
- Billing Currency: [USD/EUR/GBP/etc]
- Payment Terms: [Net 15/30/45/60]
- PO Number Required: [Yes/No]
- Notes: [Any special requirements]
```

### Invoice Numbering System

Establish a clear invoice numbering convention from day one. A reliable system includes the year, month, and a sequential number:

```
INV-2026-0316-001
```

This format breaks down as: `INV-[Year]-[MonthDay]-[Sequential]`. It makes invoices easily sortable, prevents duplicates, and helps with year-end accounting when you need to locate specific documents.

Never reuse invoice numbers. If you delete a draft invoice, skip that number rather than reassigning it. Your accounting software or tax records depend on this sequence being intact.

## Currency and Payment Strategy

### Setting Your Base Currency

Most solo developers work in their home currency as the base. If you're based in the United States, USD makes sense. European developers might choose EUR. The key is deciding whether to invoice clients in your base currency or their local currency.

Invoicing in your base currency simplifies your accounting significantly. You record revenue in one currency, making tax reporting straightforward. The downside is your clients bear the currency conversion risk, which can frustrate some clients, especially when exchange rates fluctuate significantly between invoicing and payment.

Invoicing in client currencies removes that concern for them, but you accept the conversion risk. This works when you have systems to track exchange rates and understand the tax implications in each client's jurisdiction.

### Handling Payment Methods

For international clients, consider which payment methods you'll accept:

Bank transfers (SWIFT/SEPA) work globally but involve fees, typically $15–30 per international transfer. These fees may be split with clients or absorbed into your pricing. Include your full bank details clearly on invoices, including SWIFT/BIC codes for international wires.

PayPal avoids currency conversion headaches but charges 4–4.5% plus fixed fees, making it acceptable for smaller invoices where the convenience justifies the cost. Wise (formerly TransferWise) offers mid-market exchange rates and lower fees than traditional banks, and many freelancers now use it as their primary business account for international work.

Cryptocurrency works for some clients, particularly in tech. Accepting stablecoins avoids volatility and allows conversion to fiat when needed.

Document your accepted payment methods in your initial client agreement and on every invoice to prevent confusion.

## Automating the Workflow

### Invoice Generation Process

Rather than creating invoices from scratch each time, build templates in your chosen tool. Most solo developers use either dedicated invoicing software or simple document templates.

For developers comfortable with command-line tools, generating invoices from markdown files provides maximum customization:

```python
#!/usr/bin/env python3
import datetime
import json

def generate_invoice(client_data, items, invoice_num):
    date = datetime.date.today()

    invoice = f"""# INVOICE

**Invoice Number:** {invoice_num}
**Date:** {date}
**Due Date:** {date + datetime.timedelta(days=30)}
---

**Bill To:**
{client_data['name']}
{client_data['address']}

---

| Description | Quantity | Rate | Amount |
|-------------|-----------|------|--------|
"""

 for item in items:
 invoice += f"| {item['desc']} | {item['qty']} | ${item['rate']} | ${item['qty'] * item['rate']} |\n"

 total = sum(item['qty'] * item['rate'] for item in items)
 invoice += f"""
---
**Total:** ${total}

Payment due within 30 days.
"""

 return invoice

# Example usage
client = {
 "name": "Acme Corp International",
 "address": "123 Business Ave, London, UK"
}

items = [
 {"desc": "API Integration Development", "qty": 20, "rate": 150},
 {"desc": "Documentation", "qty": 5, "rate": 100}
]

print(generate_invoice(client, items, "INV-2026-0316-001"))
```

This approach lets you version-control your invoices alongside your code, and it generates consistent output every time.

### Payment Tracking System

Create a simple tracking system to monitor invoice status. A spreadsheet or Notion database works well for solo practices:

| Invoice # | Client | Amount | Currency | Sent | Due | Paid | Notes |
|-----------|--------|--------|----------|------|-----|------|-------|
| INV-2026-0316-001 | Acme Corp | 3500 | USD | Mar 16 | Apr 15 | - | Follow up Apr 1 |

Update this tracker every time you send an invoice or receive payment. Set calendar reminders for follow-ups before invoices become overdue.

## Tax Considerations for International Work

International invoicing triggers tax obligations that vary by jurisdiction. The rules are complex, but three regions require attention.

In the European Union, VAT applies if you sell services to EU clients. Services to business clients in other EU countries are generally VAT-exempt under the reverse charge mechanism, but you may need to register if you provide services to consumers in other EU countries. Australia, New Zealand, and Canada have similar GST rules—you may need to register once your sales to that country exceed their threshold, even without a physical presence. For US clients, most digital services are not subject to sales tax, but rules vary by state.

For most solo developers, using an accountant familiar with international tax treaties provides the safest approach. They can advise on registration requirements and help you collect the correct tax information from clients.

## Client Communication Templates

Set expectations early with clear communication about your invoicing process:

**Payment Terms Statement:**
> Payment is due within 30 days of invoice date. For international payments, please account for processing time and transfer fees. Late payments incur a 1.5% monthly finance charge.

**Invoice Follow-Up:**
> Hi [Name], I wanted to follow up on invoice #[number] for [amount], which was due on [date]. Please let me know if you need any additional information or if there are any issues with processing this payment.

Keep communications professional but firm. Consistent follow-up on overdue invoices is essential for maintaining healthy cash flow.

## Tool Comparison for Solo Developers

| Tool | Cost | Best For | Limitation |
|------|------|----------|-----------|
| **Wave** | Free | Basic invoicing + bookkeeping | Limited international support, no multi-currency easily |
| **Wise** | 1-2% fee | Primary business account | Requires opening account, not invoice generation |
| **FreshBooks** | $20-50/mo | Small teams, expense tracking | More features than solo devs typically need |
| **Stripe Invoicing** | Free or 1-3% transaction | Accepting payments directly | Not invoicing specifically, more payment-first |
| **Google Forms + Sheets** | Free | Cost-conscious developers | Manual, but flexible |
| **HubDoc** | $15-30/mo | Freelancers managing multiple clients | Cloud-based, good for organization |
| **Invoice Ninja** | Free open-source | Self-hosted developers | Requires setup, no SaaS convenience |

For most solo international developers, combine **Wise** for banking (lower fees than traditional banks), **Wave** for invoicing (free), and a **Google Sheet** for payment tracking. This three-tool stack costs nothing and handles all currency conversion needs.

## Real-World Payment Flow

Here's what a typical international payment cycle looks like:

1. **Send invoice** (March 1): Invoice #INV-2026-0301-001 for €3,500 ($3,850 USD equivalent at 1.10 rate)
2. **Client receives** (March 2-3): Email with invoice attached, includes payment instructions
3. **Payment processing** (March 4-10): Wire transfer initiated by client through their bank
4. **Wise receives** (March 10-12): Deposit arrives at your Wise account (2-3 business days typical for EU transfers)
5. **Currency conversion** (March 12): Wise holds the EUR, you convert to your home currency when rate is favorable
6. **Mark paid** (March 13): Update your tracking sheet, invoice marked complete

Total time to cash in hand: 12-13 days. This is typical for SWIFT transfers internationally. Some clients use PayPal which is faster (2-3 days) but costs more in fees.

## Negotiating Payment Terms with International Clients

International payment delays are normal. Protect yourself by being explicit:

**Bad**: "Payment due upon completion" (leaves ambiguity—do they pay when delivered, approved, or invoiced?)

**Good**: "Net 30 from invoice date. Invoice date is [specific date]. First payment reminder at day 15, escalation at day 45."

**Better**: "Net 15 for international transfers due to processing delays. Invoiced [date]. Expected payment [specific date]."

For first-time clients in unknown locations, consider:

- 50% upfront, 50% on delivery
- Shorter payment terms (Net 15 instead of Net 30)
- Escrow services like Upwork escrow (if you have an Upwork account) as backup

For established clients with good history, Net 30 is reasonable. Never offer Net 45+ for international work unless it's a significant contract where that risk is calculated.

## Building Client Relationships That Reduce Friction

International invoicing works smoothly when client relationships are clear from the start.

In your initial project agreement email:

> "Billing is in [your currency]. I invoice on [your schedule, e.g., weekly, upon delivery, monthly]. Invoices are payable within 30 days via SWIFT transfer. I provide all bank details on the invoice. For faster payment, I also accept PayPal (with 4% fee) or Wise (free)."

State this clearly before work begins. Clients who object to payment terms rarely become smooth-paying clients anyway. Address friction early.

## Tax Documentation for Annual Filing

Keep invoices organized for tax time. Create an invoice export that includes:

- Invoice number
- Invoice date
- Client name and country
- Amount in original currency and home currency
- Exchange rate used (if applicable)
- Payment received date
- Payment method

Many accountants will ask for this data in spreadsheet form. The easier you make it, the lower your accounting costs.

Example annual summary:

| Invoice Range | Total USD Invoiced | Total Paid | Unpaid | Primary Currencies |
|---------------|-------------------|-----------|--------|-------------------|
| Q1 2026 | $18,500 | $17,200 | $1,300 | USD, EUR, GBP |
| Q2 2026 | $22,100 | $22,100 | $0 | USD, CAD |

This helps your accountant understand your international exposure and advise on tax treaty implications.

## Cash Flow Management for International Payments

One challenge with international invoicing: payment timing. Transfers take days, sometimes weeks. Manage cash flow explicitly:

**Track receivables aging**:
- Current (0-15 days): Money you should have received
- Overdue 15-30 days: Follow up with client
- Overdue 30+ days: Escalate (phone call if relationship allows)

Create a simple aging report weekly:

```markdown
## Receivables Aging - March 20, 2026

| Invoice | Amount | Days Overdue | Status |
|---------|--------|-------------|--------|
| INV-2026-0301-001 | $3,500 | 19 | Current |
| INV-2026-0228-001 | €4,200 | 5 | Current |
| INV-2026-0215-001 | $5,000 | 35 | OVERDUE - follow up needed |
```

When invoices are overdue, follow up immediately. The longer you wait, the less likely you'll collect.

**Payment deposit practices**:
- When payment lands in Wise/bank, convert to your home currency immediately (don't speculate on exchange rates)
- Transfer to your main business account the same week (don't let money sit)
- Record the converted amount in your tracker (this is your actual revenue)

Consistent deposit practices prevent the cash-in-hand feeling that leads to under-reporting income.

## Handling Disputes and Late Payments

Even with good processes, occasionally clients don't pay on time or question invoices.

**If client disputes the amount**:
1. Ask for specifics: "Which line item are you questioning?"
2. Reference your original agreement/scope
3. Provide evidence: "You approved this feature set on [date]"
4. Offer solution: "If we missed something, let's discuss" (not "let me reduce the price")

**If payment is late** (more than 5 days after due date):
1. Day 6: Send friendly reminder
2. Day 15: More formal follow-up with late fees
3. Day 30: Phone call or escalation
4. Day 45: Legal recovery (often not worth it for individuals)

Most late payments resolve in the 15-30 day window once you demonstrate you track payments seriously.

**Building late-payment protection**:
- On your invoice, state clearly: "Late payments subject to 1.5% monthly interest"
- For first-time clients, consider Net 15 instead of Net 30
- For risky jurisdictions, consider 50% upfront + 50% on delivery

## Multi-Currency Risk Management

Working internationally exposes you to exchange rate volatility. Manage risk:

**Conservative approach**: Invoice in your home currency. Clients bear the conversion cost. You never worry about rate movements.

**Aggressive approach**: Invoice in client currency, lock in favorable rates. Requires comfortable relationships and more administrative work.

**Hybrid approach** (recommended for most):
- Invoice in your home currency by default
- For large contracts (>$10,000), offer client currency option
- Use Wise forward contracts to lock in favorable rates if client requests their currency

Example: Large client in EU requests €15,000 invoice. You:
1. Quote EUR 15,000 = USD 16,500 (at current rate)
2. Use Wise to forward-contract that rate for 30 days
3. If rate moves favorably before payment, pocket the difference
4. If rate moves unfavorably, your locked rate protects you

Wise's forward contracts cost nothing; they just hold the rate. This is free exchange rate insurance.

## Annual Tax Reporting for International Invoicing

Year-end, you'll need to report international income to tax authorities. Organize your data:

**By jurisdiction**:
```
2026 Income Summary

USA Clients:
- Total invoiced: $45,000
- Total paid: $43,500
- Unpaid: $1,500

EU Clients:
- Total invoiced: €32,000 (converted to $35,200)
- Total paid: €30,000 (converted to $33,000)
- Unpaid: €2,000

Canada Clients:
- Total invoiced: CAD $18,000 (converted to $13,200)
- Total paid: CAD $18,000
- Unpaid: $0
```

This summary helps your accountant identify:
- Which countries you derive income from (affects tax obligations)
- Your bad debt situation (unpaid invoices can be deducted in some jurisdictions)
- Currency exposure if applicable

Keep detailed records. Most countries require invoices and payment receipts as backup.

## Final Recommendations

Consistent invoice numbering, organized client records, and a centralized payment tracker are the foundation. Clarify payment terms before starting work, and consult a tax professional about your specific situation—tax rules vary significantly based on your home country, client locations, and the nature of your services.

The invoicing workflow itself is simple; the complexity lies in managing international payment timing, currency conversion, and tax implications. Automating what you can and documenting everything else keeps the administrative overhead minimal while ensuring compliance and healthy cash flow.

## Frequently Asked Questions

**Who is this article written for?**

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.

**How current is the information in this article?**

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.

**Are there free alternatives available?**

Free alternatives exist for most tool categories, though they typically come with limitations on features, usage volume, or support. Open-source options can fill some gaps if you are willing to handle setup and maintenance yourself. Evaluate whether the time savings from a paid tool justify the cost for your situation.

**How do I get started quickly?**

Pick one tool from the options discussed and sign up for a free trial. Spend 30 minutes on a real task from your daily work rather than running through tutorials. Real usage reveals fit faster than feature comparisons.

**What is the learning curve like?**

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.

## Related Articles

- [Daily Workflow for a Solo Remote Technical Writer 2026](/remote-work-tools/daily-workflow-for-a-solo-remote-technical-writer-2026/)
- [Best Free Tools for Solo Developer Managing Side Projects](/remote-work-tools/best-free-tools-for-solo-developer-managing-side-projects-re/)
- [CI/CD Pipeline for Solo Developers: GitHub Actions](/remote-work-tools/ci-cd-pipeline-solo-developer-github-actions/)
- [Project Management for a Solo Developer with 8 Client](/remote-work-tools/project-management-for-a-solo-developer-with-8-client-projec/)
- [Remote Developer Code Review Workflow Tools for Teams](/remote-work-tools/remote-developer-code-review-workflow-tools-for-teams-without-synchronous-overlap/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
