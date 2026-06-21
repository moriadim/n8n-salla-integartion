# n8n Store Notification & Automation Workflows

This repository contains n8n workflows designed to automate customer notifications and tracking logs for Salla store events and Amazon returns.

## Workflows Included

1. **Salla Customer Login Welcome Message**
   - **Trigger**: Salla `customer.login` webhook event.
   - **Action**: Sends a welcome template message (`Welcome [Name]`) via Meta WhatsApp.

2. **Salla Order Status WhatsApp Notification**
   - **Trigger**: Salla `order.status.updated` webhook event.
   - **Action**: Routes based on status (`Shipped` / `Out for Delivery`) to send localized WhatsApp templates.

3. **Salla Order 24h Review Request**
   - **Trigger**: Salla order update webhook.
   - **Action**: If status is `Delivered` / `تم التوصيل`, pauses execution for 24 hours and sends a rating/review WhatsApp template.

4. **Salla Abandoned Cart Recovery**
   - **Trigger**: Salla `abandoned.cart` webhook event.
   - **Action**: Sends a WhatsApp template reminder with a direct checkout link and discount incentive.

5. **Salla Order Refund Notification**
   - **Trigger**: Salla `order.refunded` webhook event.
   - **Action**: Extracts refund amount and sends a WhatsApp confirmation template.

6. **Salla WhatsApp Support AI Agent**
   - **Trigger**: WhatsApp incoming customer message.
   - **Action**: Employs OpenAI Chat Model (`gpt-4o-mini`), Window Buffer Memory (Session ID based on customer number), and a Google Sheets Tool to answer order-related questions.

7. **Amazon Return Notification Tracker**
   - **Trigger**: Webhook node (direct JSON payload) or IMAP Email Trigger node (reads email).
   - **Parsing**: A JavaScript Code node uses regex patterns to extract Return ID, Item Name, and Status from emails, or extracts them directly from JSON fields.
   - **Logging**: Appends the parsed details to a tracking Google Sheet.
   - **Notification**: Pushes a Markdown-formatted message to an admin Telegram channel.

---

## Getting Started

### Import Workflow to n8n
To import any workflow:
1. Copy the contents of the respective JSON file:
   - [salla-whatsapp-welcome.json](file:///c:/Users/isaac/Desktop/salla%20workflows/salla-whatsapp-welcome.json)
   - [salla-whatsapp-order-status.json](file:///c:/Users/isaac/Desktop/salla%20workflows/salla-whatsapp-order-status.json)
   - [salla-whatsapp-review-request.json](file:///c:/Users/isaac/Desktop/salla%20workflows/salla-whatsapp-review-request.json)
   - [salla-whatsapp-abandoned-cart.json](file:///c:/Users/isaac/Desktop/salla%20workflows/salla-whatsapp-abandoned-cart.json)
   - [salla-whatsapp-refund.json](file:///c:/Users/isaac/Desktop/salla%20workflows/salla-whatsapp-refund.json)
   - [salla-whatsapp-ai-agent.json](file:///c:/Users/isaac/Desktop/salla%20workflows/salla-whatsapp-ai-agent.json)
   - [amazon-return-tracker.json](file:///c:/Users/isaac/Desktop/salla%20workflows/amazon-return-tracker.json)
2. Open your n8n canvas.
3. Paste the workflow (`Ctrl+V` or `Cmd+V`) directly onto the canvas, or click **Workflow Menu** -> **Import from File** and upload the JSON.

---

## Workflow 7: Amazon Return Notification Tracker

### Node 1: Amazon Return Webhook (Webhook Node)
*   **HTTP Method**: `POST`
*   **Path**: `amazon-return-notification`
*   **Response Mode**: `onReceived` (Acknowledge with `200 OK` immediately)

### Node 2: Amazon Return Email (Email Read IMAP Node)
*   **Function**: Connects to your email inbox (e.g., IMAP, Outlook, or Gmail) to listen for incoming return confirmation emails from Amazon.

### Node 3: Parse Return Data (Code Node)
Uses a JavaScript script to support both structured JSON payloads and email body texts.
*   **Code**:
    ```javascript
    const items = $input.all();
    const output = [];

    for (const item of items) {
      const payload = item.json.body || item.json;
      
      let returnId = 'N/A';
      let itemName = 'N/A';
      let status = 'Pending';
      
      const emailText = payload.text || payload.body || '';
      const emailSubject = payload.subject || '';
      
      if (emailText || emailSubject) {
        // Regex parse email body/subject
        const returnIdMatch = emailText.match(/(?:Return ID|Order ID|RMA):\s*([A-Za-z0-9-]+)/i) || 
                              emailSubject.match(/(?:Return ID|Order ID|RMA):\s*([A-Za-z0-9-]+)/i);
        if (returnIdMatch) {
          returnId = returnIdMatch[1];
        }
        
        const itemMatch = emailText.match(/(?:Item Name|Product|Item):\s*([^\r\n]+)/i);
        if (itemMatch) {
          itemName = itemMatch[1].trim();
        }
        
        const statusMatch = emailText.match(/(?:Status|Return Status):\s*([^\r\n]+)/i);
        if (statusMatch) {
          status = statusMatch[1].trim();
        }
      } else {
        // Direct JSON payload
        returnId = payload.return_id || payload.returnId || payload.order_id || payload.orderId || 'N/A';
        itemName = payload.item_name || payload.itemName || payload.product_name || 'N/A';
        status = payload.status || payload.return_status || 'Processed';
      }
      
      output.push({
        json: {
          original_payload: payload,
          return_id: returnId,
          item_name: itemName,
          status: status,
          logged_at: new Date().toISOString()
        }
      });
    }

    return output;
    ```

### Node 4: Log Return to Google Sheets (Google Sheets Node)
*   **Operation**: Append Row
*   **Spreadsheet ID**: `YOUR_SPREADSHEET_ID`
*   **Sheet Name**: `Sheet1`
*   **Column mapping**:
    *   `Return ID` -> `={{ $json.return_id }}`
    *   `Item Name` -> `={{ $json.item_name }}`
    *   `Status` -> `={{ $json.status }}`
    *   `Logged At` -> `={{ $json.logged_at }}`

### Node 5: Send Telegram Push Notification (Telegram Node)
Sends a styled message to your Telegram channel or chat:
*   **Chat ID**: `YOUR_TELEGRAM_ADMIN_CHANNEL_ID`
*   **Text**:
    ```text
    *New Amazon Return Logged!*

    📦 *Return ID:* {{ $json.return_id }}
    🛍️ *Item Name:* {{ $json.item_name }}
    🔄 *Status:* {{ $json.status }}
    📅 *Date:* {{ $json.logged_at }}
    ```
*   **Parse Mode**: `Markdown`

---

## Repository Structure
- `salla-whatsapp-welcome.json`
- `salla-whatsapp-order-status.json`
- `salla-whatsapp-review-request.json`
- `salla-whatsapp-abandoned-cart.json`
- `salla-whatsapp-refund.json`
- `salla-whatsapp-ai-agent.json`
- `amazon-return-tracker.json`
- `README.md`
