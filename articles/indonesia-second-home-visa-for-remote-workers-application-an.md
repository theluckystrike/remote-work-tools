---
layout: default
title: "Indonesia Second Home Visa for Remote Workers"
description: "A practical guide for developers and power users on Indonesia's Second Home Visa for remote workers. Complete application process, requirements, financial"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /indonesia-second-home-visa-for-remote-workers-application-an/
categories: [guides]
tags: [remote-work-tools, indonesia, second-home-visa, remote-work, digital-nomad, visa-guide, indonesian-visa]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}
# Indonesia Second Home Visa for Remote Workers: Application and Requirements Guide 2026

Indonesia introduced the Second Home Visa (Visa Tinggal Terbatas dengan注 sponsor Tinggal Tetap) specifically to attract remote workers, digital nomads, and long-term visitors who want to live in Indonesia without requiring local employment. Unlike the B211A tourist/business visa that requires periodic extensions, the Second Home Visa offers validity for 5 to 10 years with multiple entry privileges. This guide covers the complete application process, financial requirements, document preparation, and practical tools for developers planning a move to Indonesia.

## Eligibility Criteria for the Second Home Visa

The Indonesian Immigration Directorate General has established clear eligibility requirements for Second Home Visa applicants. Understanding these upfront prevents application rejections and wasted processing fees.

### Core Requirements

1. Passport validity: Minimum 6 months remaining from the application date
2. Financial proof: Bank statements showing IDR 1.5 billion (approximately $93,000 USD) in savings, OR proof of monthly income equivalent to IDR 250 million ($15,500 USD)
3. Clean criminal record: Certificate from country of origin
4. Health insurance: Valid international health insurance covering Indonesia for the visa duration
5. Sponsor requirement: Either an Indonesian sponsor (individual or corporate) or self-sponsorship with additional documentation

The financial requirements represent the primary barrier for most applicants. The IDR 1.5 billion threshold applies to individual applicants without Indonesian sponsorship. If you secure an Indonesian corporate sponsor, the requirements may be reduced.

### Documentation Checklist

Gathering documents before starting your application significantly accelerates the process:

- Valid passport (PDF scan, all pages)
- Recent passport-sized photographs (white background, 4x6cm)
- Bank statement (original, stamped by bank, last 3 months)
- Employment contract or business registration (if self-employed)
- Criminal background check (apostilled, translated to Indonesian or English)
- Curriculum vitae/resume
- Proof of accommodation (rental agreement or property ownership in Indonesia)
- Sponsorship letter (if applicable)

## Application Process: Step-by-Step

The Second Home Visa application submits through Indonesia's online immigration portal (https://visa-online.imigration.go.id/). Here's the practical workflow:

### Step 1: Account Creation and Initial Application

Create an account on the immigration portal and select "Second Home Visa (Visa Tinggal Terbatas)" as your application type. The system requires email verification and identity document upload.

```bash
# Prepare your documents in advance
# Required formats: PDF, max 2MB per file
# Naming convention: PASSPORT_[name].pdf, BANK_[name].pdf
```

### Step 2: Sponsor Verification (If Applicable)

If applying with an Indonesian sponsor, their details enter the system first. The sponsor must provide:
- KTP (Indonesian ID card)
- KK (Family card)
- Proof of residence
- Sponsorship letter (signed, notarized)

Self-sponsored applicants skip this step but must demonstrate higher financial standing.

### Step 3: Financial Document Upload

Upload bank statements demonstrating the required IDR 1.5 billion balance. The system accepts statements from international banks but requires English or Indonesian language. For cryptocurrency holders, converting to fiat and maintaining the balance for 3+ months before application strengthens the case.

```python
# Calculate your financial eligibility
def check_indonesia_second_home_visa_eligibility(balance_idr, monthly_income_idr=None):
    REQUIRED_BALANCE = 1_500_000_000  # IDR 1.5 billion
    REQUIRED_MONTHLY_INCOME = 250_000_000  # IDR 250 million

    if balance_idr >= REQUIRED_BALANCE:
        return "Eligible via savings path"
    elif monthly_income_idr and monthly_income_idr >= REQUIRED_MONTHLY_INCOME:
        return "Eligible via income path"
    else:
        return "Not eligible - increase funds or secure sponsor"

# Example check
result = check_indonesia_second_home_visa_eligibility(2_000_000_000)
print(result)  # "Eligible via savings path"
```

