---
date: '2025-07-22T23:24:14+05:00'
draft: false
title: 'Setting Up a Telegram Bot on Google Apps Script'
tags: ["apps-script", "telegram-bot"]
---

This is the technical companion to [Automating Business Reports with Google Apps Script and Telegram](/posts/apps-script/), which covers the project this code came from. Here I'll walk through running a Telegram bot on Apps Script: handling updates, routing messages, and managing the webhook.

## Receiving updates

Telegram delivers updates to your webhook as HTTP `POST` requests. In Apps Script, a `POST` to a deployed web app invokes the `doPost(e)` function, so that's where the bot starts:

```javascript
// Main webhook handler — receives all Telegram updates
function doPost(e) {
    try {
        if (!e || !e.postData) {
            return ContentService.createTextOutput('Bad Request');
        }

        const update = JSON.parse(e.postData.contents);

        if (update.message) {
            handleMessage(update.message);
        }
    } catch (error) {
        console.error('Error in doPost:', error);
    }

    // Apps Script web apps return HTTP 200 on any completed execution.
    // Telegram treats a 2xx response as "delivered" and won't retry, so we
    // let every request finish and return 200 — even after an error — to
    // avoid duplicate updates from retries. (Note: ContentService.TextOutput
    // has no way to set a custom status code; a thrown, uncaught error is
    // what produces a non-200 response and triggers Telegram retries.)
    return ContentService.createTextOutput('OK');
}
```

A key detail specific to Apps Script: you **cannot** set an arbitrary HTTP status code on a `ContentService` response — there is no `setStatusCode` method. A web app returns `200` whenever `doPost` runs to completion. This actually works in our favor: Telegram retries on non-`2xx` responses, and we don't want retries, so catching errors and returning normally is exactly the behavior we want. If the function throws and the error escapes, Apps Script returns a non-`200` response and Telegram will re-deliver the update.

## Routing messages

`handleMessage` inspects each message and decides what to do with it — ignore bots, dispatch commands, and process data coming from the relevant chats:

```javascript
function handleMessage(message) {
    try {
        // Ignore messages from bots or the bot itself
        if (message.from.is_bot) {
            return;
        }

        const text = message.text || '';
        const chatId = message.chat.id;

        echoMsg(message);

        // Handle commands
        if (text.startsWith('/')) {
            handleCommand(message, text);
            return;
        }

        // Capture data from groups and supergroups
        if (message.chat.type === 'group' || message.chat.type === 'supergroup') {
            processLoadMessage(message);
        }

    } catch (error) {
        console.error('Error handling message:', error);
    }
}
```

For this project the business data flowed exclusively through groups and supergroups, so those are the only chat types the bot acts on.

## Managing the webhook

Telegram needs to know where to send updates. These helpers register, remove, and inspect the webhook through the Bot API:

```javascript
function setWebhook() {
    const token = BOT_TOKEN;
    const url = WEB_APP_URL;

    UrlFetchApp.fetch(`https://api.telegram.org/bot${token}/setWebhook?url=${url}`);
}

// Remove the webhook — useful for stopping delivery immediately
function deleteWebhook() {
    try {
        const url = `${TELEGRAM_URL}/deleteWebhook`;
        const response = UrlFetchApp.fetch(url);
        console.log('Webhook deleted:', response.getContentText());
    } catch (error) {
        console.error('Error deleting webhook:', error);
    }
}

// Inspect the current webhook status
function getWebhookInfo() {
    try {
        const url = `${TELEGRAM_URL}/getWebhookInfo`;
        const response = UrlFetchApp.fetch(url);
        console.log('Webhook info:', response.getContentText());
    } catch (error) {
        console.error('Error getting webhook info:', error);
    }
}
```

## Getting the `WEB_APP_URL`

The webhook URL is the deployment URL of the script's web app. To create one:

1. Open the **Deploy** menu in the top-right of the editor.
2. Choose **New deployment**.
3. Select **Web app** as the deployment type.
4. Set **Who has access** to **Anyone**.
5. Click **Deploy**.

A successful deployment returns the web app URL. Assign it to `WEB_APP_URL` and call `setWebhook()` to point Telegram at it.

One caveat about redeploying: a new deployment produces a new URL. When you redeploy, open **Manage deployments**, archive the old deployment, deploy again, copy the new URL into `WEB_APP_URL`, then call `deleteWebhook()` followed by `setWebhook()` so Telegram is delivering to the current deployment.
