# Lab: Registering Your Cribl.Cloud Organization

**Based on:** [Register Cribl.Cloud Organization — Cribl Docs](https://docs.cribl.io/stream/cloud-initial-setup/)

---

## Overview

Cribl.Cloud is Cribl's fully-managed hosting option for Cribl Stream (and other Cribl products). Before you can create Worker Groups, add Sources/Destinations, or build Pipelines, you need a **Cribl.Cloud Organization** — your dedicated, isolated portal with its own network and access boundaries.

In this lab, you'll sign up for a free Cribl.Cloud account, register an Organization, choose a region, and confirm your setup is ready to use.

---

## Learning Objectives

By the end of this lab, you will be able to:

- Sign up for a Cribl.Cloud account using a work email address
- Register a new Cribl.Cloud Organization
- Explain what an Organization Region controls, and why the choice is permanent
- Identify the limits of the free Cribl.Cloud plan
- Locate the path to upgrade to a paid or Enterprise plan
- Know where to go next (Getting Started Guide) and where to get help if you get stuck

---

## Prerequisites

- A work email address (personal/free-mail addresses are typically not accepted)
- Access to a modern web browser
- No existing active free-tier Cribl.Cloud Organization tied to your email (**each user may only register one active Organization on the free tier**)

---

## Estimated Time

15–20 minutes

---

## Lab Environment

No local software installation is required — Cribl.Cloud is entirely browser-based for this lab.

---

## Part 1: Sign Up for Cribl.Cloud

1. Open your browser and go to the Cribl.Cloud signup page:
   👉 **https://cribl.cloud/signup/**

2. Create an account using your **work email address**.

3. Check your inbox for a verification email from Cribl, and complete the verification step.

> **Checkpoint:** You should now be logged into the Cribl.Cloud signup flow and prompted to set up your Organization.

---

## Part 2: Create Your Organization

1. On the **Create Organization** page, optionally enter an **Organization Name** — a friendly alias for the randomly generated ID Cribl will assign to your Organization.

2. Select a **Region** for your Organization (see [Part 3](#part-3-understand-organization-regions) before you confirm — this choice is permanent).

3. Provide the requested information about your goals for using Cribl. Cribl.Cloud uses this to tailor the suggested next actions on your home page.

4. Submit the form to finish registration.

> **Checkpoint:** You should land on your new Cribl.Cloud deployment home page, with suggested actions based on the goals you entered.

**Important:** Bookmark your Cribl.Cloud portal page now — you'll need it for everything that follows in future labs.

---

## Part 3: Understand Organization Regions

Before (or right after) you pick a region, understand what it controls:

| Concept | What the Region Determines |
|---|---|
| **Cribl Stream** | Where the **Leader Node** resides |
| **Cribl Search** | The physical location where searches are executed |
| **Cribl Lake** | Where your data is stored |

**Key facts:**

- Once you select a region during Organization registration, **you cannot change it**.
- You *can* still create **Cribl-managed Stream Worker Groups** in different AWS and Azure regions later, even though your Organization's home region is fixed.

**Reflection question:** If your organization has data-residency requirements (e.g., EU-only storage), why does it matter that the region choice is permanent? What would you need to do if you registered in the wrong region?

<details>
<summary>Answer</summary>

Because the region can't be changed after registration, choosing the wrong one for data-residency needs would require registering an entirely new Organization in the correct region and migrating configuration and data — there's no in-place region migration. This is a good example of why sizing/planning steps matter *before* you commit to Organization setup, not after.
</details>

---

## Part 4: Explore the Free Tier

Your new Organization starts on the **free Cribl.Cloud plan** by default.

1. In your Cribl.Cloud portal, locate the option to **Go Enterprise**.
2. Note (without necessarily clicking through) that the free plan includes specific **throughput and administration limits**.
3. Note that upgrade paths include:
   - A **paid plan**, or
   - An **Enterprise** plan
4. For organizations already using AWS, note that the **Cribl.Cloud Suite** is available on the **AWS Marketplace**, and Enterprise Discount Program (EDP) credits can be applied directly — no separate procurement process needed.

**Reflection question:** Why do you think Cribl only allows one active free-tier Organization per user?

<details>
<summary>Answer</summary>

It prevents a single user from spinning up unlimited free-tier environments to bypass throughput/admin limits — the constraint keeps the free tier sustainable while still letting anyone evaluate the product with a real, functioning Organization.
</details>

---

## Part 5: Confirm Your Setup

Verify you've completed registration successfully by checking that you can answer **yes** to each item:

- [ ] I can log into my Cribl.Cloud portal at its bookmarked URL
- [ ] My Organization has a name or ID visible in the portal
- [ ] I can see my selected region under Organization details
- [ ] I can see suggested next actions on my home page

---

## Troubleshooting

If you get stuck during registration:

- Cribl University has a walkthrough course: **How to Log Into Cribl.Cloud** (requires a free Cribl University account — sign up at university.cribl.io if needed).
- Related short courses worth bookmarking for later: *Troubleshooting Criblets*, *Advanced Troubleshooting*.

---

## Knowledge Check

1. What does the Organization Region actually determine for Cribl Stream specifically?
2. True or False: You can change your Organization's region after registration.
3. Can a single free-tier user register more than one active Organization?
4. Where would you go to increase your throughput/administration limits beyond the free tier?

<details>
<summary>Answers</summary>

1. It determines where the **Leader Node** resides.
2. **False** — the region is permanent once selected at registration.
3. **No** — only one active Organization per user on the free tier.
4. Select **Go Enterprise** in the Cribl.Cloud portal to move to a paid or Enterprise plan.
</details>

---

## Next Steps

Now that your Organization is registered:

- Continue to the **[Getting Started Guide](https://docs.cribl.io/stream/getting-started-guide)** for a walkthrough of core Cribl Stream features and use cases.
- In a follow-up lab, you'll create your first **Worker Group** and start routing data through **Sources → Pipelines → Destinations**.

---

## Additional Resources

- [Register Cribl.Cloud Organization (source doc)](https://docs.cribl.io/stream/cloud-initial-setup/)
- [Manage Cribl.Cloud Organization](https://docs.cribl.io/stream/cloud-portal/)
- [Cribl.Cloud Enterprise](https://docs.cribl.io/stream/cloud-enterprise/)
- [Cribl.Cloud Data Isolation](https://docs.cribl.io/reference-architectures/reference-arch-full-suite#isolation)
