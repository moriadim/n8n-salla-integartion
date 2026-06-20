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

5. **Salla Order Refund Notification**
   - **Trigger**: Salla `order.refunded` webhook event.
   - **Sanitization**: Extracts customer name, order ID, formatted refund amount, and cleans phone number.
   - **Action**: Sends a WhatsApp confirmation template message confirming the processed refund.

6. **Salla WhatsApp Support AI Agent**
   - **Trigger**: WhatsApp incoming customer message webhook.
   - **LLM Model**: OpenAI Chat Model (`gpt-4o-mini`).
   - **Memory**: Window Buffer Memory using the customer's WhatsApp ID/phone number as the Session ID to maintain conversation context.
   - **Tool**: Google Sheets node connected as a Tool, configured to look up and read rows matching order numbers.
   - **Action**: Sends the AI's generated response directly back to the customer.

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
2. Open your n8n canvas.
3. Use the keyboard shortcut `Ctrl + V` (Windows/Linux) or `Cmd + V` (Mac) to paste the workflow directly onto the canvas, or click the **Workflow Menu** (top-right three dots) -> **Import from File** and upload the JSON.

---

## Workflow 1: Customer Login Welcome Message
*   **Trigger**: Webhook `POST /salla-customer-login`.
*   **Action**: Sanitizes phone number and sends Meta template `welcome_message`.

---

## Workflow 2: Order Status WhatsApp Notification
*   **Trigger**: Webhook `POST /salla-order-status-update`.
*   **Action**: Sanitizes phone, checks status, and routes to `Shipped` or `Out for Delivery` templates.

---

## Workflow 3: Order 24h Review Request
*   **Trigger**: Webhook `POST /salla-order-delivered-review`.
*   **Action**: IF status is `Delivered` / `تم التوصيل`, waits 24h, formats details, and sends template `order_review_request`.

---

## Workflow 4: Salla Abandoned Cart Recovery
*   **Trigger**: Webhook `POST /salla-abandoned-cart`.
*   **Action**: Captures checkout recovery link and sends template `cart_recovery_reminder`.

---

## Workflow 5: Salla Order Refund Notification
*   **Trigger**: Webhook `POST /salla-order-refund`.
*   **Action**: Captures order ID and refunded amount, and sends template `order_refunded_notification`.

---

## Workflow 6: Salla WhatsApp Support AI Agent

### Node 1: WhatsApp Webhook Node (Webhook)
*   **HTTP Method**: `POST`
*   **Path**: `incoming-whatsapp`
*   **Response Mode**: `onReceived`
*   **Setup**: Configure Meta WhatsApp Business App Webhook to point to this endpoint for the `messages` event.

### Node 2: AI Agent Node (Advanced AI Agent)
*   **Prompt Type**: Define
*   **Input text**: `Incoming message from customer: {{ $json.body.entry[0].changes[0].value.messages[0].text.body }}`
*   **System Prompt**:
    ```text
    You are a helpful customer support agent for a Salla store. You help customers with inquiries about their orders. You have access to a tool to look up order details in a Google Sheet. When someone asks about an order, identify the order number, look it up using the tool, and explain its status to them. Keep your answers concise, professional, and friendly.
    ```

### Node 3: OpenAI Chat Model (LM Chat OpenAI)
*   **Model**: `gpt-4o-mini`
*   **Connection**: Connected to the `ai_languageModel` input of the AI Agent.

### Node 4: Window Buffer Memory (Window Buffer Memory)
*   **Session Key**: `={{ $node["WhatsApp Webhook"].json.body.entry[0].changes[0].value.messages[0].from }}`  
    *(Uses the customer's phone number as the unique session key to separate threads)*
*   **Context Window Length**: 5
*   **Connection**: Connected to the `ai_memory` input of the AI Agent.

### Node 5: Google Sheets Tool (Google Sheets Tool)
*   **Operation**: `readRow` (Lookup/Read Row)
*   **Description**: `Lookup Salla order details (like order status, products, delivery date) from Google Sheets using the Order Number provided by the customer.`
*   **Connection**: Connected to the `ai_tool` input of the AI Agent.

### Node 6: Send WhatsApp Response Node (HTTP Request)
*   **HTTP Method**: `POST`
*   **URL**: `https://graph.facebook.com/v17.0/YOUR_PHONE_NUMBER_ID/messages`
*   **Headers**: `Authorization: Bearer YOUR_WHATSAPP_ACCESS_TOKEN`
*   **JSON Body**:
    ```json
    {
      "messaging_product": "whatsapp",
      "recipient_type": "individual",
      "to": "{{ $node[\"WhatsApp Webhook\"].json.body.entry[0].changes[0].value.messages[0].from }}",
      "type": "text",
      "text": {
        "body": "{{ $json.output }}"
      }
    }
    ```
    *(Directly returns the AI Agent's string output, `{{ $json.output }}`)*

---

## Repository Structure
- `salla-whatsapp-welcome.json`: Webhook welcome node workflow.
- `salla-whatsapp-order-status.json`: Order status update notification workflow.
- `salla-whatsapp-review-request.json`: 24-hour review request workflow.
- `salla-whatsapp-abandoned-cart.json`: Abandoned cart recovery workflow.
- `salla-whatsapp-refund.json`: Order refund notification workflow.
- `salla-whatsapp-ai-agent.json`: WhatsApp support AI Agent workflow.
- `README.md`: Setup and integration guidelines.
