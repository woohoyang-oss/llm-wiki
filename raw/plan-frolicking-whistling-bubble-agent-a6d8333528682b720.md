# Open-Source Projects Similar to Auto-DM: Research Report

## Research Summary

Searched GitHub and the web across 6+ query sets, examined 15+ repositories. Projects are categorized by type and relevance to the Auto-DM use case (DM automation, event forms, multi-channel messaging via official Instagram Graph API).

---

## TIER 1: Most Relevant -- Instagram DM Automation via Official Graph API

### 1. FUMA (aadii-rawt/FUMA)
- **URL:** https://github.com/aadii-rawt/FUMA
- **Stars:** ~2 (very new/small project)
- **What it does:** Modern automation platform enabling Instagram Business accounts to automate DMs, comment replies, and engagement workflows via the Meta Graph API. Closest direct competitor to what Auto-DM is building.
- **Tech stack:** React + TypeScript + Vite (frontend), Node.js + Express + TypeScript + Prisma + PostgreSQL (backend)
- **Last activity:** Active (133 commits on main)
- **Uses official Graph API:** YES -- webhook-based, uses Meta Graph API with proper OAuth
- **Key features for our use case:**
  - Automated Instagram DM responses
  - Comment reply automation
  - Webhook integration for real-time events (messages, comments, mentions)
  - OAuth-based Meta/Facebook authentication
  - Instagram Business/Professional account support
- **Assessment:** Very similar architecture to Auto-DM. Small community but correct technical approach. Good reference implementation.

### 2. SashenJayathilaka/SAAS-Instagram-DM-Automations
- **URL:** https://github.com/SashenJayathilaka/SAAS-Instagram-DM-Automations
- **Stars:** ~44
- **What it does:** Full SaaS platform for Instagram DM automation -- keyword-based auto-replies, story reply automation, comment-to-DM workflows, analytics dashboard.
- **Tech stack:** Next.js + TypeScript + TailwindCSS + Prisma + Neon (PostgreSQL) + Clerk (auth) + Stripe (payments) + OpenAI API
- **Last activity:** Active as of April 2026
- **Uses official Graph API:** YES -- Instagram OAuth + webhooks
- **Key features for our use case:**
  - Instagram OAuth integration
  - Visual automation builder to trigger automations
  - DM automations triggered by keywords
  - Story reply automations
  - Comment-based automation triggers for DM outreach
  - Metrics dashboard
  - SaaS billing (Stripe)
- **Assessment:** Most architecturally similar to Auto-DM. Modern Next.js stack, uses official API, has SaaS infrastructure. Strong reference for product design.

### 3. Sudershhh/saas-dm-automations
- **URL:** https://github.com/Sudershhh/saas-dm-automations
- **Stars:** ~10
- **What it does:** Similar to SashenJayathilaka's project -- SaaS Instagram DM automation with webhook handlers.
- **Tech stack:** Next.js 14 (App Router) + TypeScript + TailwindCSS + shadcn/ui + Prisma + Redux Toolkit + React Query
- **Last activity:** Active
- **Uses official Graph API:** YES
- **Key features:** Webhook handlers for Instagram, API routes, modern React component architecture
- **Assessment:** Another good reference for the webhook + Next.js approach.

---

## TIER 2: Large Multi-Channel Platforms (with Instagram support)

### 4. Chatwoot (chatwoot/chatwoot)
- **URL:** https://github.com/chatwoot/chatwoot
- **Stars:** ~27,400
- **What it does:** Open-source omni-channel customer support platform. Live chat, email, Facebook, Instagram, WhatsApp, Telegram, Line, SMS. Alternative to Intercom/Zendesk.
- **Tech stack:** Ruby on Rails (backend), Vue.js (frontend), PostgreSQL, Redis, Sidekiq
- **Last activity:** Very active, regular releases (v4.x in 2025-2026)
- **Uses official Graph API:** YES for Instagram DM integration
- **Key features for our use case:**
  - Instagram DM integration (receive/reply to DMs, story replies, mentions)
  - Telegram integration
  - Webhook-based architecture
  - Multi-channel unified inbox
  - Automation rules engine
  - Self-hosted or cloud
  - Agent assignment, canned responses
- **Assessment:** Heavyweight platform. Not DM-automation-focused but has Instagram channel support. Overkill for auto-reply but excellent reference for multi-channel architecture. Note: some recent issues with Instagram integration (July 2025 bug reports).

