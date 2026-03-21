---
layout: default
title: "Automate Invoice Generation for Freelancers"
description: "Automate invoice creation, numbering, and delivery using scripts and free tools. Covers CLI invoice generation, PDF output, email delivery, and payment tracking."
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

## Related Reading

- [Best Invoicing Tools for Freelancers 2026](/remote-work-tools/best-invoicing-tools-for-freelancers-2026/)
- [Best Accounting Software for Freelancers 2026](/remote-work-tools/best-accounting-software-for-freelancers-2026/)
- [Freelancer Tax Deduction Tracking Tools 2026](/remote-work-tools/freelancer-tax-deduction-tracking-2026/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