### Step 4: Interview Scheduling

After document review, the immigration system schedules a virtual interview. This 15-20 minute video call verifies your identity and intended activities in Indonesia. Common questions include:
- Purpose of stay
- Planned duration
- Accommodation arrangements
- Financial source verification
- Intent to work (Second Home Visa does NOT permit local employment)

### Step 5: Visa Approval and Arrival

Upon approval, you receive an electronic Visa Grant Notice (VGN). Print this and present it upon arrival in Indonesia. At the airport, immigration officers issue a second home stay permit (ITAS) valid for the visa duration.

## Financial Planning Tools

For developers and digital nomads, several tools help track the financial requirements:

### Banking Documentation Tips

Indonesian immigration accepts statements from major international banks. If your primary bank doesn't support IDR accounts, maintain a separate account statement showing the equivalent USD or EUR balance. The exchange rate used for calculation follows Bank Indonesia's published rates at application time.

```javascript
// Currency conversion helper for eligibility check
const IDR_EXCHANGE_RATE = 16100; // USD to IDR (approximate)

const REQUIRED_IDR = 1_500_000_000;
const REQUIRED_USD = REQUIRED_IDR / IDR_EXCHANGE_RATE;

console.log(`Required: $${REQUIRED_USD.toLocaleString()} USD`);
```

### Insurance Requirements

Health insurance must cover the entire visa duration. Indonesian immigration accepts international policies from providers like Allianz, IMG, or Pacific Cross. Ensure your policy:
- Covers Indonesia explicitly (not just "Southeast Asia")
- Provides minimum $100,000 coverage
- Allows for visa application purposes (letter from insurer confirming coverage)

## Practical Considerations for Remote Workers

The Second Home Visa specifically targets long-term visitors who will not engage in local employment. However, it permits:
- Remote work for overseas employers
- Running online businesses
- Passive income activities
- Investment activities

This makes it ideal for developers working with international clients, startup founders running remote-first companies, and digital product creators earning through global platforms.

### Tax Implications

Indonesia does not tax foreign-sourced income for individuals without tax residency. However, if you spend more than 183 days in Indonesia within a 12-month period, you may be considered a tax resident. The Second Home Visa holders should consult with Indonesian tax advisors for personalized guidance.

```bash
# Calculate potential tax residency days
# Stay under 183 days to avoid Indonesian tax residency on foreign income
# Track days carefully using a simple script
```

## Common Application Issues and Solutions

### Rejection Reasons

1. Insufficient funds: Appeal with additional bank statements or obtain sponsorship
2. Sponsor issues: Verify sponsor's legitimacy; immigration verifies sponsors
3. Document translation: Ensure all documents in English or provide certified translations
4. Interview no-show: Reschedule through the portal; no-shows delay processing

### Processing Times

Standard processing takes 5-10 business days. Expedited processing (2-3 days) available for additional fees. During peak periods (December-January, July-August), expect delays.

## Alternative Visa Options

If the Second Home Visa requirements exceed your current situation, alternatives include:
- B211A Visa: Tourist/business visa, extendable to 6 months, no financial requirements
- KITAP (Permanent Residency): Requires 5+ years on dependent visa or investment > $1M
- Digital Nomad Visa (currently in pilot): Newer option with simpler requirements


## Related Articles

- [Example NHI enrollment at a local district office](/remote-work-tools/taiwan-gold-card-visa-for-remote-tech-workers-application-pr/)
- [Czech Republic Digital Nomad Visa (Zivno) Application Guide](/remote-work-tools/czech-republic-digital-nomad-visa-zivno-application-for-remote-freelancers-guide-2026/)
- [Hungary Digital Nomad Visa White Card Application for](/remote-work-tools/hungary-digital-nomad-visa-white-card-application-for-remote/)
- [Montenegro Digital Nomad Visa Application Process for](/remote-work-tools/montenegro-digital-nomad-visa-application-process-for-remote/)
- [Document checklist with recommended file names](/remote-work-tools/colombia-digital-nomad-visa-application-process-for-software/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