### 5. Botpress (botpress/botpress)
- **URL:** https://github.com/botpress/botpress
- **Stars:** ~14,600
- **What it does:** Open-source hub to build and deploy GPT/LLM agents. Visual flow builder with multi-channel deployment including Instagram.
- **Tech stack:** TypeScript, Node.js
- **Last activity:** Very active (2026 updates)
- **Uses official Graph API:** YES -- official Instagram integration
- **Key features for our use case:**
  - Instagram DM send/receive
  - Visual conversation flow builder
  - LLM/GPT integration built-in
  - Multi-channel (Instagram, Telegram, WhatsApp, Messenger, etc.)
  - Webhook-based
  - Self-hosted option (v12 is OSS; newer versions are cloud-focused)
- **Assessment:** Very mature platform. Note: v12 (botpress/v12) is the fully open-source self-hosted version. Current main repo is more cloud/SaaS oriented. Good for understanding conversational flow design.

### 6. Hexabot (Hexastack/Hexabot)
- **URL:** https://github.com/Hexastack/Hexabot
- **Stars:** ~929
- **What it does:** Open-source AI chatbot/agent builder. Multi-channel, multilingual, plugin-based architecture.
- **Tech stack:** TypeScript, NestJS (backend), React/Next.js (frontend), MongoDB, Docker Compose
- **Last activity:** Very active (2,834 commits)
- **Uses official Graph API:** Has Messenger channel plugin; Instagram support via Meta platform (channel extensions)
- **Key features for our use case:**
  - Drag-and-drop visual flow editor
  - Plugin/extension system for channels (WhatsApp, Messenger extensions exist)
  - LLM integration (Ollama, ChatGPT, Mistral, Gemini)
  - Multi-language support
  - Knowledge base integration
  - Human agent handover
  - Role-based access control
- **Assessment:** Good growing project with solid architecture. NestJS backend is interesting. Instagram support is via Messenger channel plugin (Meta platform), not a dedicated Instagram channel yet. Plugin architecture is worth studying.

