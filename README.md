# Salla Store & Amazon Return Automations (n8n Workflows)

Hey there! This repository contains a set of ready-to-use n8n workflows that help automate customer communication (via WhatsApp), sync subscribers to Mailchimp, dispatch Telegram alerts, and track Amazon return notifications.

We've designed these to save you time and keep your store running smoothly.

---

## 🚀 How to Use These Workflows

Getting these up and running in your n8n instance is super simple:

1. **Grab the workflow JSON**: Open any of the `.json` files in this repository and copy all of the content.
2. **Import it into n8n**: Open a blank canvas in n8n, click the three dots in the top-right corner, select **Import from File** (or simply press `Ctrl + V` / `Cmd + V` on your keyboard) and paste the JSON.
3. **Configure your keys**: Double-click on the API, Google Sheets, or Telegram nodes to add your own API credentials, spreadsheet IDs, or tokens, and activate the workflow!

---

## 📦 What's Inside?

Here is a quick overview of the workflows included:

### 1. Salla Welcome Message (WhatsApp)
Sends a friendly WhatsApp welcome template to customers as soon as they log into your Salla store. It automatically cleans up phone numbers (removing `+` signs and spaces) so they play nice with the Meta API.
* *File*: `salla-whatsapp-welcome.json`

### 2. Salla Order Status Updates (WhatsApp)
Keeps customers in the loop. When an order status changes (like "Shipped" or "Out for Delivery"), it routes the update and fires off a custom WhatsApp message.
* *File*: `salla-whatsapp-order-status.json`

### 3. Salla 24h Review Request (WhatsApp)
Waits exactly 24 hours after an order is marked as delivered (تم التوصيل) and then politely asks the customer for a rating/review on WhatsApp.
* *File*: `salla-whatsapp-review-request.json`

### 4. Salla Abandoned Cart Recovery (WhatsApp)
Tries to win back lost sales by sending a WhatsApp reminder with a direct checkout link and a discount incentive when a customer leaves their cart behind.
* *File*: `salla-whatsapp-abandoned-cart.json`

### 5. Salla Refund Confirmation (WhatsApp)
Sends a quick confirmation WhatsApp text to the customer once a refund has been successfully processed for their order.
* *File*: `salla-whatsapp-refund.json`

### 6. WhatsApp AI Customer Support Agent
An advanced AI assistant powered by OpenAI (`gpt-4o-mini`). It listens to customer inquiries on WhatsApp, automatically remembers the chat context, looks up their order status from a Google Sheet, and texts them back a helpful answer.
* *File*: `salla-whatsapp-ai-agent.json`

### 7. Amazon Return Log & Alert Tracker
Keeps track of Amazon returns. It listens via a webhook or email inbox, parses the Return ID, item name, and status, logs it in a Google Sheets tracker, and sends a quick push notification to your admin Telegram channel.
* *File*: `amazon-return-tracker.json`

### 8. Salla Low-Stock Telegram Alerts
Dispatches an urgent low-stock notification to your purchasing manager's Telegram chat as soon as a product's quantity falls below its low threshold in Salla. Includes SKU, remaining quantity, and a link to check the product in your store dashboard.
* *File*: `salla-telegram-low-stock.json`

### 9. Salla to Mailchimp Customer Sync
Automatically adds newly registered Salla customers to your Mailchimp audience list. It checks to make sure the customer has an email address (avoiding errors for phone-only signups) and applies a 'Salla Customer' tag for easy segmentation.
* *File*: `salla-mailchimp-sync.json`

---

## 🛠️ Requirements & Setup

To make full use of these, you'll need:
- A self-hosted or cloud **n8n** instance.
- A **Meta Developer App** configured with the WhatsApp Business Cloud API.
- Credentials for **Google Sheets**, **OpenAI**, **Mailchimp**, or **Telegram** depending on the workflow you choose.

Feel free to customize these workflows to better fit your store's branding or specific needs!
