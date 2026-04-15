---
id: credential-setup
title: 8. Credential & Account Setup
---

# 8. Credential & Account Setup

To run the *Capital AE Automation* effectively, **n8n** requires authentication credentials to interact with external services: Google Workspace, Slack, and Telegram.

This section provides a streamlined guide detailing the required connection types, necessary permissions, and directs you to the official n8n setup documents for creating these connections.

## 8.1 Required Integration List

There are five core nodes across the automation pipeline that need configuring:
- `Google Sheets` & `Google Sheets Trigger`
- `Google Drive`
- `Gmail`
- `Slack`
- `Telegram`

## 8.2 Google Services (Sheets, Drive, Gmail)

n8n can authenticate with Google using either **OAuth2** (better for personal access) or a **Service Account** (recommended for autonomous headless servers without a user screen).

### Required API Scopes
Ensure the following APIs are strictly enabled from your Google Cloud Console (`console.cloud.google.com`):
1. **Google Drive API**
2. **Google Sheets API**
3. **Gmail API**

:::info
Once configured, you only need to create *one* Google credential account in n8n, which can be reused across all Google Sheets, Google Drive, and Gmail nodes!
:::

### Full Official Guide
> 👉 **[n8n Docs: Setting up Google Credentials](https://docs.n8n.io/integrations/builtin/credentials/google/)**

## 8.3 Slack App / Webhook

To dispatch rendering logs, success warnings, and FATAL execution errors to a Slack channel, you will need a Slack connection.

You can configure this in two ways:
1. **Incoming Webhook (Easy)**: A static URL generated from your Slack workspace that accepts JSON payloads.
2. **OAuth2 App (Advanced)**: Generating a Bot token to send authentic, formatted app alerts.

### Full Official Guide
> 👉 **[n8n Docs: Setting up Slack Credentials](https://docs.n8n.io/integrations/builtin/credentials/slack/)**

## 8.4 Telegram Bot

For remote operators relying on mobile ping-backs, Telegram notifications are sent through the Bot API.
You need to talk to the `@BotFather` inside Telegram to create a bot and retrieve an **Access Token**. 

### Full Official Guide
> 👉 **[n8n Docs: Setting up Telegram Credentials](https://docs.n8n.io/integrations/builtin/credentials/telegram/)**

## 8.5 Updating the Built-in Workflows

After you have successfully created these credentials in your n8n workspace:
1. Open up **Workflow 1** and **Workflow 2**.
2. Double-click the respective **Alert System** nodes (Slack, Telegram, Gmail, Google Drive, and Google Sheets).
3. Under the *Credential for [Service]* dropdown menu, select the new accounts you just created.
4. Hit **Save** in the top right.
