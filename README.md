# Twitter Browser Bot Documentation

This document provides an overview of the Twitter Browser Bot implemented in `twitter-browser.js`. The bot automates interactions on Twitter (now X) using Puppeteer to monitor mentions, parse coin launch requests, and deploy Solana-based meme coins via the PumpPortal API. It's designed for reliability, with stealth features to mimic human behavior and avoid detection.

The goal here is to demonstrate how this technology works through key code snippets and explanations. This is not an installation guide—it's a technical showcase for developers and enthusiasts interested in browser automation, API integrations, and blockchain bots.

## Why This Works

This bot demonstrates effective Twitter automation by combining Puppeteer's browser control with careful UI handling (multi-selectors, delays) and integrations (SQLite, Firebase, Solana APIs). It's been tested in real scenarios, handling Twitter's anti-bot measures while launching coins reliably.

## Overview

The bot runs as a Node.js script that:
- Launches a browser instance (visible or headless).
- Logs into Twitter using credentials or cookies.
- Polls for mentions of a specific handle (e.g., `@LaunchBonkOnX`).
- Parses tweets for coin launch commands (e.g., `@bot $TICKER + Coin Name`).
- Processes images from tweets, uploads to IPFS, and launches tokens on Solana.
- Replies to users with launch details.
- Tracks processed tweets in SQLite to avoid duplicates.

It incorporates retries, random delays, and popup handling for robustness. Optional Firebase integration allows using pre-generated mint keys for coin launches.

## Key Features

- **Stealthy Automation**: Uses real Chrome profiles, random user agents, human-like typing, and delays.
- **Hybrid Mention Detection**: Checks both notifications and search results to catch all mentions.
- **Coin Launch Integration**: Handles image processing, IPFS uploads, and PumpPortal API calls.
- **Error Resilience**: Retries operations, recreates browser sessions, and logs failures.
- **Database Persistence**: SQLite for tracking processed tweets and stats.

## How It Works

The bot initializes in a class (`TwitterBrowserBot`), sets up configurations, and starts a loop to monitor mentions. When a mention is detected, it extracts data, launches a coin if valid, and replies. Here's a high-level flow:

1. **Initialization**: Set up browser, database, and configs.
2. **Login**: Handle authentication with fallbacks.
3. **Monitoring**: Poll notifications and search for mentions.
4. **Processing**: Parse tweet, download image, launch coin.
5. **Reply**: Post response and mark as processed.
6. **Cleanup**: On shutdown, save state and close resources.

Below are key code snippets that illustrate these components. Snippets are from `twitter-browser.js` and are simplified for clarity.

### 1. Bot Initialization and Configuration

The constructor sets up configs for stealth, timings, and integrations. It uses environment variables for flexibility.

```javascript:twitter-browser.js
class TwitterBrowserBot {
    constructor() {
        this.browser = null;
        this.page = null;
        this.isLoggedIn = false;
        this.botUsername = process.env.BOT_USERNAME || 'LaunchBonkOnX';
        this.config = {
            headless: process.env.HEADLESS === 'true' ? 'new' : false,
            userAgent: this.getRandomUserAgent(),
            pollIntervalMs: parseInt(process.env.POLL_INTERVAL_MS) || 30000, // 30s poll
            // ... other timings and limits
        };
        this.pumpPortalAPIKey = process.env.PUMPPORTAL_API_KEY;
        this.useCUSTOMLaunch = process.env.CUSTOM_LAUNCH === 'true';
        // ... database and session setup
        console.log(`🚀 TwitterBrowserBot initialized for @${this.botUsername}`);
    }

    getRandomUserAgent() {
        const userAgents = [ /* array of realistic user agents */ ];
        return userAgents[Math.floor(Math.random() * userAgents.length)];
    }
}
```

This shows how the bot randomizes user agents for stealth and configures polling intervals to check for mentions without overwhelming Twitter's servers.

### 2. Login to Twitter

Login uses multiple selectors for reliability and includes manual fallback if automation fails. It saves/loads cookies for session persistence.

```javascript:twitter-browser.js
async loginToTwitter() {
    console.log('🔐 Logging into Twitter...');
    await this.page.goto('https://x.com/login', { waitUntil: 'domcontentloaded' });
    // ... wait and screenshot for debugging

    // Try multiple selectors for username field
    const usernameSelectors = [ /* array of CSS selectors */ ];
    let usernameInput = null;
    for (const selector of usernameSelectors) {
        try {
            usernameInput = await this.page.$(selector);
            if (usernameInput) break;
        } catch (e) {}
    }
    if (!usernameInput) throw new Error('Username field not found');

    await this.typeHumanLike(usernameInput, process.env.TWITTER_USERNAME);
    // ... similar for "Next" button, password, and login button

    // Verify login success
    const loginSuccess = await this.checkLoginStatus();
    if (loginSuccess) {
        this.isLoggedIn = true;
        await this.saveCookies();
    }
}

async typeHumanLike(element, text) {
    await element.click();
    await this.randomDelay();
    for (const char of text) {
        await element.type(char);
        await new Promise(resolve => setTimeout(resolve, this.config.typingDelayMs));
    }
}
```