### 7. Typebot (baptisteArno/typebot.io)
- **URL:** https://github.com/baptisteArno/typebot.io
- **Stars:** ~9,800+
- **What it does:** Powerful self-hostable chatbot builder with drag-and-drop interface. Focused on conversational forms and lead capture.
- **Tech stack:** TypeScript, Next.js, Prisma, PostgreSQL
- **Last activity:** Active
- **Uses official Graph API:** NO native Instagram integration. Facebook/Instagram messenger request was closed as "not planned" (issue #1547).
- **Key features for our use case:**
  - Drag-and-drop conversation flow builder
  - Webhook integrations
  - OpenAI integration
  - Google Sheets, Zapier, Make.com integration
  - Self-hostable
  - Embeddable widget
- **Assessment:** Great for conversational forms but lacks Instagram integration. Could be useful for the "event form" part of Auto-DM if embedded, but not for DM automation itself.

### 8. Rasa (RasaHQ/rasa)
- **URL:** https://github.com/RasaHQ/rasa
- **Stars:** ~19,000-20,000+
- **What it does:** ML framework for text/voice-based conversational AI. NLU, dialogue management, channel connectors.
- **Tech stack:** Python, ML/NLP libraries
- **Last activity:** Active (commercial Rasa Pro + open-source Rasa)
- **Uses official Graph API:** Has Facebook Messenger channel; no dedicated Instagram channel out of the box
- **Key features for our use case:**
  - Advanced NLU/intent classification
  - Dialogue management with ML
  - Custom channel connectors (could build Instagram)
  - Slot filling / form handling
  - Self-hosted
- **Assessment:** Heavyweight NLP framework. Overkill for keyword-based auto-reply but interesting if Auto-DM needs sophisticated intent understanding. No built-in Instagram support.

### 9. Tiledesk (Tiledesk/tiledesk-chatbot)
- **URL:** https://github.com/Tiledesk/tiledesk-chatbot
- **Stars:** ~70 (chatbot engine); parent org has multiple repos
- **What it does:** Open-source AI chatbot engine with multichannel support. Visual no-code designer. Alternative to Voiceflow.
- **Tech stack:** Node.js/JavaScript, MongoDB, RabbitMQ/MQTT, Redis, Docker
- **Last activity:** Active (1,940 commits)
- **Uses official Graph API:** Supports WhatsApp, Facebook Messenger, Telegram. Instagram support unclear.
- **Key features for our use case:**
  - Visual chatbot designer (Design Studio)
  - Write once, run on any channel (auto-adapts responses)
  - LLM/GPT integration
  - Human handover
  - Scalable architecture
- **Assessment:** Interesting "write once, deploy everywhere" approach to multi-channel. Architecture worth studying but Instagram DM support not confirmed.

---

## TIER 3: Workflow Automation (not chatbot-specific but supports Instagram)

### 10. n8n (n8n-io/n8n)
- **URL:** https://github.com/n8n-io/n8n
- **Stars:** ~182,000
- **What it does:** Fair-code workflow automation platform with 400+ integrations and native AI capabilities. Visual workflow builder.
- **Tech stack:** TypeScript, Node.js, Vue.js
- **Last activity:** Extremely active
- **Uses official Graph API:** YES -- has Instagram nodes via Meta Graph API (media publishing, comment moderation, private replies, messaging, hashtag search)
- **Key features for our use case:**
  - Instagram node with Graph API support
  - Webhook triggers
  - AI agent capabilities
  - Can build Instagram DM auto-reply workflows
  - 400+ other integrations (Telegram, Slack, Google Sheets, etc.)
  - Self-hostable
  - Community templates for Instagram DM management with ManyChat
- **Assessment:** Not a chatbot platform per se, but could be used to BUILD Instagram DM automation workflows. Extremely popular and well-maintained. Could serve as an orchestration layer for Auto-DM.

---

## TIER 4: Instagram Bots (Browser Automation / Private API -- NOT Recommended)

These use browser automation or Instagram's private API, violating ToS. Listed for awareness only.

### 11. InstaPy (InstaPy/InstaPy)
- **URL:** https://github.com/InstaPy/InstaPy
- **Stars:** ~17,000+ (declining, many unfollowed)
- **What it does:** Instagram bot for automated interactions (likes, follows, comments). Uses Selenium.
- **Status:** Effectively dead. Facebook sent cease-and-desist to creator. Not maintained.
- **DO NOT USE:** Browser automation, violates ToS.

### 12. GramAddict (GramAddict/bot)
- **URL:** https://github.com/GramAddict/bot
- **Stars:** ~3,000+
- **What it does:** Human-like Instagram bot using UIAutomator2 on Android devices.
- **Status:** Active but risky.
- **DO NOT USE:** Android device automation, violates ToS.

### 13. MR.DM (Oxlac/MR.DM)
- **URL:** https://github.com/Oxlac/MR.DM
- **Stars:** ~60
- **What it does:** Desktop tool for mass-sending Instagram DMs using Selenium.
- **Tech stack:** Python + Kivy (desktop GUI)
- **DO NOT USE:** Selenium-based, violates ToS.

### 14. instagrapi (subzeroid/instagrapi)
- **URL:** https://github.com/subzeroid/instagrapi
- **Stars:** ~5,000+
- **What it does:** Python library for Instagram's private/internal API. Very comprehensive.
- **Tech stack:** Python
- **DO NOT USE for production:** Uses private API, risk of account bans.

### 15. IGopher (hbollon/IGopher)
- **URL:** https://github.com/hbollon/IGopher
- **Stars:** ~200+
- **What it does:** Customizable Instagram DM bot with TUI and Electron GUI. Uses Selenium.
- **Tech stack:** Go + Electron.js
- **DO NOT USE:** Browser automation approach.

---

## Key Takeaways for Auto-DM

1. **Closest competitors using official Graph API:** FUMA and SAAS-Instagram-DM-Automations are the most directly comparable. Both are small projects (<50 stars) meaning the space is still wide open.

2. **Dominant tech stack for this space:** Next.js + TypeScript + Prisma + PostgreSQL is the clear winner for new Instagram DM SaaS projects.

3. **Multi-channel gap:** No open-source project does Instagram DM automation + Telegram + event forms well. Chatwoot is the closest but it's a customer support tool, not an automation/marketing tool.

4. **Official API adoption is low:** Most Instagram bot repos still use browser automation or private APIs. Projects using the official Graph API are rare and small, which means Auto-DM has an opportunity to become the leading open-source solution in this space.

5. **Architecture references to study:**
   - **FUMA** -- for Graph API webhook architecture
   - **SashenJayathilaka's project** -- for SaaS product design (OAuth, automation builder, dashboard)
   - **Hexabot** -- for plugin-based multi-channel architecture (NestJS)
   - **Chatwoot** -- for mature multi-channel inbox design
   - **n8n** -- for workflow automation patterns

6. **The ManyChat alternative space is unserved in open source.** All the "ManyChat alternatives" are either closed-source SaaS products or general-purpose chatbot platforms. None are purpose-built open-source Instagram DM marketing automation tools.
