---
layout: default
title: "How to Maintain a US Phone Number While Working Remotely from Portugal or Spain"
description: "A practical guide for developers and power users on keeping your US phone number while working remotely from Portugal or Spain. Covers VoIP solutions, virtual numbers, SMS forwarding, and 2FA strategies."
date: 2026-03-16
author: "Remote Work Tools"
permalink: /how-to-maintain-us-phone-number-while-working-remotely-from-/
reviewed: true
score: 8
categories: [guides]
---

{% raw %}
# How to Maintain a US Phone Number While Working Remotely from Portugal or Spain

Working remotely from Portugal or Spain offers incredible lifestyle benefits—affordable cost of living, excellent weather, and access to the wider European continent. However, many remote workers face a common challenge: keeping their US phone number active for work, banking, and essential services.

This guide covers practical methods to maintain a US phone number while living abroad, focusing on solutions that work well for developers and power users who need reliability and technical flexibility.

## Why Keep Your US Number

Your US phone number serves more purposes than just making calls. Consider all the services linked to that number:

- **Two-factor authentication (2FA)** for banking, email, GitHub, AWS, and other critical accounts
- **SMS notifications** from payment processors and payroll systems
- **Business communications** with US-based clients and teams
- **Account recovery** options for various online services

When you leave the US without a forwarding solution, you lose access to these critical services. Many banks and financial institutions specifically require a US number for verification, and switching to a local number means updating dozens of accounts.

## Option 1: Google Voice (Limited but Free)

Google Voice remains the most accessible option, though it comes with restrictions you should understand.

**What works:**
- Incoming calls to your US number ring through to any phone or computer
- Voicemail transcription sends emails to your inbox
- Free to use for calls within the US

**What doesn't work:**
- No SMS sending to short codes (many banks use these)
- No verification code reception from certain services
- Requires a US-based phone number for initial setup (you'll need a friend or service to verify)

If you're just starting your remote journey and haven't left the US yet, setting up Google Voice before departure is worth considering. Once established, it can handle basic call forwarding indefinitely.

## Option 2: VoIP.ms (Professional Grade)

For developers who want full control, VoIP.ms provides a SIP-based solution with excellent flexibility.

### Setup Example

```bash
# Install a SIP client like MicroSIP on your laptop
# Or configure with asterisk/pjsip for custom solutions

# VoIP.ms account configuration:
Server: atlanta.voip.ms
Username: your_username
Password: your_secure_password
```

With VoIP.ms, you get:
- A dedicated US phone number
- Full SMS capability including short codes
- Call forwarding to any destination
- Detailed billing (pay per minute or monthly plans)

The monthly cost ranges from $2-5 USD for a basic US number, making it cost-effective for power users.

## Option 3: Twilio (Developer-Friendly API)

Twilio offers programmable phone numbers with API access, ideal for developers who want to build custom workflows.

### Basic Python Example for Call Forwarding

```python
from twilio.rest import Client

# Your credentials (store securely in environment variables)
account_sid = os.environ.get('TWILIO_ACCOUNT_SID')
auth_token = os.environ.get('TWILIO_AUTH_TOKEN')
client = Client(account_sid, auth_token)

# Configure call forwarding
call = client.calls.create(
    to='+351912345678',  # Your Portuguese number
    from_='+12025551234',  # Your US Twilio number
    url='http://demo.twilio.com/docs/voice.xml'
)
```

### Receiving SMS with a Webhook

```python
from flask import Flask, request

app = Flask(__name__)

@app.route('/sms', methods=['POST'])
def receive_sms():
    from_number = request.form['From']
    message_body = request.form['Body']
    
    # Process the message - forward to email, log to database, etc.
    print(f"Received SMS from {from_number}: {message_body}")
    
    return 'OK', 200
```

Twilio provides $15-20 in free credits for new accounts, enough to test thoroughly before committing. US numbers cost around $1/month plus usage charges.

## Option 4: Google Fi (Integrated Solution)

Google Fi works internationally with some caveats:

- Requires an Android phone or iPhone with eSIM support
- Works in Portugal and Spain but with data speed limitations
- Provides a US number that functions like a regular carrier

The advantage is simplicity—you get a single number that works everywhere. However, the monthly cost ($20+ base) plus international data fees adds up compared to VoIP solutions.

## Managing Two-Factor Authentication

This is where most remote workers encounter frustration. Many US services only send 2FA codes via SMS to US numbers. Here's how to handle it:

### Strategy 1: Use Authenticator Apps Wherever Possible

Migrate from SMS-based 2FA to authenticator apps (Google Authenticator, Authy, 1Password). These don't depend on phone numbers and work globally.

```bash
# Example: Set up CLI-based 2FA with oathtool
oathtool --totp -b $(cat ~/.2fa_secret)
```

### Strategy 2: Keep One SMS-Capable Number

Maintain at least one service (VoIP.ms or Twilio) that can receive SMS for critical accounts where authenticator apps aren't available.

### Strategy 3: Virtual SIM Solutions

Services like Airalo provide eSIM data plans. Some users maintain a separate device or eSIM specifically for SMS verification while using local numbers for daily communication.

## Practical Setup Recommendations

For developers and power users, here's a recommended stack:

1. **Primary number**: Use a local Portuguese or Spanish SIM for daily communication and data
2. **US number for verification**: VoIP.ms ($3/month) or Twilio ($1/month)
3. **Forwarding**: Configure your US number to forward to your local SIM for calls
4. **SMS handling**: Use Twilio webhooks to forward important messages to email or Slack

### Sample Call Forwarding Configuration (VoIP.ms)

Log into your VoIP.ms portal:
- Go to "Call Routing" → "Simple Forwarding"
- Set "Forward To" to your local number with country code
- Enable "Ring Simultaneous" if you want both phones to ring

## Common Pitfalls to Avoid

- **Relying solely on Google Voice** — You'll hit limitations with banks and services
- **Forgetting to pay bills** — VoIP numbers get disconnected quickly for non-payment
- **Not testing SMS** — Always verify your setup can receive messages before you need it
- **Ignoring timezone differences** — Set appropriate voicemail greetings

## Cost Comparison

| Service | Monthly Cost | SMS | Calls | Best For |
|---------|--------------|-----|-------|----------|
| Google Voice | Free (limited) | Partial | US only | Basic needs |
| VoIP.ms | $3-5 | Yes | Yes | Reliability |
| Twilio | $1-10 | Yes | Pay-per-use | Developers |
| Google Fi | $20+ | Yes | Yes | Simplicity |

## Conclusion

Maintaining a US phone number while working remotely from Portugal or Spain is entirely feasible with the right tools. For developers and technical users, Twilio or VoIP.ms provide the best balance of cost, reliability, and flexibility. The key is setting up your solution before you need it—configure everything while you still have easy access to US-based verification.

Once you have a reliable US number forwarding to your local device, you can work remotely with the confidence that your critical accounts remain accessible.

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
