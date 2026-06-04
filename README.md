# CyberResonance.io Uptime & Readability Monitor

A professional-grade, state-aware n8n workflow designed to monitor `https://cyberresonance.io` for status uptime and content readability. If the website goes down or fails the validation checks, the workflow runs a DNS lookup diagnostics test and alerts you via Discord.

---

## 🌟 Key Features

1. **Uptime & Readability Checks**:
   - Performs an HTTP GET request to `https://[...]` with a `10s` timeout.
   - Sends a custom browser `User-Agent` header to bypass Cloudflare/Hostinger CDN security challenge pages.
   - Verifies the response status code is `200`.
   - Inspects the HTML body to ensure readability using keywords

2. **Out-of-Band DNS Diagnostics**:
   - If the site is unreachable or fails readability, the workflow queries Cloudflare's public DNS-over-HTTPS API (`https://cloudflare-dns.com/dns-query`) for the `A` records of `https://[...]`.
   - This differentiates between DNS propagation/resolution issues and server-side failures (e.g. webserver crashes).

3. **Intelligent State-Aware Notification (No Spam)**:
   - Uses n8n's persistent static data (`getWorkflowStaticData('global')`).
   - Keeps track of the last known state (`up` or `down`).
   - Triggers a Discord alert **only on status transition** (e.g. when the site goes down, and when it recovers). You will not be spammed every 15 minutes!

4. **Premium Discord Embeds**:
   - **Down Alert**: Red-themed embed with failure details, HTTP status code, and DNS diagnostic resolution details.
   - **Recovery Alert**: Green-themed embed celebrating the site's return to service.

---

## 📁 Project Structure

* [Website_Monitor_Workflow.json]: The exportable JSON workflow file for direct import into n8n.
* [Website up or down.md]: Context and tutorial helper.
* [README.md]: This documentation.

---

## 🚀 Setup & Installation Instructions

Follow these simple steps to deploy the monitor in your n8n instance:

### Step 1: Import into n8n
1. Open your n8n canvas.
2. Open the file [Website_Monitor_Workflow.json] and copy its entire content.
3. Paste the contents directly into your n8n workspace canvas (or select **Import from File** in the top-right menu).

### Step 2: Configure the Discord Webhook
1. Open the **Send Discord Notification** HTTP Request node (the last node in the flow).
2. Replace `YOUR_DISCORD_WEBHOOK_URL_HERE` in the **URL** parameter with your actual Discord channel webhook URL.
3. Save the workflow.

### Step 3: Enable the Schedule Trigger
1. By default, the **Schedule Trigger** is configured to run every 15 minutes. You can customize the frequency in the node's settings.
2. Toggle the workflow to **Active** in the top-right corner of the n8n interface.

---

## 🛠️ How it Works under the Hood

### State Machine Logic
The **Check State & Filter Notifications** node runs the following JavaScript to keep track of the site's status:
```javascript
const items = $input.all();
const processed = [];

// Retrieve n8n static data
const staticData = getWorkflowStaticData('global');
const previousState = staticData.previousState || 'up';

for (const item of items) {
  const json = item.json;
  const currentState = json.isHealthy ? 'up' : 'down';
  
  let triggerAlert = false;
  let alertType = '';
  
  if (currentState === 'down' && previousState === 'up') {
    triggerAlert = true;
    alertType = 'down';
    staticData.previousState = 'down';
  } else if (currentState === 'up' && previousState === 'down') {
    triggerAlert = true;
    alertType = 'recovery';
    staticData.previousState = 'up';
  } else {
    triggerAlert = false;
  }
  
  processed.push({
    json: {
      ...json,
      previousState,
      currentState,
      triggerAlert,
      alertType
    }
  });
}

return processed;
```

### Discord Notification Templates
The workflow sends custom rich Discord embed payloads based on the transition state:

#### Down Embed
* **Color**: Red (`#E74C3C` / Decimal `15158332`)
* **Title**: `🚨 CyberResonance.io is DOWN!`
* **Description**: Includes the failure reason and DNS diagnostic results.
* **Fields**: Displays HTTP Status Code and UTC Timestamp.

#### Recovery Embed
* **Color**: Green (`#2ECC71` / Decimal `3066993`)
* **Title**: `✅ CyberResonance.io is UP & Healthy`
* **Description**: Confirms the site is back online and readable.
* **Fields**: Displays HTTP Status Code and UTC Timestamp.
