---
prompts:
  - prompt: "Log who opens our shared demo links: time, city, country, device and browser, stored in Supabase, with a Slack message the first time a customer opens one."
    stack: Supabase
  - prompt: Record where and on what device each user signed up, and show it on their profile in the admin panel.
    stack: Firestore
  - prompt: Our visit log says the customer opened the demo when nobody did. Filter out prefetches, bots and staff previews.
---

# Prompts

What an operator types after installing this skill, in their own words. An agent eval installs the skill
into an empty Next.js app, gives the agent one of these prompts and no further help, then type-checks, builds
and tests the result; the first prompt runs before every release. The results are the other files in this
folder. Section 10 of [STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) says how a
run is made. The prompts and the newest runs are on
[the skill's page](https://timerise.ai/skills/visit-logger) on timerise.ai.
