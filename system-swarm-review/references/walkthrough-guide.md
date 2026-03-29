# Walkthrough Guide

Reference for generating walkthrough steps that are grounded in the actual product
and codebase -- not generic UX theory. Read during Phase 2 before writing steps.

---

## Core Principle

Every walkthrough step must correspond to a real screen or interaction in the
codebase. If you can't point to a component, route, or file that represents a step,
it shouldn't be a step.

The goal is to walk through the product the way a real first-time user would --
not a curated demo path, and not an exhaustive feature inventory.

---

## Step Generation Process

1. From the codebase scan, identify all user-facing routes or screens
2. Map the natural first-session journey: what does a new user encounter in order?
3. Select 5-7 steps from that journey, always including:
   - First impression (onboarding / landing / auth screen)
   - Primary value-delivery moment (where the product delivers its core promise)
   - Completion / commitment flow (submit, publish, buy, save)
4. Add middle steps that represent meaningful decision points or friction moments

---

## Step Templates by Product Type

Academic / EdTech Platform:
1. Landing on the onboarding screen for the first time
2. The required setup step (uploading a transcript, connecting an account)
3. First look at the dashboard after setup
4. Using the primary planning or tracking feature
5. Interacting with the AI or advisory feature (if present)
6. The submission or sharing flow (advisor review, export, etc.)

SaaS Dashboard / B2B Tool:
1. First login or trial signup
2. Initial setup or onboarding wizard
3. Creating the first artifact (project, report, record)
4. The core workflow loop (the action the user will do daily)
5. Collaboration or sharing feature
6. Settings or configuration (especially billing/plan limits)

Consumer App:
1. App store listing / first impression (if reviewable)
2. Sign up or guest flow
3. The moment they first get it -- the aha moment
4. Daily use interaction (the thing they'd do every day)
5. Account management or personalization
6. The friction point (upgrade prompt, paywall, permission request)

Developer Tool / API:
1. Documentation landing page / quickstart
2. First installation or setup
3. First successful API call or command
4. The common use case they came for
5. Error handling (what happens when something goes wrong)
6. Advanced features or configuration

Marketplace / E-commerce:
1. Landing on a product or category page
2. Product detail / evaluation step
3. Adding to cart or saving
4. Checkout or conversion flow
5. Post-purchase or confirmation state
6. Account or order management

Internal Tool / Admin Dashboard:
1. First login (often provisioned by someone else)
2. Finding the data or record they need
3. Performing the core action (edit, approve, escalate, export)
4. Generating or viewing a report
5. Admin or settings area

---

## Describing Steps

Each step title should be action-oriented and specific:
- GOOD: "Landing on the DARS upload screen for the first time"
- GOOD: "Opening the AI Advisor and sending the first message"
- BAD: "Using the main feature"
- BAD: "Navigation"

---

## What to Look For at Each Step

Cognitive load: Is there too much on screen? What's the first thing they look at?
Trust signals: What makes them feel safe (or unsafe) giving data or clicking?
Jargon: Any terms that would confuse a newcomer?
Missing context: What question does this screen need to answer that it doesn't?
Dead ends: What happens if something goes wrong? Is there a path forward?
The gap: What did the persona come hoping to find? Did they find it?
