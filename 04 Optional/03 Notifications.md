# Overview

Notifications keep you informed about what is happening across your media server without requiring you to open every application individually.

In this setup, Discord acts as the central notification hub. Radarr, Sonarr, Prowlarr, Bazarr, and Jellyseerr can send messages to Discord through webhooks whenever specific events occur.

For example, you can receive notifications when a movie is imported, a TV episode finishes downloading, a subtitle is found, or a media request is approved.

> [!NOTE] Overview
> - **Notification platform:** Discord
> - **Integration method:** Discord Webhooks
> - **Applications:** Radarr, Sonarr, Prowlarr, Bazarr, Jellyseerr
> - **Configuration:** Each application's notification or connection settings
> - **Recommended approach:** Separate channels for different notification categories

## Table of Contents
- [1. Create a Discord Server](#1-create-a-discord-server)
- [2. Create Notification Channels](#2-create-notification-channels)
- [3. Create Discord Webhooks](#3-create-discord-webhooks)
- [4. Configure Radarr](#4-configure-radarr)
- [5. Configure Sonarr](#5-configure-sonarr)
- [6. Configure Prowlarr](#6-configure-prowlarr)
- [7. Configure Bazarr](#7-configure-bazarr)
- [8. Configure Jellyseerr](#8-configure-jellyseerr)
- [9. Test and Troubleshoot](#9-test-and-troubleshoot)

---
## 1. Create a Discord Server

If you do not already have a Discord server for your media stack, create one.
1. Open [Discord](https://discord.com/).
2. Create a new server.
3. Give it a recognizable name, such as `Media Server`.
4. Configure the server's membership and permissions according to your preferences.

You can use an existing private server if you prefer. A dedicated server simply keeps automated notifications separate from your other Discord activity.

> [!TIP] Keep It Private
> If your notifications contain information about your media library, usernames, or server activity, limit server membership to people you trust.

---
## 2. Create Notification Channels

Create text channels based on the type of notification you want to receive.

A simple starting structure could look like this:

```
Media Server
│
├── #movie-notifications
├── #tv-notifications
└── #server-activity
```

### Suggested Channel Purposes

|Channel|Purpose|
|---|---|
|`#movie-notifications`|Radarr movie events|
|`#tv-notifications`|Sonarr TV series and episode events|
|`#server-activity`|Prowlarr, Bazarr, Jellyseerr, and other operational events|

You can use a different structure if you prefer more granular notifications. For example, a dedicated `#requests` channel can keep Jellyseerr activity separate from technical events.

> [!TIP] Avoid Notification Overload
> Start with a small number of channels and enable only the events you actually care about. You can add more channels later if the volume becomes difficult to manage.

---
## 3. Create Discord Webhooks

A Discord webhook gives an application a URL through which it can send messages to a particular channel.

You do not need to create a Discord bot or host a separate notification service for basic webhook notifications.

### 3.1 Create a Webhook

For each channel you want to use:
1. Open your Discord server.
2. Go to **Server Settings**.
3. Open **Integrations** or the relevant Apps section.
4. Select **Webhooks**.
5. Create a new webhook.
6. Give it a recognizable name and picture.
7. Choose the destination channel.
8. Save the webhook.

The exact menu labels may vary slightly with Discord's interface.

### 3.2 Create Multiple Webhooks

I recommend creating separate webhooks for different notification groups. For example:

|Webhook Name|Destination Channel|Applications|
|---|---|---|
|`Radarr`|`#movie-notifications`|Radarr|
|`Sonarr`|`#tv-notifications`|Sonarr|
|`Media Services`|`#server-activity`|Prowlarr, Bazarr, Jellyseerr|

You can create more or fewer webhooks depending on how you want to organize your notifications.

**To make it fun**, you can also name the webhooks after certain stuff. For Example: In my server, each webhook is a character from the Witcher books and Games 

### 3.3 Copy the Webhook URL

Open the webhook you created and copy its URL. Paste that URL into the relevant application's notification settings.

> [!WARNING] Webhook URLs Are Secrets
> Anyone who obtains a Discord webhook URL may be able to post messages to its channel. Do not share webhook URLs publicly, commit them to Git, or include them in screenshots.
> If a webhook URL is exposed, regenerate or delete the webhook and update the affected applications.

---
## 4. Configure Radarr

Radarr manages your movie library. Discord notifications can tell you when a movie is grabbed, imported, added, or requires manual intervention.

### 4.1 Open the Notification Settings

Navigate to:

`Radarr → Settings → Connect`
1. Select **Add**.
2. Choose the Discord integration if it is available in your installed version.
3. Configure the notification settings.

### 4.2 Configure Discord

|Setting|Recommended Value|
|---|---|
|Name|`Discord`|
|Webhook URL|Your Radarr Discord webhook URL|
|Username|The webhook's configured name, if the integration exposes this field|

Enable the following notification triggers as a starting point:
- On Grab
- On File Import
- On Movie Added
- On Manual Interaction Required

The exact trigger labels may vary with the Radarr version and Discord integration.

**What these notifications mean**

|Trigger|Purpose|
|---|---|
|On Grab|Radarr has grabbed a release|
|On File Import|A media file has been imported into the library|
|On Movie Added|A movie has been added to Radarr|
|On Manual Interaction Required|Radarr requires attention or intervention|

Save the configuration and use the available test function to verify delivery.

---
## 5. Configure Sonarr

Sonarr manages your TV series and episodes. Its notifications are similar to Radarr's, but the event names refer to series and episodes rather than movies.

### 5.1 Open the Notification Settings

Navigate to:

`Sonarr → Settings → Connect`
1. Select **Add**.
2. Choose the Discord integration if available.
3. Configure the webhook and notification triggers.

### 5.2 Configure Discord

|Setting|Recommended Value|
|---|---|
|Name|`Discord`|
|Webhook URL|Your Sonarr Discord webhook URL|
|Username|The webhook's configured name, if supported|

Suggested notification triggers:
- On Grab
- On File Import
- On Import Complete
- On Series Add
- On Manual Interaction Required

The exact options depend on the Sonarr version and notification integration.

> [!NOTE] Series, Not Movies
> Sonarr manages TV series and episodes, so use the corresponding **Series Added** or **Series Add** event if available. The equivalent Radarr event is **Movie Added**.

**What these notifications mean**

|Trigger|Purpose|
|---|---|
|On Grab|Sonarr has grabbed a release|
|On File Import|An episode file has been imported|
|On Import Complete|An import operation has completed, if supported|
|On Series Add|A series has been added to Sonarr|
|On Manual Interaction Required|Sonarr needs attention|

Save the configuration and test the notification delivery.

---
## 6. Configure Prowlarr

Prowlarr manages indexers and synchronizes them with Radarr and Sonarr. Its notifications are useful for tracking release grabs and identifying indexer or application health issues.

### 6.1 Open the Notification Settings

Navigate to:

`Prowlarr → Settings → Notifications`
1. Select **Add**.
2. Choose Discord if available in your installed version.
3. Enter your webhook details.
4. Enable the desired triggers.

### 6.2 Configure Discord

|Setting|Recommended Value|
|---|---|
|Name|`Discord`|
|Webhook URL|Your Prowlarr Discord webhook URL|
|Username|The webhook's configured name, if supported|

Suggested triggers:
- On Release Grab
- Include Manual Grabs Made Inside Prowlarr
- On Health Issue
- On Health Restored

These events can help you distinguish routine activity from problems that need attention.

|Trigger|Purpose|
|---|---|
|On Release Grab|Reports release-grab events|
|Manual Grab Inclusion|Includes manual grabs when supported|
|On Health Issue|Alerts you to a reported health issue|
|On Health Restored|Reports when a health issue is resolved|

Save the settings and test the integration.

---
## 7. Configure Bazarr

Bazarr manages subtitle discovery and downloads for your media library. Notifications can be useful when subtitle-related events require attention.

### 7.1 Open the Notification Settings

Navigate to:

`Bazarr → Settings → Notifications`

Find the Discord integration or the supported webhook notification option in your version.

### 7.2 Configure Discord

Enter your Discord webhook URL and configure the supported notification events.

|Setting|Value|
|---|---|
|Webhook URL|Your Bazarr Discord webhook URL|
|Notification Events|Choose the subtitle-related events you want to receive|

The available options depend on your Bazarr version and the configured notification provider.

Potentially useful events include subtitle downloads, failed operations, and other relevant subtitle activity, if supported by the integration.

Save the settings and use the available test function if provided.

> [!TIP] Keep Subtitle Notifications Useful
> If Bazarr downloads subtitles frequently, notifying Discord for every routine event may create unnecessary noise. Consider enabling only the events that help you monitor your library or troubleshoot problems.

---
## 8. Configure Jellyseerr

Jellyseerr manages requests for movies and TV shows. Its Discord notifications can tell you when a request is awaiting approval, has been approved, or is available in your library.

### 8.1 Open the Notification Settings

Navigate to:

`Jellyseerr → Settings → Notifications`

Select Discord and configure the notification agent.

### 8.2 Configure Discord

|Setting|Recommended Value|
|---|---|
|Enable Agent|Yes|
|Webhook URL|Your Jellyseerr Discord webhook URL|
|Bot Username|The webhook's configured name|
|Enable Mentions|Yes, if you want supported mentions|

Enable the notification types you want to receive:
- Request Pending Approval
- Request Automatically Approved
- Request Approved
- Request Available

The exact labels and available events may vary by Jellyseerr version.

**What these notifications mean**

|Notification|Purpose|
|---|---|
|Request Pending Approval|A request is waiting for someone to approve it|
|Request Automatically Approved|A request has been approved automatically|
|Request Approved|A request has been approved|
|Request Available|The requested media is available in the library|

Save the configuration and send a test notification if the interface provides that option.

> [!WARNING] Mentions and Permissions
> Enabling mentions may allow supported notifications to mention users or roles. Check the configured mention behavior and Discord permissions so that notifications do not unexpectedly ping everyone.

---
## 9. Test and Troubleshoot

After configuring the applications, verify that notifications arrive in the intended Discord channels.

### 9.1 Test Each Integration

For each application:
1. Open its notification settings.
2. Use the test function if available.
3. Confirm that the message appears in the expected Discord channel.
4. Verify that the correct webhook and channel are being used.
5. Trigger a real event when appropriate to confirm the selected triggers work.

A successful test message confirms basic webhook connectivity, but it does not guarantee that every event trigger is configured correctly.

### 9.2 Notifications Are Not Arriving

Check the following:
- The webhook URL is correct.
- The webhook still exists in Discord.
- The destination channel is correct.
- The application has saved the settings.
- The desired event triggers are enabled.
- The application can reach Discord over the network.
- The application logs do not report notification errors.

If the webhook was regenerated or deleted, update the URL in the affected application.

### 9.3 Notifications Are Going to the Wrong Channel

Open Discord's webhook settings and verify the destination channel. Then confirm that the application is using the intended webhook URL.

Using separate webhooks for each application makes it easier to identify and correct routing mistakes.

### 9.4 Too Many Notifications

If Discord becomes noisy:
1. Disable low-value notification triggers.
2. Separate routine activity from health issues and failures.
3. Route different applications to different channels.
4. Keep manual-intervention and health alerts enabled where useful.

The goal is to make important events visible without burying them in routine activity.

---

> [!TIP] Notifications Configured
> Discord can now serve as the notification hub for your media stack. Once the integrations are tested, you can keep track of movie and episode imports, subtitle activity, indexer health, and media requests without opening every application individually.

---

> [!TIP] ➜ Up Next
> You’ve completed  [03 Notifications](#03-notifications)
> **Previous:** [02 Tailscale](02%20Tailscale.md) 
