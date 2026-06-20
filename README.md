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

4. **Salla Abandoned Cart Recovery**
   - **Trigger**: Salla `abandoned.cart` webhook event.
   - **Sanitization**: Extracts customer's first name, recovery checkout URL, and cleans phone number.
   - **Action**: Sends a WhatsApp template reminder with a direct checkout link and discount incentive.

---

## Getting Started

### Import Workflow to n8n
To import any workflow:
1. Copy the contents of the respective JSON file:
   - [salla-whatsapp-welcome.json](file:///c:/Users/isaac/Desktop/salla%20workflows/salla-whatsapp-welcome.json)
   - [salla-whatsapp-order-status.json](file:///c:/Users/isaac/Desktop/salla%20workflows/salla-whatsapp-order-status.json)
   - [salla-whatsapp-review-request.json](file:///c:/Users/isaac/Desktop/salla%20workflows/salla-whatsapp-review-request.json)
   - [salla-whatsapp-abandoned-cart.json](file:///c:/Users/isaac/Desktop/salla%20workflows/salla-whatsapp-abandoned-cart.json)
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
* **Details**: Pauses execution. n8n offloads execution data to its database, ensuring no active memory is consumed during the delay.

### Node 4: Parse Customer Data Node (Code)
Extracts customer name and cleans the phone number.

### Node 5: Send WhatsApp Review Request Node (HTTP Request)
Sends template `order_review_request` with `customer_name` as parameter.

---

## Workflow 4: Salla Abandoned Cart Recovery

### Node 1: Salla Abandoned Cart Webhook Node (Webhook)
* **HTTP Method**: `POST`
* **Path**: `salla-abandoned-cart`
* **Response Mode**: `onReceived`
* **Salla Setup**:
  1. Go to your **Salla Partners Dashboard** -> App Settings.
  2. Navigate to **Webhooks**.
  3. Subscribe to the event `abandoned.cart`.
  4. Paste the n8n Webhook URL.

### Node 2: Parse Cart Data Node (Code)
Extracts and sanitizes checkout details:
* **Language**: JavaScript
* **Code**:
  ```javascript
  const items = $input.all();
  const output = [];

  for (const item of items) {
    const payload = item.json.body || item.json;
    const cartData = payload.data || {};
    const customer = cartData.customer || {};
    
    // Extract customer name
    let firstName = customer.first_name || '';
    if (!firstName && customer.name) {
      firstName = customer.name.split(' ')[0];
    }
    if (!firstName) {
      firstName = 'Customer';
    }
    
    // Extract phone number and clean it
    const rawPhone = customer.mobile || customer.phone || customer.phone_number || '';
    const cleanedPhone = String(rawPhone).replace(/[\s+]/g, '');
    
    // Extract checkout URL
    const checkoutUrl = cartData.checkout_url || cartData.url || '';
    
    output.push({
      json: {
        original_payload: payload,
        first_name: firstName,
        phone_number: cleanedPhone,
        checkout_url: checkoutUrl
      }
    });
  }

  return output;
  ```

### Node 3: Send WhatsApp Recovery Message Node (HTTP Request)
Sends the recovery reminder using Meta WhatsApp API:
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
      "name": "cart_recovery_reminder",
      "language": {
        "code": "en_US"
      },
      "components": [
        {
          "type": "body",
          "parameters": [
            { "type": "text", "text": "{{ $json.first_name }}" },
            { "type": "text", "text": "{{ $json.checkout_url }}" }
          ]
        }
      ]
    }
  }
  ```
  *(Make sure the template `"cart_recovery_reminder"` is pre-approved in Meta Developer Portal and contains variables for the user name and checkout link)*

---

## Repository Structure
- `salla-whatsapp-welcome.json`: Webhook-to-WhatsApp welcome node workflow.
- `salla-whatsapp-order-status.json`: Webhook-to-WhatsApp order status update notification workflow.
- `salla-whatsapp-review-request.json`: Webhook-to-WhatsApp 24-hour review request workflow.
- `salla-whatsapp-abandoned-cart.json`: Webhook-to-WhatsApp abandoned cart recovery workflow.
- `README.md`: Setup and integration guidelines.
