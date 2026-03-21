---
layout: default
title: "Using Microsoft Graph API to create named locations"
description: "A practical guide for developers and IT professionals on implementing Azure Conditional Access policies to secure remote work environments. Includes"
date: 2026-03-16
last_modified_at: 2026-03-16
author: "Remote Work Tools"
permalink: /how-to-implement-conditional-access-policies-for-remote-work/
reviewed: true
score: 8
categories: [guides]
intent-checked: true
voice-checked: true
tags: [remote-work-tools, remote-work]
---

{% raw %}
Implement Conditional Access policies in Azure Entra ID to require multi-factor authentication for remote users and block access from non-compliant devices. For remote teams, Conditional Access policies balance security with usability—ensuring developers can work productively while protecting sensitive company data. Azure Entra ID evaluates signals about user identity and environment to grant, block, or require additional verification for access. This guide walks through practical implementation for remote work scenarios with real configurations you can deploy today.

## Understanding Conditional Access Fundamentals

Conditional Access works on a simple principle: evaluate signals about a user's identity and environment, then decide whether to grant access, block access, or require additional verification. The core components include:

- **Assignments** define who the policy applies to and what conditions must be met
- **Access controls** specify what happens when conditions are satisfied
- **Grant or Block** decisions enforce the final outcome

For remote workers, the most relevant signals include device compliance, location, sign-in risk, and user risk levels. You combine these signals to create policies that protect your organization without creating friction for legitimate users.

## Building Your First Remote Worker Policy

The most common starting point is requiring multi-factor authentication (MFA) for remote access. This ensures that even if credentials are compromised, attackers cannot access your resources without a second factor.

### Policy: Require MFA for Remote Users

Navigate to Microsoft Entra ID > Protection > Conditional Access > Policies and create a new policy:

```json
{
  "name": "Require MFA for Remote Users",
  "state": "enabled",
  "conditions": {
    "signInRisk": {
      "operator": "notEquals",
      "riskLevel": "low"
    },
    "ipRanges": [
      {
        "cidrAddress": "0.0.0.0/0",
        "description": "All locations - catch all"
      }
    ]
  },
  "grantControls": {
    "operator": "AND",
    "builtInControls": ["mfa"]
  }
}
```

In practice, you would exclude your corporate VPN IP ranges from this condition so that users on the company network don't need MFA, while all other locations require it.

## Controlling Access by Device Compliance

Remote workers often use personal devices or laptops that may not meet your organization's security standards. Device-based Conditional Access ensures only compliant devices can access sensitive resources.

### Policy: Block Access from Non-Compliant Devices

```json
{
  "name": "Block Non-Compliant Devices",
  "state": "enabled",
  "conditions": {
    "deviceFilters": {
      "includeDevices": ["All devices"],
      "excludeDevices": ["Compliant devices"]
    },
    "applications": {
      "includeApplications": [
        "Office 365",
        "Azure Portal",
        "Custom SaaS Apps"
      ]
    }
  },
  "accessControls": [
    {
      "blockAccess": true,
      "customAuthenticationFactors": []
    }
  ]
}
```

This policy prevents access to Microsoft 365, the Azure portal, and custom applications from devices that aren't marked as compliant in Intune. For developers accessing code repositories or CI/CD pipelines, you can target specific applications with this same approach.

## Location-Based Access Control

Geographic restrictions add another security layer. You can create named locations in Microsoft Entra ID and use them in Conditional Access policies to allow or block access from specific countries.

### Policy: Block Sign-Ins from High-Risk Locations

First, define your trusted locations:

```powershell
# Using Microsoft Graph API to create named locations
New-MgNamedLocation -CountryNamedLocation `
  -DisplayName "Trusted Countries" `
  -CountriesAndRegions @("US", "CA", "GB", "DE") `
  -IncludeUnknownCountriesAndRegions $false
