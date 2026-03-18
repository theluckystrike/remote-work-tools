---
layout: default
title: "How to Maintain a US Phone Number While Working Remotely from Portugal or Spain"
description: "A practical guide for developers and power users on keeping your US phone number while working remotely from Portugal or Spain. Includes code examples and setup tips."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-maintain-us-phone-number-while-working-remotely-from-/
categories: [guides]
---

{% raw %}
Maintaining a US phone number while living in Portugal or Spain is essential for many developers and remote workers. Whether you need it for banking, two-factor authentication, or staying in touch with US-based clients, keeping your American number provides continuity and avoids the hassle of updating contacts and services across international boundaries.

## Why Keep Your US Number

When you relocate to Europe, your US phone number serves critical functions beyond personal communication. Most US banks require a valid US phone number for account verification. Two-factor authentication from services like Google, GitHub, and AWS often sends SMS codes to your registered number. Client communications and job interviews often expect a US contact number.

Portuguese and Spanish SIM cards give you local numbers, but they do not replace your US number for these purposes. Switching everything to a European number creates friction with services tied to your American identity.

## Option 1: VoIP Services with SMS Support

VoIP services represent the most flexible approach for maintaining a US number abroad. Several providers offer US phone numbers with full SMS and voice capabilities, often at reasonable monthly rates.

### Google Voice (Limited but Free)

Google Voice provides a free US number with texting and voicemail. However, it has significant limitations for international users. The service requires a US-based verification phone number during setup, and Google actively blocks usage from certain international locations. If you already have a Google Voice number from before your move, it may continue working, but obtaining a new one from abroad proves difficult.

### Twilio: Programmatic Control

For developers comfortable with APIs, Twilio offers US phone numbers with complete control. You purchase a number and configure it to forward SMS messages to your current phone number or handle them programmatically.

Here's a basic example using the Twilio Node.js SDK to forward incoming SMS:

```javascript
const twilio = require('twilio');
const client = new twilio(
  process.env.TWILIO_ACCOUNT_SID,
  process.env.TWILIO_AUTH_TOKEN
);

// Configure webhook endpoint for incoming SMS
app.post('/sms', async (req, res) => {
  const incomingMessage = req.body.Body;
  const fromNumber = req.body.From;
  
  // Forward to your current European number
  await client.messages.create({
    body: `US Number: ${incomingMessage}`,
    from: process.env.TWILIO_NUMBER,
    to: process.env.YOUR_CURRENT_NUMBER
  });
  
  res.send('<Response></Response>');
});
```

Twilio numbers cost approximately $1-2 per month for the number plus per-message fees. This approach gives you full control over message handling.

### NumberProxy and Similar Services

Services like NumberProxy specialize in maintaining US numbers for international users. These services typically forward SMS to your current number via app notifications or email. Pricing ranges from $5-15 monthly depending on features. Research current options carefully as the market evolves.

## Option 2: eSIM Solutions

eSIM technology allows you to maintain a US cellular number alongside your European SIM. This approach keeps your US number active on cellular networks without carrying a second physical phone.

### Airalo and Other eSIM Providers

Several eSIM providers offer US phone numbers with data and SMS capabilities. The setup process involves purchasing an eSIM profile and installing it on your compatible phone. You then maintain both your European and US numbers simultaneously.

The advantage here is reliability—your US number operates on actual cellular networks rather than VoIP, ensuring better delivery rates for banking and authentication codes. The downside is additional cost and managing two phone numbers on one device.

## Option 3: US Carrier International Plans

If you maintain a relationship with a US carrier, some offer international roaming packages. T-Mobile, AT&T, and Verizon provide plans that include international data and sometimes voice/SMS. However, these plans often prove expensive for long-term usage and may not support numbers for permanent overseas residence.

A more practical approach involves keeping your US number on a minimal prepaid plan with the carrier. Some carriers offer "keep your number" programs for customers traveling abroad, though terms vary and fees apply.

## Option 4: Google Fi with International Data

Google Fi (now called Google Fi Wireless) provides US phone numbers with international data usage. The service works in Portugal and Spain with reasonable rates. You receive a US number that functions identically to being in the United States, including SMS and voice.

The monthly cost covers data usage, and you can set up call forwarding to your European SIM if needed. This option balances convenience with cost-effectiveness for many remote workers.

## Configuration Example: Call Forwarding

Regardless of which option you choose, setting up call and SMS forwarding ensures you never miss important communications. Here's how you might configure call forwarding using a simple approach:

```bash
# Using Twilio to forward calls to your European number
# In Twilio console, set voice webhook to your handler

const VoiceResponse = require('twilio').twiml.VoiceResponse;

app.post('/voice', (req, res) => {
  const forwardingNumber = process.env.EUROPEAN_NUMBER;
  
  const twiml = new VoiceResponse();
  twiml.dial({ callerId: process.env.TWILIO_NUMBER }, forwardingNumber);
  
  res.type('text/xml');
  res.send(twiml.toString());
});
```

## Practical Considerations

**Time zone management** becomes important when your US number receives calls during European evening hours. Configure voicemail greetings indicating your availability, and consider setting business hours that overlap with US East Coast business hours when possible.

**Verification challenges** arise when services detect international usage. Some banks flag accounts when the associated phone number appears to be used internationally. In these cases, having a US-based contact who can verify codes on your behalf helps, or using a VoIP service that routes through US-based servers.

**Backup communication methods** matter. Always maintain alternative contact information with critical services, as relying entirely on a single phone number creates vulnerability. Update your email and secondary phone numbers with banks and important accounts.

## Summary

Keeping your US phone number while working from Portugal or Spain requires choosing between VoIP services like Twilio, eSIM solutions from providers like Airalo, carrier international plans, or Google Fi. Each option offers different tradeoffs between cost, reliability, and convenience. For developers comfortable with APIs, Twilio provides the most control. For simplicity, Google Fi or eSIM solutions work well. Evaluate your specific needs for banking, authentication, and client communication when making your choice.

The right solution depends on your technical comfort level and which services require your US number most urgently. Start with one approach and adjust as your needs become clearer.

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