Human-like typing (`typeHumanLike`) adds delays between keystrokes to avoid detection. The method retries selectors due to Twitter's dynamic UI.

### 3. Monitoring Mentions

The bot uses a hybrid approach: checking notifications and search results, then deduplicating.

```javascript:twitter-browser.js
async checkMentions() {
    if (this.isProcessing) return;
    this.isProcessing = true;

    try {
        // Check notifications
        const notificationMentions = await this.checkNotificationsMentions();
        // Check search
        const searchMentions = await this.checkSearchMentions();
        // Combine and process unique mentions
        const uniqueMentions = this.deduplicateMentions([...notificationMentions, ...searchMentions]);
        for (const mention of uniqueMentions) {
            await this.processMentionData(mention);
        }
    } finally {
        this.isProcessing = false;
    }
}

async checkNotificationsMentions() {
    await this.page.goto('https://x.com/notifications');
    await this.clickAllNotificationsTab();
    const notifications = await this.getNotificationElements();
    // ... extract mentions from elements
}

deduplicateMentions(mentions) {
    const seen = new Set();
    const unique = [];
    for (const mention of mentions) {
        if (!seen.has(mention.tweetId)) {
            seen.add(mention.tweetId);
            unique.push(mention);
        }
    }
    return unique.sort((a, b) => b.timestamp - a.timestamp); // Newest first
}
```

This hybrid detection ensures no mentions are missed, even if Twitter's notification system lags. Deduplication prevents reprocessing.

### 4. Processing a Mention

Extracts tweet data, parses launch request, downloads image, launches coin, and replies.

```javascript:twitter-browser.js
async processMentionData(mentionData) {
    const { tweetId, tweetText, username } = mentionData;
    const launchData = this.parseLaunchRequest(tweetText);
    if (!launchData) return;

    // Extract image URL
    const imageUrl = /* ... find image in tweet */;
    if (!imageUrl) {
        await this.replyToTweet(tweetId, '❌ No image found!');
        return;
    }

    // Launch coin
    const result = await this.launchCoin(launchData, imageUrl, username, tweetId);
    // Reply
    await this.replyToTweet(tweetId, `🚀 Your coin $${launchData.symbol} is live! https://letsbonk.fun/token/${result.mintAddress}`);
    // Mark processed in DB
    await this.markTweetProcessed(tweetId, username, launchData.symbol /* ... */);
}

parseLaunchRequest(text) {
    const pattern = new RegExp(`@${this.botUsername}\\s+\\$([A-Z]+)\\s*\\+?\\s*(.+)`, 'i');
    const match = text.match(pattern);
    if (!match) return null;
    return { symbol: match[1].toUpperCase().substring(0, 7), name: match[2].trim() /* ... */ };
}
```

Parsing uses regex to extract ticker and name. The process checks for images and handles failures gracefully.

### 5. Launching a Coin

Downloads/processes image, uploads to IPFS, creates metadata, and calls PumpPortal API.

```javascript:twitter-browser.js
async launchCoin(launchData, imageUrl, authorDisplayName, tweetId, authorHandle) {
    // Generate or get mint key (Firebase optional)
    const mintKeypair = this.useCUSTOMLaunch ? /* Firebase key */ : Keypair.generate();

    // Process image
    const imageBuffer = await this.downloadAndProcessImage(imageUrl);
    const imgUris = await this.uploadImageToIPFS(imageBuffer);

    // Upload metadata
    const metadataUri = await this.uploadMetadataToIPFS({ name: launchData.name, symbol: launchData.symbol /* ... */ });

    // PumpPortal API call
    const response = await fetch('https://pumpportal.fun/api/trade?api-key=' + this.pumpPortalAPIKey, {
        method: 'POST',
        body: JSON.stringify({ /* tokenMetadata, mint, etc. */ })
    });
    const result = await response.json();
    return { mintAddress: mintKeypair.publicKey.toBase58(), signature: result.signature /* ... */ };
}
```

This integrates with Solana (`@solana/web3.js`), image processing (`sharp`), and IPFS uploads, culminating in a PumpPortal transaction.

### 6. Reply and Error Handling

Replies use retries and popup handling. The bot recreates sessions on errors like frame detachment.

```javascript:twitter-browser.js
async replyToTweet(tweetId, message) {
    await this.page.goto(`https://x.com/i/status/${tweetId}`);
    await this.page.click('[data-testid="reply"]');
    await this.page.type('[data-testid="tweetTextarea_0"]', message, { delay: 50 });
    await this.page.click('[data-testid="tweetButtonInline"]');
    // ... verify and handle popups
}

async retryOperation(operation, maxAttempts = 3) {
    for (let i = 0; i < maxAttempts; i++) {
        try { return await operation(); } catch (error) {
            console.log(`⚠️ Attempt ${i + 1} failed: ${error.message}`);
            await this.randomDelay();
        }
    }
    throw new Error('All attempts failed');
}
```

Retries (`retryOperation`) ensure flaky actions succeed, common in browser automation.
