# n8n Salla store to WhatsApp Welcome Integration

This repository contains an n8n workflow designed to send a welcome WhatsApp message to customers when they log into your Salla store.

## Workflow Overview

The workflow consists of three primary nodes connected in sequence:
1. **Webhook Node**: Listens for the `customer.login` event from Salla.
2. **Code Node (JavaScript)**: Parses the payload, extracts customer details, and cleans the phone number (stripping out any spaces and `+` signs).
3. **HTTP Request Node**: Sends a pre-approved Meta WhatsApp template message to the customer using the Meta WhatsApp Business Cloud API.

---

## Getting Started

### 1. Import Workflow to n8n
To import this workflow:
1. Copy the contents of [salla-whatsapp-welcome.json](file:///c:/Users/isaac/Desktop/salla%20workflows/salla-whatsapp-welcome.json).
2. Open your n8n canvas.
3. Use the keyboard shortcut `Ctrl + V` (Windows/Linux) or `Cmd + V` (Mac) to paste the workflow directly onto the canvas, or click the **Workflow Menu** (top-right three dots) -> **Import from File** and upload the JSON.

---

## Node Configurations & Detailed Guide

### Node 1: Salla Webhook Node (Webhook)
* **Type**: Webhook Node
* **HTTP Method**: `POST`
* **Path**: `salla-customer-login`
* **Response Mode**: `onReceived` (responds immediately to Salla with `200 OK` to confirm receipt)
* **Salla Setup**:
  1. Go to your **Salla Partners Dashboard** and select your app.
  2. Navigate to the **Webhooks** section.
  3. Add a new webhook subscription for the event `customer.login`.
  4. Copy the production **Test/Webhook URL** from your n8n Webhook node and paste it into the Salla Webhook URL input.
  5. Save the configuration.

### Node 2: Parse Customer Data Node (Code)
This JavaScript Code node extracts and sanitizes data from the webhook body:
* **Language**: JavaScript
* **Mode**: Run Once For All Items
* **Code**:
  ```javascript
  const items = $input.all();
  const output = [];

  for (const item of items) {
    const payload = item.json.body || item.json;
    const customer = payload.data || {};
    
    // Extract customer first name (fallback to first word of fullname or default 'Customer')
    let firstName = customer.first_name || '';
    if (!firstName && customer.name) {
      firstName = customer.name.split(' ')[0];
    }
    if (!firstName) {
      firstName = 'Customer';
    }
    
    // Extract phone number from potential properties (mobile, phone_number, phone)
    const rawPhone = customer.mobile || customer.phone_number || customer.phone || '';
    
    // Sanitize phone number (strip spaces and '+')
    const cleanedPhone = String(rawPhone).replace(/[\s+]/g, '');
    
    output.push({
      json: {
        original_payload: payload,
        first_name: firstName,
        phone_number: cleanedPhone
      }
    });
  }

  return output;
  ```

### Node 3: Send WhatsApp Welcome Node (HTTP Request)
This node makes a direct POST API request to Meta's WhatsApp Cloud API endpoint:
* **HTTP Method**: `POST`
* **URL**: `https://graph.facebook.com/v17.0/YOUR_PHONE_NUMBER_ID/messages`  
  *(Replace `YOUR_PHONE_NUMBER_ID` with your Meta Business Phone Number ID)*
* **Headers**:
  * `Authorization`: `Bearer YOUR_WHATSAPP_ACCESS_TOKEN`  
    *(Replace `YOUR_WHATSAPP_ACCESS_TOKEN` with your Meta Permanent/Temporary Access Token)*
  * `Content-Type`: `application/json`
* **Body (JSON)**:
  ```json
  {
    "messaging_product": "whatsapp",
    "recipient_type": "individual",
    "to": "{{ $json.phone_number }}",
    "type": "template",
    "template": {
      "name": "welcome_message",
      "language": {
        "code": "en_US"
      },
      "components": [
        {
          "type": "body",
          "parameters": [
            {
              "type": "text",
              "text": "{{ $json.first_name }}"
            }
          ]
        }
      ]
    }
  }
  ```
  *(Make sure the template `"welcome_message"` is pre-approved in your Meta App Dashboard under WhatsApp Manager)*

---

## Repository Structure
- `salla-whatsapp-welcome.json`: n8n workflow configuration file.
- `README.md`: Setup and integration guidelines.
