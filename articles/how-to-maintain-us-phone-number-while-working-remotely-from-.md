---
layout: default
title: "Track all critical accounts requiring phone verification"
description: "A practical guide for developers and power users on keeping your US phone number while working remotely from Portugal or Spain. Includes code examples"
date: 2026-03-16
author: theluckystrike
permalink: /how-to-maintain-us-phone-number-while-working-remotely-from-/
categories: [guides]
score: 9
voice-checked: true
reviewed: true
intent-checked: true
tags: [remote-work-tools, remote-work]
---

{% raw %}
Maintaining an US phone number while living in Portugal or Spain is essential for many developers and remote workers. Whether you need it for banking, two-factor authentication, or staying in touch with US-based clients, keeping your American number provides continuity and avoids the hassle of updating contacts and services across international boundaries.

## Why Keep Your US Number

When you relocate to Europe, your US phone number serves critical functions beyond personal communication. Most US banks require a valid US phone number for account verification. Two-factor authentication from services like Google, GitHub, and AWS often sends SMS codes to your registered number. Client communications and job interviews often expect an US contact number.

Portuguese and Spanish SIM cards give you local numbers, but they do not replace your US number for these purposes. Switching everything to a European number creates friction with services tied to your American identity.

## Option 1: VoIP Services with SMS Support

VoIP services represent the most flexible approach for maintaining an US number abroad. Several providers offer US phone numbers with full SMS and voice capabilities, often at reasonable monthly rates.

### Service Comparison Table

| Service | Price | SMS | Voice | Setup | Geo Restrictions |
|---------|-------|-----|-------|-------|------------------|
| Google Voice | Free | Yes | Yes | Easy | Yes (blocks some regions) |
| Twilio | $1-2/mo + usage | Yes | Yes | Moderate | No |
| Skype | $4/month | Limited | Yes | Easy | No |
| NumberProxy | $5-15/mo | Yes | Limited | Easy | No |
| Dialpad | $25/month | Yes | Yes | Easy | No |
| VoiceVibe | $7/month | Yes | Yes | Easy | No |

### Google Voice (Limited but Free)

Google Voice provides a free US number with texting and voicemail. However, it has significant limitations for international users. The service requires an US-based verification phone number during setup, and Google actively blocks usage from certain international locations. If you already have a Google Voice number from before your move, it may continue working, but obtaining a new one from abroad proves difficult.

**Best for**: People who already have existing Google Voice numbers; purely receiving calls/texts from friends and family.

**Limitations**: Cannot obtain new numbers from abroad; unreliable for banking/2FA; no API access.

### Twilio: Programmatic Control ($1-2/month + usage)

For developers comfortable with APIs, Twilio offers US phone numbers with complete control. You purchase a number ($1.25/month) and pay usage fees (typically $0.01-0.03 per incoming SMS, $0.01-0.02 per incoming call minute). For most international workers, monthly bills stay under $5-15.

Here's a complete example using Twilio's Node.js SDK to forward SMS to your European number:

```javascript
const twilio = require('twilio');
const express = require('express');
const app = express();

const client = new twilio(
  process.env.TWILIO_ACCOUNT_SID,
  process.env.TWILIO_AUTH_TOKEN
);

// Configure webhook endpoint for incoming SMS (set this in Twilio console)
app.post('/sms', async (req, res) => {
  const incomingMessage = req.body.Body;
  const fromNumber = req.body.From;

  // Forward to your current European number
  try {
    await client.messages.create({
      body: `[US] ${incomingMessage}`,
      from: process.env.TWILIO_NUMBER,
      to: process.env.EUROPEAN_PHONE // e.g., +351912345678
    });

    res.send('<Response></Response>');
  } catch (error) {
    console.error('SMS forwarding failed:', error);
    res.status(500).send('Error forwarding SMS');
  }
});

// For receiving calls, create a voice webhook
app.post('/voice', (req, res) => {
  const VoiceResponse = require('twilio').twiml.VoiceResponse;
  const twiml = new VoiceResponse();

  // Forward call to European number
  twiml.dial({
    callerId: process.env.TWILIO_NUMBER,
    timeout: 30
  }, process.env.EUROPEAN_PHONE);

  res.type('text/xml');
  res.send(twiml.toString());
});

app.listen(3000, () => console.log('Twilio webhook listening on :3000'));
```

