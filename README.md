# n8n Salla store to WhatsApp Integrations

This repository contains n8n workflows designed to automate customer notifications via Meta WhatsApp Business Cloud API based on Salla store events.

## Workflows Included

1. **Salla Customer Login Welcome Message**
   - **Trigger**: Salla `customer.login` webhook event.
   - **Sanitization**: Strips spaces and `+` from the customer's phone number.
   - **Action**: Sends a pre-approved Meta WhatsApp template message (`Welcome [Name]`).

2. **Salla Order Status WhatsApp Notification**
   - **Trigger**: Salla `order.status.updated` webhook event.
   - **Sanitization**: Extracts customer name, order ID, new status name, and cleans phone number (stripping spaces and `+`).
   - **Routing**: Employs a Switch Node to route based on the order status (`Shipped` or `Out for Delivery`).
   - **Action**: Sends localized WhatsApp templates specific to the status branch.

---

## Getting Started

### Import Workflow to n8n
To import any workflow:
1. Copy the contents of the respective JSON file:
   - [salla-whatsapp-welcome.json](file:///c:/Users/isaac/Desktop/salla%20workflows/salla-whatsapp-welcome.json)
   - [salla-whatsapp-order-status.json](file:///c:/Users/isaac/Desktop/salla%20workflows/salla-whatsapp-order-status.json)
2. Open your n8n canvas.
3. Use the keyboard shortcut `Ctrl + V` (Windows/Linux) or `Cmd + V` (Mac) to paste the workflow directly onto the canvas, or click the **Workflow Menu** (top-right three dots) -> **Import from File** and upload the JSON.

---

## Workflow 1: Customer Login Welcome Message

### Node 1: Salla Webhook Node (Webhook)
* **Type**: Webhook Node
* **HTTP Method**: `POST`
* **Path**: `salla-customer-login`
* **Response Mode**: `onReceived` (responds immediately to Salla with `200 OK`)
* **Salla Setup**: Subscribe to the `customer.login` event in Salla Partners App settings.

### Node 2: Parse Customer Data Node (Code)
* **Language**: JavaScript
* **Mode**: Run Once For All Items
* **Functionality**: Extracts customer's first name, identifies phone number fields, and strips `+` / spaces.

### Node 3: Send WhatsApp Welcome Node (HTTP Request)
* **HTTP Method**: `POST`
* **URL**: `https://graph.facebook.com/v17.0/YOUR_PHONE_NUMBER_ID/messages`
* **Headers**: `Authorization: Bearer YOUR_WHATSAPP_ACCESS_TOKEN`
* **Template name**: `welcome_message`
* **Template Parameter**: `{{ $json.first_name }}`

---

## Workflow 2: Order Status WhatsApp Notification

### Node 1: Salla Order Status Webhook Node (Webhook)
* **Type**: Webhook Node
* **HTTP Method**: `POST`
* **Path**: `salla-order-status-update`
* **Response Mode**: `onReceived`
* **Salla Setup**:
  1. Go to your **Salla Partners Dashboard** -> App Settings.
  2. Navigate to **Webhooks**.
  3. Subscribe to the event `order.status.updated`.
  4. Paste the n8n Webhook URL.

### Node 2: Parse Order Data Node (Code)
Extracts and sanitizes keys from the payload:
* **Language**: JavaScript
* **Code**:
  ```javascript
  const items = $input.all();
  const output = [];

  for (const item of items) {
    const payload = item.json.body || item.json;
    const orderData = payload.data || {};
    const customer = orderData.customer || {};
    const status = orderData.status || {};
    
    // Extract customer name
    const customerName = customer.name || customer.first_name || 'Customer';
    
    // Extract order ID
    const orderId = orderData.id || orderData.reference_id || 'N/A';
    
    // Extract status name
    const statusName = status.name || 'Updated';
    
    // Extract phone number and clean it
    const rawPhone = customer.mobile || customer.phone || customer.phone_number || '';
    const cleanedPhone = String(rawPhone).replace(/[\s+]/g, '');
    
    output.push({
      json: {
        original_payload: payload,
        customer_name: customerName,
        phone_number: cleanedPhone,
        order_id: orderId,
        status_name: statusName
      }
    });
  }

  return output;
  ```

### Node 3: Route by Status Node (Switch)
Routes the workflow to different branches depending on the new status:
* **Type**: Switch Node
* **Data Type**: String
* **Value 1**: `={{ $json.status_name }}`
* **Rules**:
  - **Index 0 (Shipped)**: If `status_name` Equals `Shipped`
  - **Index 1 (Out for Delivery)**: If `status_name` Equals `Out for Delivery`

### Node 4 & 5: WhatsApp API Nodes (HTTP Request)
For each matching status, an HTTP Request node makes a POST API request to Meta's WhatsApp Cloud API endpoint:
* **URL**: `https://graph.facebook.com/v17.0/YOUR_PHONE_NUMBER_ID/messages`
* **Headers**: `Authorization: Bearer YOUR_WHATSAPP_ACCESS_TOKEN`

#### Output 0: WhatsApp Shipped Notification
* **JSON Body**:
  ```json
  {
    "messaging_product": "whatsapp",
    "recipient_type": "individual",
    "to": "{{ $json.phone_number }}",
    "type": "template",
    "template": {
      "name": "order_shipped_notification",
      "language": {
        "code": "en_US"
      },
      "components": [
        {
          "type": "body",
          "parameters": [
            { "type": "text", "text": "{{ $json.customer_name }}" },
            { "type": "text", "text": "{{ $json.order_id }}" }
          ]
        }
      ]
    }
  }
  ```

#### Output 1: WhatsApp Out for Delivery Notification
* **JSON Body**:
  ```json
  {
    "messaging_product": "whatsapp",
    "recipient_type": "individual",
    "to": "{{ $json.phone_number }}",
    "type": "template",
    "template": {
      "name": "order_out_for_delivery_notification",
      "language": {
        "code": "en_US"
      },
      "components": [
        {
          "type": "body",
          "parameters": [
            { "type": "text", "text": "{{ $json.customer_name }}" },
            { "type": "text", "text": "{{ $json.order_id }}" }
          ]
        }
      ]
    }
  }
  ```

---

## Repository Structure
- `salla-whatsapp-welcome.json`: Webhook-to-WhatsApp welcome node workflow.
- `salla-whatsapp-order-status.json`: Webhook-to-WhatsApp order status update notification workflow.
- `README.md`: Setup and integration guidelines.
