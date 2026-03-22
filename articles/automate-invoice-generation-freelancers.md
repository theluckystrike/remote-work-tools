---
layout: default
title: "Automate Invoice Generation for Freelancers"
description: "Automate invoice creation, numbering, and delivery using scripts and free tools. Covers CLI invoice generation, PDF output, email delivery, and payment"
date: 2026-03-21
author: theluckystrike
permalink: /automate-invoice-generation-freelancers/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Manual invoicing is a tax on your time. If you spend 15 minutes creating and sending each invoice and you bill 10 clients a month, that is 2.5 hours of admin per month — roughly 30 hours a year. Automating invoice generation brings that to under 30 seconds per invoice.

## Table of Contents

- [The Stack](#the-stack)
- [Install invoice-cli](#install-invoice-cli)
- [Project Structure](#project-structure)
- [Your Business Config](#your-business-config)
- [Client Config](#client-config)
- [Automation Script](#automation-script)
- [Install Dependencies](#install-dependencies)
- [Send an Invoice](#send-an-invoice)
- [Mark an Invoice Paid](#mark-an-invoice-paid)
- [Automate Monthly Recurring Invoices](#automate-monthly-recurring-invoices)
- [Self-Hosted Alternative: Invoice Ninja](#self-hosted-alternative-invoice-ninja)
- [Adding Tax Calculation](#adding-tax-calculation)
- [Late Payment Follow-Up Automation](#late-payment-follow-up-automation)
- [Generating Annual Revenue Reports](#generating-annual-revenue-reports)
- [Comparison of Invoice Automation Approaches](#comparison-of-invoice-automation-approaches)
- [Related Reading](#related-reading)

This guide builds a practical invoice automation system using scripts: a CLI tool that generates numbered PDFs, sends them by email, and logs payment status in a CSV.

## The Stack

- **invoice-cli** — Node.js CLI that generates PDF invoices from YAML data
- **Nodemailer** — email delivery
- **CSV ledger** — simple payment tracking without a SaaS subscription

## Install invoice-cli

```bash
npm install -g invoice-cli

# Verify install
invoice --version
```

invoice-cli takes a YAML config and outputs a styled PDF. It handles itemized line items, tax rates, due dates, and client data.

## Project Structure

```bash
~/invoices/
├── config/
│   └── me.yml          # your business details
├── clients/
│   ├── acme.yml
│   └── globex.yml
├── output/             # generated PDFs land here
├── ledger.csv          # payment tracking
└── send-invoice.js     # automation script
```

## Your Business Config

```yaml
# config/me.yml
business:
  name: "Your Name / Studio Name"
  address: "123 Main St, City, State 00000"
  email: "billing@yourdomain.com"
  website: "https://yourdomain.com"
  payment_terms: "Net 30"
  tax_id: "XX-XXXXXXX"
```

## Client Config

```yaml
# clients/acme.yml
client:
  name: "ACME Corp"
  contact: "Jane Smith"
  email: "jane@acme.com"
  address: "456 Client Ave, New York, NY 10001"
  currency: "USD"
  tax_rate: 0
```

## Automation Script

```javascript
// send-invoice.js
const { execSync } = require("child_process");
const nodemailer = require("nodemailer");
const fs = require("fs");
const path = require("path");
const yaml = require("js-yaml");

const args = process.argv.slice(2);
// Usage: node send-invoice.js acme "Website redesign" 4500 1 --send

const [clientSlug, description, rate, quantity, flag] = args;
const send = flag === "--send";

const me = yaml.load(fs.readFileSync("config/me.yml", "utf8"));
const client = yaml.load(fs.readFileSync(`clients/${clientSlug}.yml`, "utf8"));

// Auto-increment invoice number from ledger
const ledgerPath = "ledger.csv";
const ledger = fs.existsSync(ledgerPath)
  ? fs.readFileSync(ledgerPath, "utf8").split("\n").filter(Boolean)
  : [];
const nextNumber = ledger.length + 1001;
const invoiceNumber = `INV-${nextNumber}`;

// Dates
const today = new Date();
const due = new Date(today);
due.setDate(due.getDate() + 30);
const fmt = (d) => d.toISOString().split("T")[0];

// Build and write invoice YAML
const invoiceData = {
  number: invoiceNumber,
  date: fmt(today),
  due_date: fmt(due),
  items: [{ description, rate: parseFloat(rate), quantity: parseFloat(quantity) }],
};

const tmpFile = `/tmp/invoice-${invoiceNumber}.yml`;
fs.writeFileSync(tmpFile, yaml.dump({ ...me, ...client, invoice: invoiceData }));

// Generate PDF
const outputPath = `output/${invoiceNumber}-${clientSlug}.pdf`;
execSync(`invoice generate ${tmpFile} --output ${outputPath}`);
console.log(`Generated: ${outputPath}`);

// Log to ledger
const total = parseFloat(rate) * parseFloat(quantity);
fs.appendFileSync(ledgerPath,
  `${invoiceNumber},${clientSlug},${fmt(today)},${fmt(due)},${total},pending\n`);

// Send email
if (send) {
  const transporter = nodemailer.createTransport({
    host: process.env.SMTP_HOST,
    port: 587,
    auth: { user: process.env.SMTP_USER, pass: process.env.SMTP_PASS },
  });

  transporter.sendMail({
    from: me.business.email,
    to: client.client.email,
    subject: `Invoice ${invoiceNumber} from ${me.business.name}`,
    text: `Hi ${client.client.contact},\n\nPlease find attached invoice ${invoiceNumber} for ${description}.\n\nAmount due: $${total.toFixed(2)}\nDue date: ${fmt(due)}\n\nThank you,\n${me.business.name}`,
    attachments: [{ filename: path.basename(outputPath), path: outputPath }],
  });
  console.log(`Sent to ${client.client.email}`);
}
```

## Install Dependencies

```bash
cd ~/invoices
npm init -y
npm install nodemailer js-yaml

# Set SMTP credentials as environment variables
export SMTP_HOST=smtp.gmail.com
export SMTP_USER=you@gmail.com
export SMTP_PASS=your-app-password
```

For Gmail, generate an App Password at myaccount.google.com/apppasswords with "Mail" access. This is separate from your main Gmail password and can be revoked independently.

## Send an Invoice

```bash
# Generate only (review before sending)
node send-invoice.js acme "Website redesign phase 1" 4500 1

# Generate and send immediately
node send-invoice.js acme "Website redesign phase 1" 4500 1 --send

# Check outstanding invoices
grep ',pending' ledger.csv
```

## Mark an Invoice Paid

```bash
# Mark INV-1003 as paid
node -e "
const fs = require('fs');
const id = process.argv[1];
const data = fs.readFileSync('ledger.csv','utf8');
fs.writeFileSync('ledger.csv', data.replace(new RegExp('(' + id + ',.*),pending'), '\$1,paid'));
" INV-1003
```

## Automate Monthly Recurring Invoices

For retainer clients billed every month, use a cron job:

```bash
# crontab -e
# Run on the 1st of each month at 8am
0 8 1 * * cd ~/invoices && node send-invoice.js globex "Monthly retainer" 200 10 --send >> ~/invoices/logs/cron.log 2>&1
```

## Self-Hosted Alternative: Invoice Ninja

For a full GUI with online payment and client portal, Invoice Ninja is free when self-hosted:

```bash
docker run -d \
  --name invoiceninja \
  -e APP_KEY=$(openssl rand -base64 32) \
  -e APP_URL=http://localhost \
  -p 80:80 \
  invoiceninja/invoiceninja:5
```

Invoice Ninja adds recurring invoice automation, Stripe/PayPal payment links, and client portal access — all without a monthly SaaS fee.

## Adding Tax Calculation

Different clients may require different tax rates depending on jurisdiction. Extend the invoice system:

```javascript
// tax-config.js
const TAX_RATES = {
  US: { CA: 0.0725, NY: 0.08, TX: 0.0625, OR: 0 },
  EU: { DE: 0.19, FR: 0.20, NL: 0.21 },
  DEFAULT: 0
};

function calculateTax(subtotal, country, state) {
  if (country === 'US') return subtotal * (TAX_RATES.US[state] || 0);
  if (country === 'EU') return subtotal * (TAX_RATES.EU[state] || 0);
  return subtotal * TAX_RATES.DEFAULT;
}

module.exports = { calculateTax, TAX_RATES };
```

## Late Payment Follow-Up Automation

Unpaid invoices require follow-up. Automate reminders based on due date:

```javascript
// check-overdue.js
const fs = require("fs");

function getOverdueInvoices() {
  const ledger = fs.readFileSync("ledger.csv", "utf8")
    .split("\n").filter(Boolean);
  const today = new Date();
  const overdue = [];
  for (const line of ledger) {
    const [id, client, issued, due, amount, status] = line.split(",");
    if (status === "pending" && new Date(due) < today) {
      const daysOverdue = Math.floor((today - new Date(due)) / (1000*60*60*24));
      overdue.push({ id, client, due, amount, daysOverdue });
    }
  }
  return overdue;
}

const overdue = getOverdueInvoices();
for (const inv of overdue) {
  console.log(`${inv.id}: ${inv.client} - $${inv.amount} - ${inv.daysOverdue} days overdue`);
  if ([7, 14, 30].includes(inv.daysOverdue)) {
    console.log(`  -> Sending ${inv.daysOverdue}-day reminder`);
  }
}
```

Run this daily via cron:

```bash
0 9 * * * cd ~/invoices && node check-overdue.js >> logs/overdue.log 2>&1
```

## Generating Annual Revenue Reports

At tax time, generate a summary report from your ledger:

```bash
#!/bin/bash
# annual-report.sh
YEAR=${1:-$(date +%Y)}
echo "=== Revenue Report for $YEAR ==="
echo "Total Revenue:"
grep ",$YEAR-" ledger.csv | grep ",paid" | \
  awk -F',' '{sum += $5} END {printf "$%.2f\n", sum}'
echo ""
echo "Outstanding Invoices:"
grep ",$YEAR-" ledger.csv | grep ",pending" | \
  awk -F',' '{sum += $5; count++} END {printf "%d invoices, $%.2f total\n", count, sum}'
echo ""
echo "Revenue by Client:"
grep ",$YEAR-" ledger.csv | grep ",paid" | \
  awk -F',' '{clients[$2] += $5} END {for (c in clients) printf "  %-20s $%.2f\n", c, clients[c]}'
```

## Comparison of Invoice Automation Approaches

| Approach | Setup Time | Monthly Cost | Customization | Payment Links |
|----------|-----------|-------------|---------------|---------------|
| CLI Scripts (this guide) | 2-3 hours | $0 | Full control | Manual |
| Invoice Ninja (self-hosted) | 1 hour | $5-10 (hosting) | High | Stripe/PayPal |
| FreshBooks | 15 minutes | $17/month | Limited | Built-in |
| Wave | 15 minutes | Free | Limited | Built-in |
| Xero | 30 minutes | $15/month | Moderate | Built-in |

The CLI approach works best for developers who want full control and already have a command-line workflow.

## Related Reading

- [Best Invoicing Tools for Freelancers 2026](/remote-work-tools/best-invoicing-tools-for-freelancers-2026/)
- [Best Accounting Software for Freelancers 2026](/remote-work-tools/best-accounting-software-for-freelancers-2026/)
- [Freelancer Tax Deduction Tracking Tools 2026](/remote-work-tools/freelancer-tax-deduction-tracking-2026/)

## Related Articles

- [Best Invoicing and Client Payment Portal for Remote Agencies](/remote-work-tools/best-invoicing-and-client-payment-portal-for-remote-agencies/)
- [Best Payment Collection Automation for Remote Businesses](/remote-work-tools/best-payment-collection-automation-for-remote-businesses-sending-invoice-reminders-2026/)
- [Best Tools for Managing Client Contracts Invoices Freelance](/remote-work-tools/best-tools-for-managing-client-contracts-invoices-freelance-developer/)
- [Automation Tools for Freelance Business Operations](/remote-work-tools/automation-tools-for-freelance-business-operations/)
- [Payment Terms Best Practices for Freelancers](/remote-work-tools/payment-terms-best-practices-for-freelancers/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

## Frequently Asked Questions

**Are there any hidden costs I should know about?**

Watch for overage charges, API rate limit fees, and costs for premium features not included in base plans. Some tools charge extra for storage, team seats, or advanced integrations. Read the full pricing page including footnotes before signing up.

**Is the annual plan worth it over monthly billing?**

Annual plans typically save 15-30% compared to monthly billing. If you have used the tool for at least 3 months and plan to continue, the annual discount usually makes sense. Avoid committing annually before you have validated the tool fits your needs.

**Can I change plans later without losing my data?**

Most tools allow plan changes at any time. Upgrading takes effect immediately, while downgrades typically apply at the next billing cycle. Your data and settings are preserved across plan changes in most cases, but verify this with the specific tool.

**Do student or nonprofit discounts exist?**

Many AI tools and software platforms offer reduced pricing for students, educators, and nonprofits. Check the tool's pricing page for a discount section, or contact their sales team directly. Discounts of 25-50% are common for qualifying organizations.

**What happens to my work if I cancel my subscription?**

Policies vary widely. Some tools let you access your data for a grace period after cancellation, while others lock you out immediately. Export your important work before canceling, and check the terms of service for data retention policies.

{% endraw %}