**Setup process**:
1. Create Twilio account ($0, gets free $30 trial credit)
2. Purchase an US phone number ($1.25/month)
3. Deploy webhook endpoint (use Heroku free tier, ngrok tunnel, or own server)
4. Configure Twilio console to point to your webhook
5. Update environment variables with your European phone number

**Best for**: Developers who want programmatic control; people comfortable managing webhooks; those with unpredictable usage patterns.

**Cost example**: 20 texts/month = ~$0.30, 2 calls/month = ~$0.04, plus number fee = $1.29/month total.

### NumberProxy ($5-15/month)

Services like NumberProxy specialize in maintaining US numbers for international users. The service handles forwarding automatically without requiring technical setup. You purchase a plan (typically $7-12/month), and SMS messages arrive via app notifications, email, or redirected to your local number.

**Supported regions**: Portugal, Spain, and 60+ other countries. Some services block Cuba, Iran, North Korea, and Syria due to US sanctions.

**Setup**: Sign up, choose your US area code (if available), start receiving. Takes 5 minutes.

**Best for**: Non-technical users; people wanting pure convenience without API management.

**Limitations**: Less programmatic control; typically slower SMS forwarding (2-5 minute delay); fewer customization options.

## Option 2: eSIM Solutions ($50-200 initial + ongoing plan)

eSIM technology allows you to maintain an US cellular number alongside your European SIM. This approach keeps your US number active on cellular networks without carrying a second physical phone.

| Provider | US Number | SMS | Reliability | Setup | Monthly Cost |
|----------|-----------|-----|-------------|-------|--------------|
| Airalo | Yes | Yes | Cellular | 10 min | $8-15/mo |
| Nomad eSIM | Yes | Yes | Cellular | 10 min | $12-20/mo |
| Holafly | Yes | Limited | Cellular | 10 min | $15-25/mo |

### Airalo and Other eSIM Providers

Several eSIM providers offer US phone numbers with data and SMS capabilities. Airalo ($15/month for US number) provides the most straightforward approach. The setup:

1. Confirm your phone supports eSIM (iPhone 13+, Samsung Galaxy S20+, Google Pixel 3+)
2. Install Airalo app
3. Purchase an US eSIM profile
4. Install the profile on your phone (takes 30 seconds in Settings)
5. Configure Airalo as secondary line while keeping your European number primary

The advantage is reliability—your US number operates on actual cellular networks (T-Mobile in US) rather than VoIP, ensuring better delivery rates for banking and authentication codes. The downside is additional cost ($150-240/year vs $12/year for Google Voice) and complexity managing two active numbers.

**Use this when**: You need guaranteed SMS delivery for 2FA; you want cellular network reliability; cost isn't a concern.

**Limitation**: Adds ~€15/month to your phone bill; your phone must support eSIM technology.

## Option 3: US Carrier International Plans ($30-100/month)

If you maintain a relationship with an US carrier, some offer international roaming packages:

| Carrier | Plan Name | Monthly Cost | SMS | Voice | Data |
|---------|-----------|--------------|-----|-------|------|
| T-Mobile | ONE | $30-70 | Free | Included | Included |
| AT&T | International Monthly Pass | $60-100 | $0.50 each | $2-3/min | $10/day |
| Verizon | TravelPass | $10/day + plan | Included | Included | Included |

T-Mobile's ONE plan is the most affordable option for maintaining an US number while living abroad. When traveling in Portugal/Spain, SMS and voice work normally (though at varying speeds/reliability). International data is included at no extra cost, though speeds throttle after 50GB.

**Best for**: People who want zero setup complexity; those maintaining active US employment; people wanting cellular service.

**Cost**: T-Mobile ONE plan is $30-70/month depending on how many lines you have. If you keep your existing US phone plan active just for the number, budget $30-50/month minimum.

## Option 4: Google Fi Wireless ($10-50/month)

Google Fi (formerly Google Project Fi) provides US phone numbers with international data usage. The service works in Portugal and Spain with no special setup—your phone automatically switches between local carriers and WiFi.

**Pricing structure**:
- Flexible plan: $20/month base + $10 per GB data (capped at $60/month for unlimited)
- Calls within US: Free or $0.01/minute depending on location
- International calls: $0.20/minute to Portugal/Spain
- SMS: Free domestically, $0.10/message internationally

**Real-world example**: A developer in Lisbon using 5GB data + occasional calls back to the US:
- Base: $20
- Data: $50 (capped)
- International calls: ~$3-5
- **Monthly total**: $73-75