```

Then create the policy:

```json
{
  "name": "Block High-Risk Countries",
  "conditions": {
    "locations": {
      "includeLocations": ["All"],
      "excludeLocations": ["Trusted Countries"]
    },
    "clientApps": ["Browser", "MobileApps", "ExchangeActiveSync"]
  },
  "grantControls": {
    "blockAccess": true
  }
}
```

For remote teams with global distribution, you might instead choose to require MFA for any sign-in outside your expected regions, rather than blocking entirely.

## Implementing Risk-Based Policies

Azure Identity Protection provides risk detection that feeds directly into Conditional Access. You can create policies that respond to risky sign-ins automatically.

### Policy: Require Password Change for High-Risk Users

```json
{
  "name": "High Risk User - Password Reset",
  "state": "enabled",
  "conditions": {
    "userRiskLevels": ["high"],
    "signInRiskLevels": ["high"]
  },
  "accessControls": [
    {
      "grantControl": {
        "operator": "AND",
        "builtInControls": ["passwordChange"]
      }
    }
  ]
}
```

This policy forces users flagged as high risk to change their password before gaining access. Combined with MFA requirements, this creates a defense-in-depth approach.

## Session Policies for Data Protection

Beyond blocking or granting access, Conditional Access supports session policies that control what users can do after authenticating. These are particularly useful for protecting sensitive data in cloud applications.

### Policy: Require Session Re-authentication for Sensitive Actions

```json
{
  "name": "Require Fresh Authentication for Data Export",
  "conditions": {
    "applicationConditions": {
      "includeApplications": ["Office 365 SharePoint Online"]
    },
    "userActions": {
      "protectBrowserSession": false,
      "registerDevice": false
    }
  },
  "sessionControls": {
    "applicationEnforcedRestrictions": {
      "enabled": true
    },
    "continuousAccessEvaluation": {
      "enabled": true
    }
  }
}
```

The continuous access evaluation feature provides real-time token revocation—when a user's account is disabled or their risk level changes, active sessions are terminated immediately rather than waiting for token expiration.

## Combining Policies for Layered Security

The most effective Conditional Access implementations use multiple policies together. A typical remote work scenario might include:

1. Baseline policy: Require MFA for all cloud apps
2. Device policy: Block non-compliant devices from production environments
3. Location policy: Require MFA for sign-ins outside trusted regions
4. Risk policy: Block high-risk sign-ins entirely

Test each policy in report-only mode before enabling enforcement. Microsoft's Conditional Access insights workbook helps you understand the impact before deployment.

## Troubleshooting Remote Access Issues

When implementing Conditional Access for remote teams, you'll inevitably encounter access issues. Common problems include:

Users blocked unexpectedly: Check the sign-in logs in Microsoft Entra ID. Filter by the user and examine the detailed error. The "Why blocked" column often provides specific guidance.

MFA prompts every sign-in: Ensure trusted locations are configured correctly, or adjust session duration settings. You can also exclude browser sign-ins from MFA requirements if appropriate for your risk tolerance.

Device compliance issues: Verify Intune enrollment status and compliance policies. Users need to enroll their devices and receive compliant status before device-based policies will work.

## Deployment Best Practices

Follow these principles when rolling out Conditional Access:

- **Start with report-only mode** to understand policy impact before enforcement
- **Create exclusion groups** for break-glass accounts that must always have access
- **Use named locations** to define your corporate networks and trusted regions
- **Implement gradually**—enable policies for pilot groups before organization-wide deployment
- **Document your policies** so other administrators understand the security rationale

For remote teams specifically, ensure your policies account for legitimate use cases: developers traveling to conferences, employees working from coffee shops, and contractors accessing resources from various locations.

---


## Related Reading

- [How to Implement Geo-Fencing Access Controls for Remote](/remote-work-tools/how-to-implement-geo-fencing-access-controls-for-remote-team/)
- [How to Implement Just-in-Time Access for Remote Team.](/remote-work-tools/how-to-implement-just-in-time-access-for-remote-team-cloud-r/)
- [How to Implement Least Privilege Access for Remote Team](/remote-work-tools/how-to-implement-least-privilege-access-for-remote-team-clou/)
- [Example: Create a booking via API](/remote-work-tools/best-client-scheduling-tool-for-remote-agency-multiple-time-/)
- [Example: Verify MFA is enabled via API (GitHub Enterprise)](/remote-work-tools/how-to-create-security-onboarding-checklist-for-new-remote-t/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
