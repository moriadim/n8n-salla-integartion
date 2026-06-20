# n8n Salla store to WhatsApp Integrations

This repository contains n8n workflows designed to automate customer notifications via Meta WhatsApp Business Cloud API based on Salla store events.

## Workflows Included

1. **Salla Customer Login Welcome Message**
   - **Trigger**: Salla `customer.login` webhook event.
   - **Sanitization**: Strips spaces and `+` from the customer's phone number.
   - **Action**: Sends a pre-approved Meta WhatsApp welcome template message (`Welcome [Name]`).

2. **Salla Order Status WhatsApp Notification**
   - **Trigger**: Salla `order.status.updated` webhook event.
   - **Sanitization**: Extracts customer name, order ID, new status name, and cleans phone number.
   - **Routing**: Employs a Switch Node to route based on the order status (`Shipped` or `Out for Delivery`).
   - **Action**: Sends localized WhatsApp templates specific to the status branch.

3. **Salla Order 24h Review Request**
   - **Trigger**: Salla order updates webhook.
   - **Condition**: IF Node validates that the order status strictly equals `Delivered` or `تم التوصيل`. If not, workflow stops.
   - **Delay**: Wait Node pauses execution for exactly 24 hours.
   - **Sanitization**: Cleans the customer's phone number and extracts their name.
   - **Action**: Sends a WhatsApp template requesting feedback/rating.

---

## Getting Started

### Import Workflow to n8n
To import any workflow:
1. Copy the contents of the respective JSON file:
   - [salla-whatsapp-welcome.json](file:///c:/Users/isaac/Desktop/salla%20workflows/salla-whatsapp-welcome.json)
   - [salla-whatsapp-order-status.json](file:///c:/Users/isaac/Desktop/salla%20workflows/salla-whatsapp-order-status.json)
   - [salla-whatsapp-review-request.json](file:///c:/Users/isaac/Desktop/salla%20workflows/salla-whatsapp-review-request.json)
2. Open your n8n canvas.
3. Use the keyboard shortcut `Ctrl + V` (Windows/Linux) or `Cmd + V` (Mac) to paste the workflow directly onto the canvas, or click the **Workflow Menu** (top-right three dots) -> **Import from File** and upload the JSON.

---

## Workflow 1: Customer Login Welcome Message

### Node 1: Salla Webhook Node (Webhook)
* **HTTP Method**: `POST`
* **Path**: `salla-customer-login`
* **Response Mode**: `onReceived` (responds immediately with `200 OK`)
* **Salla Setup**: Subscribe to the `customer.login` event in Salla Partners App settings.

### Node 2: Parse Customer Data Node (Code)
* **Functionality**: Extracts customer's first name, identifies phone number fields, and strips `+` / spaces.

### Node 3: Send WhatsApp Welcome Node (HTTP Request)
* **URL**: `https://graph.facebook.com/v17.0/YOUR_PHONE_NUMBER_ID/messages`
* **Headers**: `Authorization: Bearer YOUR_WHATSAPP_ACCESS_TOKEN`
* **Template name**: `welcome_message`
* **Template Parameter**: `{{ $json.first_name }}`

---

## Workflow 2: Order Status WhatsApp Notification

### Node 1: Salla Order Status Webhook Node (Webhook)
* **HTTP Method**: `POST`
* **Path**: `salla-order-status-update`
* **Response Mode**: `onReceived`
* **Salla Setup**: Subscribe to the event `order.status.updated` in Salla Partners.

### Node 2: Parse Order Data Node (Code)
Extracts and sanitizes keys from the payload (first_name, phone_number, order_id, status_name).

### Node 3: Route by Status Node (Switch)
* **Value 1**: `={{ $json.status_name }}`
* **Rules**:
  - **Index 0 (Shipped)**: If `status_name` Equals `Shipped`
  - **Index 1 (Out for Delivery)**: If `status_name` Equals `Out for Delivery`

### Node 4 & 5: WhatsApp API Nodes (HTTP Request)
* **Output 0**: Sends template `order_shipped_notification` with `customer_name` and `order_id` as arguments.
* **Output 1**: Sends template `order_out_for_delivery_notification` with `customer_name` and `order_id` as arguments.

---

## Workflow 3: Order 24h Review Request

### Node 1: Salla Order Webhook Node (Webhook)
* **HTTP Method**: `POST`
* **Path**: `salla-order-delivered-review`
* **Response Mode**: `onReceived`
* **Salla Setup**: Subscribe to order update webhooks in Salla Partners.

### Node 2: Is Delivered? Node (IF)
* **Conditions (Combine: ANY)**:
  - Check if `{{ $json.body.data.status.name }}` Equals `Delivered`
  - Check if `{{ $json.body.data.status.name }}` Equals `تم التوصيل`
* **Workflow branching**: If True, continues to Wait node. If False, stops.

### Node 3: Wait 24 Hours Node (Wait)
* **Amount**: `24`
* **Unit**: `hours`
* **Details**: Pauses execution. n8n offloads execution data to its database, ensuring no active memory consumption during the delay.

### Node 4: Parse Customer Data Node (Code)
* **Language**: JavaScript
* **Code**:
  ```javascript
  const items = $input.all();
  const output = [];

  for (const item of items) {
    const payload = item.json.body || item.json;
    const orderData = payload.data || {};
    const customer = orderData.customer || {};
    
    // Extract customer name
    const customerName = customer.name || customer.first_name || 'Customer';
    
    // Extract phone number and clean it
    const rawPhone = customer.mobile || customer.phone || customer.phone_number || '';
    const cleanedPhone = String(rawPhone).replace(/[\s+]/g, '');
    
    output.push({
      json: {
        original_payload: payload,
        customer_name: customerName,
        phone_number: cleanedPhone
      }
    });
  }

  return output;
  ```

### Node 5: Send WhatsApp Review Request Node (HTTP Request)
* **HTTP Method**: `POST`
* **URL**: `https://graph.facebook.com/v17.0/YOUR_PHONE_NUMBER_ID/messages`
* **Headers**: `Authorization: Bearer YOUR_WHATSAPP_ACCESS_TOKEN`
* **JSON Body**:
  ```json
  {
    "messaging_product": "whatsapp",
    "recipient_type": "individual",
    "to": "{{ $json.phone_number }}",
    "type": "template",
    "template": {
      "name": "order_review_request",
      "language": {
        "code": "en_US"
      },
      "components": [
        {
          "type": "body",
          "parameters": [
            { "type": "text", "text": "{{ $json.customer_name }}" }
          ]
        }
      ]
    }
  }
  ```
  *(Make sure the template `"order_review_request"` is pre-approved in Meta Developer Portal)*

---

## Repository Structure
- `salla-whatsapp-welcome.json`: Webhook-to-WhatsApp welcome node workflow.
- `salla-whatsapp-order-status.json`: Webhook-to-WhatsApp order status update notification workflow.
- `salla-whatsapp-review-request.json`: Webhook-to-WhatsApp 24-hour review request workflow.
- `README.md`: Setup and integration guidelines.