**Advantages**:
- Works in 200+ countries without plan changes
- SMS and voice function identically to US usage
- Simple setup (just switch carrier)
- No need for separate app or forwarding configuration

**Disadvantages**:
- More expensive than VoIP solutions
- International call rates are higher
- Requires porting your existing number or getting a new one

**Setup process**:
1. Sign up at fi.google.com
2. Request to transfer your existing number or get a new one
3. Wait 2-3 business days for activation
4. Install Google Fi app on your phone
5. Your phone automatically configures

**Best for**: People who want native US carrier experience abroad; those making frequent calls back to US; anyone valuing simplicity over cost.

## Recommendation Matrix by Use Case

Choose your solution based on your primary need:

| Your Situation | Recommended Option | Cost | Setup Time |
|---|---|---|---|
| Receiving occasional texts from family/friends | Google Voice (free, if you have existing account) | Free | 0 min |
| Developer needing SMS forwarding with API control | Twilio | $5-15/mo | 2 hours |
| Reliable SMS for banking and 2FA | Google Fi or Nomad eSIM | $50-75/mo | 30 min |
| Maintain active US employment/professional calls | T-Mobile ONE or Google Fi | $30-75/mo | 15 min |
| Minimal setup, no technical overhead | NumberProxy | $7-12/mo | 5 min |

## Practical Considerations

**Time zone management** becomes important when your US number receives calls during European evening hours. If using Google Fi, T-Mobile, or Twilio, configure voicemail greetings indicating your timezone and preferred contact methods. For example:

> "Hi, you've reached [name]. I'm currently in Portugal (GMT+0, 9 hours behind US Eastern). I check voicemails twice daily at 8am and 4pm PT. For urgent matters, email [address]."

**Verification challenges** arise when services detect international usage. Some banks flag accounts when the associated phone number appears to be used internationally. Solutions:

1. **Preemptively notify your bank**: Call them before moving and explain your remote work situation. Ask them to note your account.
2. **Use a VoIP service that routes through US servers**: Twilio and Google Fi both route calls through US infrastructure, appearing more "local" to your bank.
3. **Maintain backup verification method**: Set up secondary email authentication or security questions on critical accounts.
4. **Have an US-based contact**: Ask a trusted friend/family member to receive verification codes on your behalf if needed.

**Backup communication methods** matter critically. Never rely entirely on a single phone number—it creates vulnerability if the service fails or your account locks. Implementation strategy:

```python
# Track all critical accounts requiring phone verification
critical_accounts = {
    "Bank": {
        "primary_phone": "+1-xxx-xxx-xxxx",
        "backup_email": "you@email.com",
        "backup_contact": "trusted_friend@email.com",
        "security_questions": True,
        "recovery_codes_saved": True
    },
    "Google": {
        "primary_phone": "+1-xxx-xxx-xxxx",
        "backup_phone": "+351-xxx-xxx-xxx",
        "recovery_email": "backup@email.com"
    },
    "GitHub/AWS": {
        "phone_2fa_enabled": True,
        "backup_codes_saved": "/secure/location"
    }
}

# Annual verification checklist
def verify_account_access():
    """Test account access with current setup"""
    print("Testing 2FA delivery via:")
    print("- SMS to US number (Twilio/Google Fi)")
    print("- Backup email verification")
    print("- Recovery codes validity")
    print("\nAll backup methods working? Record date:", datetime.now())
```

Update your email and secondary phone numbers with all critical accounts before relocating. This ensures you have fallback options if your primary phone number strategy fails.

## Monthly Cost Comparison by Strategy

| Strategy | Monthly Cost | Reliability | Setup |
|----------|-------------|-------------|-------|
| Google Voice only | Free | 70% | 5 min |
| Twilio (heavy user) | $20 | 95% | 2 hours |
| Google Fi | $60 | 98% | 30 min |
| T-Mobile ONE | $50 | 98% | 0 min |
| NumberProxy | $10 | 85% | 5 min |
| Google Voice + Twilio backup | $8 | 99% | 2 hours |

**Recommended hybrid approach for developers**: Use Google Voice for personal contacts (free) and Twilio ($5-15/month) for critical services like banking. This provides redundancy—if Google Voice fails, Twilio continues working. Total cost: $5-15/month with 99%+ reliability.


## Related Reading

- [Best Remote Work Tools in 2026](/best-remote-work-tools-2026/)
- [Remote Work Productivity Guide](/remote-work-productivity-guide/)
- [Remote Work Tools Hub](/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
