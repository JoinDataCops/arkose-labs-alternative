# DataCops vs Arkose Labs: signup fraud + CAPI feedback layer

A reference implementation for tying signup-fraud verdicts to your Meta and Google CAPI event payloads. The layer most signup-fraud vendors don't cover.

## Why this exists

When a bot signs up, the pixel fires before the form submits. Meta and Google receive the Lead or CompleteRegistration event. Their optimization models train on the bot. Lookalike audiences get poisoned. CAC creeps up silently.

Most signup-fraud vendors block the signup at the form layer. The pixel still fires. The CAPI event still goes through. The optimization layer still trains on bots.

This README is the working pattern for closing that loop.

## The architecture

```
[user form submit]
       |
       v
[network-layer verdict at the CNAME endpoint]
       |
       +-- (verdict: human) --> [Lead event with verdict=human in CAPI]
       |
       +-- (verdict: bot)   --> [Lead event blocked, no pixel/CAPI fire]
       |
       +-- (verdict: risky) --> [Lead event with verdict=risky in CAPI, optimize_for=manual_review]
```

## The Meta CAPI event payload

```json
{
  "event_name": "Lead",
  "event_time": 1715299200,
  "event_id": "evt_abc123",
  "action_source": "website",
  "user_data": {
    "em": ["hashed_email"],
    "client_ip_address": "203.0.113.42",
    "client_user_agent": "Mozilla/5.0..."
  },
  "custom_data": {
    "fraud_verdict": "human",
    "fraud_score": 0.04,
    "fraud_reasons": []
  }
}
```

When `fraud_verdict` is `bot` or `risky`, set `data_processing_options: ["LDU"]` and tag the event so Meta excludes it from optimization.

## The Google Ads Enhanced Conversions payload

```json
{
  "conversion_action": "customers/123/conversionActions/abc",
  "conversion_date_time": "2026-05-10 12:00:00 UTC",
  "user_identifiers": [{"hashed_email": "..."}],
  "custom_variables": {
    "fraud_verdict": "human",
    "fraud_score": "0.04"
  }
}
```

## When Arkose Labs is the right call

- SMS toll fraud is a top 3 risk. Arkose's $1M warranty is real.
- Adversarial-scale gaming or social platform (Roblox-tier).
- Fortune 500 procurement budget and a security team that wants a single vendor for bot defense + device fingerprinting + behavioral + CAPTCHA.

## When DataCops is the right call

- Paid-acquisition SaaS or ecommerce. CAC creep is the silent killer.
- You want published pricing and a 5-minute setup.
- You want the same IP reputation pipeline filtering ad fraud and validating signups.
- You want the CAPI event tagged with the fraud verdict so Meta and Google stop optimizing toward bots.

## Setup

```bash
# 1. CNAME your tracking endpoint
# CNAME datacops -> cdn.datacops.com

# 2. Drop the script
```

```html
<script async src="https://datacops.yourdomain.com/dc.js" data-site="YOUR_SITE_ID"></script>
```

```bash
# 3. Wire the signup form
```

```html
<form id="signup">
  <input name="email" type="email" required>
  <button type="submit">Sign up</button>
</form>

<script>
document.getElementById('signup').addEventListener('submit', async (e) => {
  e.preventDefault();
  const verdict = await window.dc.verifySignup({
    email: e.target.email.value
  });
  if (verdict.status === 'block') {
    // Optionally show generic friendly error
    return;
  }
  // Submit to your backend, include verdict.event_id
  await fetch('/api/signup', {
    method: 'POST',
    body: JSON.stringify({
      email: e.target.email.value,
      event_id: verdict.event_id,
      fraud_verdict: verdict.status
    })
  });
});
</script>
```

```bash
# 4. Server-side: pass the verdict into the CAPI event
```

```javascript
// Node example
async function sendMetaCAPI(eventData, verdict) {
  return fetch(`https://graph.facebook.com/v18.0/${PIXEL_ID}/events`, {
    method: 'POST',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify({
      data: [{
        event_name: 'Lead',
        event_time: Math.floor(Date.now() / 1000),
        event_id: eventData.event_id,
        action_source: 'website',
        user_data: { em: [hashEmail(eventData.email)] },
        custom_data: {
          fraud_verdict: verdict,
          fraud_score: eventData.score
        },
        ...(verdict !== 'human' && { data_processing_options: ['LDU'] })
      }],
      access_token: META_CAPI_TOKEN
    })
  });
}
```

## Pricing comparison

| Vendor | Entry tier | Mid-market | Enterprise | Published? |
|---|---|---|---|---|
| Arkose Labs | n/a | Custom | $50K to $500K+ /yr | No |
| DataCops | Free (500 verifications/mo) | $49/mo Business | $299/mo Organization | Yes |

## Disclaimer

DataCops is in the SOC 2 Type II in-progress phase, not certified. The Enterprise tier offers a single-tenant runtime, dedicated IP DB, custom DPA, and EU/US data residency today.

## Links

- SignUp Cops: joindatacops.com/signup-cops
- Conversion API: joindatacops.com/conversion-api
- Pricing: joindatacops.com/pricing

---

Research by [DataCops](https://www.joindatacops.com) · First-party tracking, consent infrastructure & fraud prevention.
