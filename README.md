This is an ambitious and impressive automation project for an f-commerce business! Building an AI-powered Messenger bot with real-time sales and order management using n8n and Meta's Commerce API will significantly enhance your customer experience.

The N8n workflow JSON you provided (`fb_messenger_features.json`) is an excellent starting point, demonstrating how to use various Messenger API features like `mark_seen`, `typing_on`, `url_button`, `persistent_menu`, `postback_button`, `call_button`, `quick_replies`, `receipt_template`, and `Templates`.

However, the provided workflow is a collection of feature demonstrations that currently trigger all at once. To build a robust f-commerce bot, we need to integrate these features dynamically based on user intent and conversational flow, driven by an AI agent.

Below, I'll provide a **High-Level Architecture** for your n8n automation, followed by a **Step-by-Step Guide with the N8n Workflow Code**, incorporating your existing features into the AI-driven logic.

---

## High-Level Architecture for N8n F-Commerce Automation

The automation will operate as a conversational AI agent, using n8n as the orchestrator, integrating with Facebook Messenger, an AI/LLM service, and your backend e-commerce system (product catalog, cart, orders, user state).

```mermaid
graph TD
    A[Facebook Messenger User] --> B{Messenger Webhook Trigger (n8n)};

    subgraph n8n Workflow - Core Loop
        B -- Incoming Message/Postback --> C[Webhook Verification (If Node)];
        C -- Valid --> D[Mark Seen & Typing On (HTTP Request)];
        D --> E[Parse Message/Payload (Set Node)];
        E --> F[Combine User Input (Code Node)];
        F --> G{AI Intent Recognition (LLM Node)};
        G -- Identified Intent --> H{Switch by Intent};
    end

    subgraph n8n Workflow - Intent Branches
        H -- product_info/recommendation --> I[Product Search Branch];
        H -- add_to_cart_initial --> J[Add to Cart Branch];
        H -- select_variant --> K[Select Variant Branch];
        H -- set_quantity --> L[Set Quantity Branch];
        H -- view_cart --> M[View Cart Branch];
        H -- proceed_to_checkout --> N[Initiate Checkout Branch];
        H -- address_input --> O[Address Input Handling Branch];
        H -- confirm_address --> P[Order Confirmation/Creation Branch];
        H -- order_cancel --> Q[Order Cancellation Branch];
        H -- delivery_info --> R[Delivery Tracking Branch];
        H -- business_info --> S[Business Info/FAQ Branch];
        H -- general_greeting/general_query --> T[General Response Branch];
    end

    subgraph External Systems
        G -- Queries --> LLM[Large Language Model (e.g., OpenAI, Gemini)];
        I -- Queries --> ECOM_API[E-commerce API (Products)];
        J -- Adds --> ECOM_API;
        K -- Fetches --> ECOM_API;
        L -- Updates --> ECOM_API;
        M -- Fetches --> ECOM_API;
        N -- Updates --> ECOM_API;
        O -- Updates --> DB_USER_STATE[Database: User State];
        P -- Creates/Updates --> ECOM_API & DB_USER_STATE;
        Q -- Updates --> ECOM_API;
        R -- Fetches --> ECOM_API/COURIER_API[Courier Tracking API];
        S -- Queries --> KB[Knowledge Base (Vector DB / LLM)];
    end

    I --> U[Format Product Carousel (Templates)];
    J -- If Variants --> V[Quick Replies for Variants];
    K --> W[Quick Replies for Quantity];
    L --> X[Add to Cart Confirmation];
    M --> Y[Cart Summary + Checkout Button];
    O --> Z[Address Confirmation + Buttons];
    P --> AA[Send Receipt Template];
    Q --> BB[Order Cancel Confirmation];
    R --> CC[Tracking Info + URL Button];
    S --> DD[AI Business Info Response];
    T --> EE[General AI Response];

    U, V, W, X, Y, Z, AA, BB, CC, DD, EE --> F_OFF[Typing Off (HTTP Request)];
    F_OFF --> F_OUT[Send Messenger Response (HTTP Request)];
    F_OUT --> B;

    subgraph One-Time Setup Workflow
        OM[One-time Manual Trigger] --> P_MENU[Set Persistent Menu (HTTP Request)];
        P_MENU --> B;
    end
```

---

## Step-by-Step Guild and N8n Workflow Code

We will integrate your `fb_messenger_features.json` nodes where applicable. I'll use `v19.0` as the Facebook API version; please update it to the latest stable version if different.

**Important Setup Notes:**

1.  **Facebook Page Access Token & App Secret:** Replace `EAARmZBZCk9xtsBPGH90Ra6tjPgQvNSZCwlcjqH2PaGEOU0Apbyf7HgGibPPHhOSkYRcZAytUpPZBdhsJubKyQD3jLurB4xlO2ssSSVYWZCZBKSXHCBxUaNEM8XQMPITatd5ZAPp8f6cY7RVNWC6tjEiojKtfLpNHGEimCjoHV5mE7OjxMt72Fit2nRjleBFcX8VgIaVTTgy3ngZDZD` with your actual, secure Page Access Token. Do not hardcode it in the node URL; use n8n Credentials. I've left it as is from your JSON for continuity, but this is critical.
2.  **Facebook API Version:** I've updated `v23.0` to `v19.0`. Please verify the current version.
3.  **External APIs:** Replace placeholder URLs like `https://yourstore.com/api/products` with your actual backend API endpoints.
4.  **AI Service:** You'll need credentials for your chosen LLM (e.g., OpenAI API Key).
5.  **User State Database:** Implement a simple database (e.g., PostgreSQL, MongoDB, Airtable, or even Redis for temporary state) to store user-specific conversational state (e.g., `awaiting_address_input`, `last_product_id`, `cart_id`).

---

### **1. Core Workflow: Messenger Ingestion & AI Routing**

This is the main workflow that listens for messages, processes them, and routes to the appropriate AI-driven logic.

```json
{
  "name": "F-Commerce Messenger Automation",
  "nodes": [
    {
      "parameters": {
        "multipleMethods": true,
        "path": "webhook_fb_messenger",
        "responseMode": "responseNode",
        "options": {}
      },
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2.1,
      "position": [
        240,
        140
      ],
      "id": "3861c445-5c91-41e3-b3e6-c28aa3ca7420",
      "name": "1. Webhook (FB Messenger Trigger)",
      "webhookId": "059a0cd0-d6f2-4821-a89c-6ba82cc3585a"
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 2
          },
          "conditions": [
            {
              "id": "95cc6b73-e467-463c-af3a-3f3bc4caf6fc",
              "leftValue": "={{ $json.query['hub.mode'] }}",
              "rightValue": "subscribe",
              "operator": {
                "type": "string",
                "operation": "equals",
                "name": "filter.operator.equals"
              }
            },
            {
              "id": "f519ec71-362b-4e70-83ed-8ba97493c619",
              "leftValue": "={{ $json.query['hub.verify_token'] }}",
              "rightValue": "chartbot",
              "operator": {
                "type": "string",
                "operation": "equals",
                "name": "filter.operator.equals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.2,
      "position": [
        480,
        140
      ],
      "id": "8e1066c2-c08c-494c-a7dd-e1c0f5c1deeb",
      "name": "2. If (Webhook Verification)"
    },
    {
      "parameters": {
        "respondWith": "text",
        "responseBody": "={{ $json.query['hub.challenge'] }}",
        "options": {}
      },
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1.4,
      "position": [
        720,
        140
      ],
      "id": "1eae7c71-ce1a-4cc9-a655-5726aaf15434",
      "name": "3. Respond to Webhook (Verification)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.body.entry[0].messaging[0].sender.id }}\"\n  },\n  \"sender_action\": \"mark_seen\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        480,
        340
      ],
      "id": "aa2af423-f387-4565-b412-4ec03a0ad8b4",
      "name": "4. Mark Seen"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.body.entry[0].messaging[0].sender.id }}\"\n  },\n  \"sender_action\": \"typing_on\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        720,
        340
      ],
      "id": "d4be1110-758e-4432-9eb8-3de1b5881df2",
      "name": "5. Typing On"
    },
    {
      "parameters": {
        "values": [
          {
            "name": "PSID",
            "value": "={{ $json.body.entry[0].messaging[0].sender.id }}"
          },
          {
            "name": "MessageText",
            "value": "={{ $json.body.entry[0].messaging[0].message?.text || '' }}"
          },
          {
            "name": "Payload",
            "value": "={{ $json.body.entry[0].messaging[0].postback?.payload || $json.body.entry[0].messaging[0].message?.quick_reply?.payload || '' }}"
          },
          {
            "name": "Referral",
            "value": "={{ $json.body.entry[0].messaging[0].referral?.ref || '' }}"
          },
          {
            "name": "UserEmail",
            "value": "={{ $json.body.entry[0].messaging[0].message?.quick_reply?.payload === 'user_email' ? $json.body.entry[0].messaging[0].message.text : '' }}"
          },
          {
            "name": "UserPhoneNumber",
            "value": "={{ $json.body.entry[0].messaging[0].message?.quick_reply?.payload === 'user_phone_number' ? $json.body.entry[0].messaging[0].message.text : '' }}"
          }
        ],
        "options": {}
      },
      "type": "n8n-nodes-base.set",
      "typeVersion": 1,
      "position": [
        960,
        340
      ],
      "id": "2d94a9a0-d7c3-4d4b-9d41-b0e6e73715c0",
      "name": "6. Set (Parse Messenger Event)"
    },
    {
      "parameters": {
        "functionCode": "const messageText = $json.MessageText;\nconst payload = $json.Payload;\nlet fullUserInput = messageText;\n\nif (payload && payload.length > 0) {\n  if (messageText && messageText.length > 0) {\n    fullUserInput = `${messageText} (Payload: ${payload})`;\n  } else {\n    fullUserInput = `(Payload: ${payload})`;\n  }\n}\n\nreturn {\n    json: {\n        PSID: $json.PSID,\n        fullUserInput: fullUserInput,\n        MessageText: messageText,\n        Payload: payload,\n        UserEmail: $json.UserEmail,\n        UserPhoneNumber: $json.UserPhoneNumber\n    }\n};"
      },
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [
        1200,
        340
      ],
      "id": "d049f537-b45b-4260-848f-3de7d23a105c",
      "name": "7. Code (Prepare AI Input)"
    },
    {
      "parameters": {
        "model": "gpt-4o",
        "authentication": "credentials",
        "credentials": {
          "credentialData": {
            "apiKey": "={{ $credentials.openAiApi.apiKey }}"
          }
        },
        "messages": [
          {
            "role": "system",
            "content": "You are an AI assistant for an f-commerce business on Facebook Messenger. Your primary goal is to identify the user's intent based on their message and/or a provided payload. Prioritize payloads for explicit actions. Output ONLY the intent string. If unsure, default to 'general_query'.\n\n**Available Intents:**\n- `product_info`: User is asking about a product, specific item, or browsing products. Triggered also by `VIEW_DETAILS_PRODUCT_ID_XYZ` or `VIEW_PRODUCTS_INITIAL_PAYLOAD`.\n- `recommendation_request`: User is explicitly asking for product recommendations.\n- `add_to_cart_initial`: User wants to add an item to cart or explicitly starts a purchase. Triggered by `ADD_TO_CART_PRODUCT_ID_XYZ`.\n- `select_variant`: User is choosing a product variant. Triggered by `SELECT_VARIANT_PRODUCT_X_VARIANT_Y`.\n- `set_quantity`: User is specifying a quantity. Triggered by `SET_QUANTITY_PRODUCT_X_VARIANT_Y_QTY_Z` or `SET_QUANTITY_OTHER_PRODUCT_X_VARIANT_Y` followed by a number.\n- `view_cart`: User wants to see their current cart. Triggered by `VIEW_CART_PAYLOAD`.\n- `proceed_to_checkout`: User has confirmed cart and wants to start checkout. Triggered by `PROCEED_TO_CHECKOUT_PAYLOAD`.\n- `address_input`: User is providing their delivery address, or has clicked 'Edit Address' (`EDIT_ADDRESS_PAYLOAD`).\n- `confirm_address`: User confirms the provided address. Triggered by `CONFIRM_ADDRESS_PAYLOAD`.\n- `order_cancel`: User wants to cancel an order. Triggered by `CANCEL_ORDER_PAYLOAD_XYZ`.\n- `delivery_info`: User wants to track an order or ask about delivery. Triggered by `TRACK_ORDER_INITIAL_PAYLOAD` or `TRACK_ORDER_PAYLOAD_XYZ`.\n- `business_info`: User is asking general questions about the business, policies (returns, shipping, privacy), or contact info. Triggered by `BUSINESS_INFO_INITIAL_PAYLOAD`.\n- `general_greeting`: User is saying hello, thank you, or simple pleasantries.\n- `general_query`: For anything else not covered by the above intents or when intent is unclear.\n\nConsider the message and payload together to determine the most accurate intent."
          },
          {
            "role": "user",
            "content": "{{ $json.fullUserInput }}"
          }
        ],
        "options": {}
      },
      "type": "n8n-nodes-base.openAiChat",
      "typeVersion": 2,
      "position": [
        1440,
        340
      ],
      "id": "c1f7b0f0-c515-4659-af6d-54157d60533f",
      "name": "8. AI (Intent Recognition)"
    },
    {
      "parameters": {
        "value": "={{ $json.choices[0].message.content }}",
        "conditions": [
          {
            "value": "product_info",
            "id": "e30d7b27-3135-43ea-981f-702b85e05d21",
            "type": "string",
            "name": "Product Info"
          },
          {
            "value": "recommendation_request",
            "id": "b78b0f7e-751d-4054-935f-a392e21074e4",
            "type": "string",
            "name": "Recommendation Request"
          },
          {
            "value": "add_to_cart_initial",
            "id": "f5125d80-49c0-482a-89a0-62e08e612f02",
            "type": "string",
            "name": "Add to Cart (Initial)"
          },
          {
            "value": "select_variant",
            "id": "276b6d5f-f0a9-4b10-a299-d46a81e3532f",
            "type": "string",
            "name": "Select Variant"
          },
          {
            "value": "set_quantity",
            "id": "1b017b20-1e5b-488f-9a4f-563d76e3381e",
            "type": "string",
            "name": "Set Quantity"
          },
          {
            "value": "view_cart",
            "id": "0b15104a-8924-42b7-a3ac-53a992e59e51",
            "type": "string",
            "name": "View Cart"
          },
          {
            "value": "proceed_to_checkout",
            "id": "01851e39-16e5-4f36-829d-ee13f898b3c9",
            "type": "string",
            "name": "Proceed to Checkout"
          },
          {
            "value": "address_input",
            "id": "1b17b3b7-7e9b-46a0-9777-6f8d0a424a6e",
            "type": "string",
            "name": "Address Input"
          },
          {
            "value": "confirm_address",
            "id": "70d741c8-04fb-4b53-ae16-0951667d7194",
            "type": "string",
            "name": "Confirm Address"
          },
          {
            "value": "order_cancel",
            "id": "a9a304e2-63b7-4c7b-b0b2-4d76f0c69d8b",
            "type": "string",
            "name": "Order Cancel"
          },
          {
            "value": "delivery_info",
            "id": "06d4e84b-0c9f-4311-b0e5-a0a95f9c158d",
            "type": "string",
            "name": "Delivery Info"
          },
          {
            "value": "business_info",
            "id": "a244b7d1-e634-4b47-83d3-982d13735165",
            "type": "string",
            "name": "Business Info"
          },
          {
            "value": "general_greeting",
            "id": "a998b4c0-0255-46aa-ab94-f65582f3c7e7",
            "type": "string",
            "name": "General Greeting"
          }
        ],
        "default": {
          "value": "general_query",
          "id": "a2283020-f472-4d2c-80a2-f9a8ed2f317b",
          "name": "General Query"
        },
        "options": {}
      },
      "type": "n8n-nodes-base.switch",
      "typeVersion": 1.1,
      "position": [
        1680,
        340
      ],
      "id": "79b380a1-77b0-4541-86e4-41d3b37b420f",
      "name": "9. Switch (Route by Intent)"
    }
  ],
  "connections": {
    "1. Webhook (FB Messenger Trigger)": {
      "main": [
        [
          {
            "node": "2. If (Webhook Verification)",
            "type": "main",
            "index": 0
          },
          {
            "node": "4. Mark Seen",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "2. If (Webhook Verification)": {
      "main": [
        [
          {
            "node": "3. Respond to Webhook (Verification)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "4. Mark Seen": {
      "main": [
        [
          {
            "node": "5. Typing On",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "5. Typing On": {
      "main": [
        [
          {
            "node": "6. Set (Parse Messenger Event)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "6. Set (Parse Messenger Event)": {
      "main": [
        [
          {
            "node": "7. Code (Prepare AI Input)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "7. Code (Prepare AI Input)": {
      "main": [
        [
          {
            "node": "8. AI (Intent Recognition)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "8. AI (Intent Recognition)": {
      "main": [
        [
          {
            "node": "9. Switch (Route by Intent)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  },
  "active": false,
  "settings": {
    "executionOrder": "v1"
  },
  "versionId": "4696ced4-9ba0-43bc-929b-2924dbb5f6c0",
  "meta": {
    "instanceId": "0722834b00fc3f402b42c001ea7434db302102e53f78333bd7b87cde2277d5a4"
  },
  "id": "XPFGa2fqRGJ5Oa9M"
}
```

**Explanation of Core Workflow:**

1.  **1. Webhook (FB Messenger Trigger):** Your provided webhook node. Configured to listen on a path (e.g., `webhook_fb_messenger`).
2.  **2. If (Webhook Verification):** Your provided `If` node, crucial for Facebook to verify your webhook. `hub.verify_token` must match what you set in your Facebook App.
3.  **3. Respond to Webhook (Verification):** Your provided `Respond to Webhook` node. Sends the `hub.challenge` back to Facebook.
4.  **4. Mark Seen:** Your `mark_seen` node. Sends a "message seen" indicator to the user.
5.  **5. Typing On:** Your `typing_on` node. Shows a "typing..." indicator to the user.
6.  **6. Set (Parse Messenger Event):** Extracts `PSID`, `MessageText`, `Payload`, `Referral`, and also attempts to capture `UserEmail` or `UserPhoneNumber` if the user sent it via quick reply.
7.  **7. Code (Prepare AI Input):** Combines `MessageText` and `Payload` into `fullUserInput` for the AI to get the full context.
8.  **8. AI (Intent Recognition):** This is the core AI node.
    *   **Model:** `gpt-4o` (or your chosen LLM).
    *   **System Prompt:** Carefully engineered to define all possible intents and how to map user input/payloads to them.
    *   **User Prompt:** Takes the `fullUserInput` from the previous `Code` node.
    *   **Output:** The identified intent string (e.g., `product_info`, `add_to_cart_initial`).
9.  **9. Switch (Route by Intent):** This node takes the `intent` string from the AI node and branches the workflow to the appropriate logic for that intent. Each branch name corresponds to an intent.

---

### **2. Intent-Specific Branches (Connected to the Switch Node)**

Now, we'll build out each branch from the "9. Switch (Route by Intent)" node. Each branch will end with a "10. Typing Off" node and a "11. Send Messenger Response" node to complete the interaction.

---

#### **Branch: `General Greeting`**

*   **Connect from:** `9. Switch (Route by Intent)` -> `General Greeting` branch
*   **Purpose:** Respond politely to greetings.

```json
{
  "nodes": [
    {
      "parameters": {
        "model": "gpt-4o",
        "authentication": "credentials",
        "credentials": {
          "credentialData": {
            "apiKey": "={{ $credentials.openAiApi.apiKey }}"
          }
        },
        "messages": [
          {
            "role": "system",
            "content": "You are a friendly f-commerce AI assistant. Respond politely to greetings. Keep it concise."
          },
          {
            "role": "user",
            "content": "{{ $json.MessageText }}"
          }
        ],
        "options": {}
      },
      "type": "n8n-nodes-base.openAiChat",
      "typeVersion": 2,
      "position": [
        1920,
        -100
      ],
      "id": "e6f4a8e2-f542-4f3b-b27e-8c6f1a8e9e1f",
      "name": "General Greeting AI Response"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"sender_action\": \"typing_off\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2160,
        -100
      ],
      "id": "efb92aa3-2a9b-4d79-98cb-2b80e79d0ef1_greeting",
      "name": "10. Typing Off (Greeting)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"message\": {\n    \"text\": \"{{ $json.choices[0].message.content }}\"\n  },\n  \"messaging_type\": \"RESPONSE\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2400,
        -100
      ],
      "id": "1d8b1e4c-f123-4d5e-9a0b-c2d3e0f1a2b3_greeting",
      "name": "11. Send Message (Greeting)"
    }
  ],
  "connections": {
    "9. Switch (Route by Intent)": {
      "main": [
        [
          {
            "node": "General Greeting AI Response",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "General Greeting AI Response": {
      "main": [
        [
          {
            "node": "10. Typing Off (Greeting)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "10. Typing Off (Greeting)": {
      "main": [
        [
          {
            "node": "11. Send Message (Greeting)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```

---

#### **Branch: `General Query`**

*   **Connect from:** `9. Switch (Route by Intent)` -> `General Query` branch
*   **Purpose:** Handle general questions or when intent is unclear.

```json
{
  "nodes": [
    {
      "parameters": {
        "model": "gpt-4o",
        "authentication": "credentials",
        "credentials": {
          "credentialData": {
            "apiKey": "={{ $credentials.openAiApi.apiKey }}"
          }
        },
        "messages": [
          {
            "role": "system",
            "content": "You are a helpful AI assistant for an f-commerce business. Respond to general queries. If the query isn't specific, politely suggest common tasks like 'browsing products', 'tracking an order', or 'viewing your cart'. Keep it concise and helpful."
          },
          {
            "role": "user",
            "content": "{{ $json.MessageText }}"
          }
        ],
        "options": {}
      },
      "type": "n8n-nodes-base.openAiChat",
      "typeVersion": 2,
      "position": [
        1920,
        100
      ],
      "id": "e6f4a8e2-f542-4f3b-b27e-8c6f1a8e9e1f_query",
      "name": "General Query AI Response"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"sender_action\": \"typing_off\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2160,
        100
      ],
      "id": "efb92aa3-2a9b-4d79-98cb-2b80e79d0ef1_query",
      "name": "10. Typing Off (Query)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"message\": {\n    \"text\": \"{{ $json.choices[0].message.content }}\"\n  },\n  \"messaging_type\": \"RESPONSE\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2400,
        100
      ],
      "id": "1d8b1e4c-f123-4d5e-9a0b-c2d3e0f1a2b3_query",
      "name": "11. Send Message (Query)"
    }
  ],
  "connections": {
    "9. Switch (Route by Intent)": {
      "main": [
        [
          {
            "node": "General Query AI Response",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "General Query AI Response": {
      "main": [
        [
          {
            "node": "10. Typing Off (Query)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "10. Typing Off (Query)": {
      "main": [
        [
          {
            "node": "11. Send Message (Query)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```

---

#### **Branch: `Business Info`**

*   **Connect from:** `9. Switch (Route by Intent)` -> `Business Info` branch
*   **Purpose:** Answer questions using a knowledge base.

```json
{
  "nodes": [
    {
      "parameters": {
        "model": "gpt-4o",
        "authentication": "credentials",
        "credentials": {
          "credentialData": {
            "apiKey": "={{ $credentials.openAiApi.apiKey }}"
          }
        },
        "messages": [
          {
            "role": "system",
            "content": "You are an AI agent for an f-commerce business. Your task is to answer user questions using the provided business knowledge base. Respond accurately and concisely. If the information is not available in the knowledge base, politely state that you cannot find the answer and suggest contacting customer support or browsing the website.\n\n**Knowledge Base:**\n- **Shipping Policy:** \"Standard shipping takes 3-5 business days. Express shipping is available for an extra fee and takes 1-2 business days. We ship internationally to select countries. Please see our full policy at https://yourstore.com/shipping-policy\"\n- **Return Policy:** \"Returns are accepted within 30 days of purchase, provided the item is unused and in its original packaging. Refunds are processed within 7 business days. See full details at https://yourstore.com/return-policy\"\n- **Contact Info:** \"You can reach customer support at support@yourstore.com or call us at +1-XXX-XXX-XXXX during business hours (Mon-Fri, 9 AM - 5 PM EST).\"\n- **About Us:** \"We are a premium online retailer specializing in handcrafted goods since 2020, dedicated to quality and customer satisfaction.\"\n- **Privacy Policy:** \"Your privacy is important to us. Read our full privacy policy here: https://docs.google.com/document/d/1oaTk8Tz1II9j3EbEb-0d6Qj1besGMHZV/edit?usp=sharing&ouid=115309775547107992968&rtpof=true&sd=trueL\"\n\nIf the user explicitly asks for the privacy policy, use the `Structured_Information_Template` (privacy_policy_template) for the response."
          },
          {
            "role": "user",
            "content": "{{ $json.MessageText }}"
          }
        ],
        "options": {}
      },
      "type": "n8n-nodes-base.openAiChat",
      "typeVersion": 2,
      "position": [
        1920,
        300
      ],
      "id": "e6f4a8e2-f542-4f3b-b27e-8c6f1a8e9e1f_business",
      "name": "Business Info AI Response"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"sender_action\": \"typing_off\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2160,
        300
      ],
      "id": "efb92aa3-2a9b-4d79-98cb-2b80e79d0ef1_business",
      "name": "10. Typing Off (Business)"
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 2
          },
          "conditions": [
            {
              "id": "134e7240-8c29-45e0-811d-617a022b7a9f",
              "leftValue": "={{ $json.MessageText.toLowerCase().includes('privacy policy') }}",
              "rightValue": "true",
              "operator": {
                "type": "string",
                "operation": "equals",
                "name": "filter.operator.equals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.2,
      "position": [
        2160,
        450
      ],
      "id": "be8b532a-7170-4d5c-9c71-c9f1e102d99d",
      "name": "If (Privacy Policy Request)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"message\": {\n    \"attachment\": {\n      \"type\": \"template\",\n      \"payload\": {\n        \"template_type\": \"customer_information\",\n        \"countries\": [\n          \"US\"\n        ],\n        \"business_privacy\": {\n          \"url\": \"https://docs.google.com/document/d/1oaTk8Tz1II9j3EbEb-0d6Qj1besGMHZV/edit?usp=sharing&ouid=115309775547107992968&rtpof=true&sd=trueL\"\n        },\n        \"expires_in_days\": 1\n      }\n    }\n  },\n  \"messaging_type\": \"RESPONSE\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2400,
        450
      ],
      "id": "15ceab7d-b616-491b-97d8-137e3f038c8e_business",
      "name": "11. Send (Structured Info Template - Privacy)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"message\": {\n    \"text\": \"{{ $json.choices[0].message.content }}\"\n  },\n  \"messaging_type\": \"RESPONSE\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2400,
        300
      ],
      "id": "1d8b1e4c-f123-4d5e-9a0b-c2d3e0f1a2b3_business",
      "name": "11. Send Message (Business Info)"
    }
  ],
  "connections": {
    "9. Switch (Route by Intent)": {
      "main": [
        [
          {
            "node": "Business Info AI Response",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Business Info AI Response": {
      "main": [
        [
          {
            "node": "10. Typing Off (Business)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "10. Typing Off (Business)": {
      "main": [
        [
          {
            "node": "If (Privacy Policy Request)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "If (Privacy Policy Request)": {
      "main": [
        [
          {
            "node": "11. Send (Structured Info Template - Privacy)",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "11. Send Message (Business Info)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```

---

#### **Branch: `Product Info` / `Recommendation Request`**

*   **Connect from:** `9. Switch (Route by Intent)` -> `Product Info` branch and `Recommendation Request` branch
*   **Purpose:** Search for products or provide recommendations, displaying them using a generic template.

```json
{
  "nodes": [
    {
      "parameters": {
        "model": "gpt-4o",
        "authentication": "credentials",
        "credentials": {
          "credentialData": {
            "apiKey": "={{ $credentials.openAiApi.apiKey }}"
          }
        },
        "messages": [
          {
            "role": "system",
            "content": "You are an AI assistant. Analyze the user's message (and any payload) to extract product keywords, categories, or attributes for searching an e-commerce catalog. If the user is asking for recommendations, set 'recommendation': true. \n\nOutput a JSON object. Example:\n{\"search_query\": \"red t-shirt\", \"category\": \"apparel\", \"color\": \"red\", \"recommendation\": false, \"product_id\": null}\n{\"search_query\": \"\", \"category\": \"shoes\", \"recommendation\": true, \"product_id\": null}\n{\"search_query\": \"\", \"category\": \"\", \"recommendation\": false, \"product_id\": \"some_product_id_from_payload\"}\n\nIf the payload contains `VIEW_DETAILS_PRODUCT_ID_XYZ`, extract XYZ as `product_id` and set `search_query` to null. Otherwise, if there is a general text search query, populate `search_query`."
          },
          {
            "role": "user",
            "content": "{{ $json.fullUserInput }}"
          }
        ],
        "options": {}
      },
      "type": "n8n-nodes-base.openAiChat",
      "typeVersion": 2,
      "position": [
        1920,
        550
      ],
      "id": "e6f4a8e2-f542-4f3b-b27e-8c6f1a8e9e1f_product_ai",
      "name": "Product Search AI"
    },
    {
      "parameters": {
        "method": "GET",
        "url": "https://yourstore.com/api/products",
        "queryParameters": {
          "parameters": [
            {
              "name": "q",
              "value": "={{ $json.choices[0].message.content.json.search_query }}"
            },
            {
              "name": "category",
              "value": "={{ $json.choices[0].message.content.json.category }}"
            },
            {
              "name": "color",
              "value": "={{ $json.choices[0].message.content.json.color }}"
            },
            {
              "name": "recommendation",
              "value": "={{ $json.choices[0].message.content.json.recommendation }}"
            },
            {
              "name": "id",
              "value": "={{ $json.choices[0].message.content.json.product_id }}"
            }
          ]
        },
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2160,
        550
      ],
      "id": "a6b7c8d9-e0f1-4a2b-8c3d-4e5f6a7b8c9d",
      "name": "12. Fetch Products from API"
    },
    {
      "parameters": {
        "functionCode": "const products = $json.data || [];\nconst messages = [];\n\nif (products && products.length > 0) {\n  const elements = [];\n  for (const product of products.slice(0, 10)) { // Limit to 10 for carousel\n    elements.push({\n      title: product.name,\n      image_url: product.image_url,\n      subtitle: `${product.currency || 'USD'} ${parseFloat(product.price).toFixed(2)} - ${product.short_description || ''}`,\n      default_action: {\n        type: \"web_url\",\n        url: product.product_url || `https://yourstore.com/products/${product.id}`,\n        webview_height_ratio: \"tall\"\n      },\n      buttons: [\n        {\n          type: \"postback\",\n          title: \"Add to Cart\",\n          payload: `ADD_TO_CART_PRODUCT_ID_${product.id}`\n        },\n        {\n          type: \"postback\",\n          title: \"Details\",\n          payload: `VIEW_DETAILS_PRODUCT_ID_${product.id}`\n        }\n      ]\n    });\n  }\n  \n  messages.push({\n    attachment: {\n      type: \"template\",\n      payload: {\n        template_type: \"generic\",\n        elements: elements\n      }\n    }\n  });\n\n} else {\n  messages.push({\n    text: \"Sorry, I couldn't find any products matching your request. Can I help you with something else?\"\n  });\n}\n\nreturn { json: { PSID: $json.PSID, messages: messages, is_product_details_view: $json.Payload.startsWith('VIEW_DETAILS_PRODUCT_ID_') } };"
      },
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [
        2400,
        550
      ],
      "id": "1b2035af-f19e-4589-ac37-e2656d63d8e5_format",
      "name": "13. Code (Format Product Carousel)"
    },
    {
      "parameters": {
        "node": "13. Code (Format Product Carousel)",
        "mode": "json",
        "value": "={{ $json.messages }}",
        "options": {}
      },
      "type": "n8n-nodes-base.loop",
      "typeVersion": 1,
      "position": [
        2640,
        550
      ],
      "id": "e98e154f-5c2a-4a6c-9d0a-b1c2d3e4f5a6",
      "name": "14. Loop Over Products"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"sender_action\": \"typing_off\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2880,
        600
      ],
      "id": "efb92aa3-2a9b-4d79-98cb-2b80e79d0ef1_product",
      "name": "10. Typing Off (Product)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"message\": {{ JSON.stringify($item.attachment || $item) }},\n  \"messaging_type\": \"RESPONSE\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        3120,
        600
      ],
      "id": "1d8b1e4c-f123-4d5e-9a0b-c2d3e0f1a2b3_product",
      "name": "11. Send Message (Product Card)"
    }
  ],
  "connections": {
    "9. Switch (Route by Intent)": {
      "main": [
        [
          {
            "node": "Product Search AI",
            "type": "main",
            "index": 0
          },
          {
            "node": "Product Search AI",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Product Search AI": {
      "main": [
        [
          {
            "node": "12. Fetch Products from API",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "12. Fetch Products from API": {
      "main": [
        [
          {
            "node": "13. Code (Format Product Carousel)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "13. Code (Format Product Carousel)": {
      "main": [
        [
          {
            "node": "14. Loop Over Products",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "14. Loop Over Products": {
      "main": [
        [
          {
            "node": "10. Typing Off (Product)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "10. Typing Off (Product)": {
      "main": [
        [
          {
            "node": "11. Send Message (Product Card)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```

---

#### **Branch: `Add to Cart (Initial)`**

*   **Connect from:** `9. Switch (Route by Intent)` -> `Add to Cart (Initial)` branch
*   **Purpose:** Handles initial "Add to Cart" clicks, checking for variants and prompting for quantity.

```json
{
  "nodes": [
    {
      "parameters": {
        "functionCode": "let productId = null;\nconst payload = $json.Payload;\n\n// Extract from payload (e.g., ADD_TO_CART_PRODUCT_ID_XYZ)\nif (payload.startsWith(\"ADD_TO_CART_PRODUCT_ID_\")) {\n  productId = payload.split(\"ADD_TO_CART_PRODUCT_ID_\")[1];\n}\n\nreturn { json: { PSID: $json.PSID, productId: productId } };"
      },
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [
        1920,
        750
      ],
      "id": "f5125d80-49c0-482a-89a0-62e08e612f02_extract",
      "name": "12. Code (Extract Product ID)"
    },
    {
      "parameters": {
        "method": "GET",
        "url": "https://yourstore.com/api/products/{{ $json.productId }}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2160,
        750
      ],
      "id": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f",
      "name": "13. Fetch Product Details"
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 2
          },
          "conditions": [
            {
              "id": "e6a1b2c3-d4e5-4f6g-7h8i-9j0k1l2m3n4o",
              "leftValue": "={{ $json.data.variants && $json.data.variants.length > 0 }}",
              "rightValue": "true",
              "operator": {
                "type": "boolean",
                "operation": "equals",
                "name": "filter.operator.equals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.2,
      "position": [
        2400,
        750
      ],
      "id": "a1b2c3d4-e5f6-4g7h-8i9j-0k1l2m3n4o5p",
      "name": "14. If (Product Has Variants)"
    },
    {
      "parameters": {
        "functionCode": "const product = $json.data;\nconst quickReplies = product.variants.map(variant => ({\n  content_type: \"text\",\n  title: variant.name,\n  payload: `SELECT_VARIANT_${product.id}_${variant.id}`\n}));\n\nreturn { json: { PSID: $json.PSID, text: `Please choose a variant for *${product.name}*:`, quick_replies: quickReplies, productId: product.id } };"
      },
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [
        2640,
        700
      ],
      "id": "64fe7b3c-1d49-447f-b5c8-7e6920d60f10_variants",
      "name": "15. Code (Format Variant Quick Replies)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"message\": {\n    \"text\": \"{{ $json.text }}\",\n    \"quick_replies\": {{ JSON.stringify($json.quick_replies) }}\n  },\n  \"messaging_type\": \"RESPONSE\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2880,
        700
      ],
      "id": "c9e0f1g2-h3i4-5j6k-7l8m-9n0o1p2q3r4s",
      "name": "16. Send (Variant Quick Replies)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"sender_action\": \"typing_off\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        3120,
        700
      ],
      "id": "efb92aa3-2a9b-4d79-98cb-2b80e79d0ef1_add_cart_variants",
      "name": "10. Typing Off (Add Cart w/ Variants)"
    },
    {
      "parameters": {
        "functionCode": "const product = $json.data;\nconst maxQuantity = Math.min(product.stock_level || 5, 5); // Max 5 for quick replies, fallback to 5\n\nconst quickReplies = [];\nfor (let i = 1; i <= maxQuantity; i++) {\n  quickReplies.push({\n    content_type: \"text\",\n    title: i.toString(),\n    payload: `SET_QUANTITY_${product.id}_NO_VARIANT_${i}` // Use NO_VARIANT placeholder\n  });\n}\nquickReplies.push({\n    content_type: \"text\",\n    title: \"Other (Type)\",\n    payload: `SET_QUANTITY_OTHER_${product.id}_NO_VARIANT` // Special payload for user input\n});\n\nreturn { json: { PSID: $json.PSID, text: `How many *${product.name}* would you like?`, quick_replies: quickReplies, productId: product.id, variantId: 'NO_VARIANT' } };"
      },
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [
        2640,
        800
      ],
      "id": "b1c2d3e4-f5a6-7b8c-9d0e-1f2a3b4c5d6e",
      "name": "15. Code (Format Quantity Quick Replies - No Variant)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"message\": {\n    \"text\": \"{{ $json.text }}\",\n    \"quick_replies\": {{ JSON.stringify($json.quick_replies) }}\n  },\n  \"messaging_type\": \"RESPONSE\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2880,
        800
      ],
      "id": "d0e1f2a3-b4c5-6d7e-8f9a-0b1c2d3e4f5a",
      "name": "16. Send (Quantity Quick Replies - No Variant)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"sender_action\": \"typing_off\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        3120,
        800
      ],
      "id": "efb92aa3-2a9b-4d79-98cb-2b80e79d0ef1_add_cart_no_variants",
      "name": "10. Typing Off (Add Cart No Variants)"
    }
  ],
  "connections": {
    "9. Switch (Route by Intent)": {
      "main": [
        [
          {
            "node": "12. Code (Extract Product ID)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "12. Code (Extract Product ID)": {
      "main": [
        [
          {
            "node": "13. Fetch Product Details",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "13. Fetch Product Details": {
      "main": [
        [
          {
            "node": "14. If (Product Has Variants)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "14. If (Product Has Variants)": {
      "main": [
        [
          {
            "node": "15. Code (Format Variant Quick Replies)",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "15. Code (Format Quantity Quick Replies - No Variant)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "15. Code (Format Variant Quick Replies)": {
      "main": [
        [
          {
            "node": "16. Send (Variant Quick Replies)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "16. Send (Variant Quick Replies)": {
      "main": [
        [
          {
            "node": "10. Typing Off (Add Cart w/ Variants)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "15. Code (Format Quantity Quick Replies - No Variant)": {
      "main": [
        [
          {
            "node": "16. Send (Quantity Quick Replies - No Variant)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "16. Send (Quantity Quick Replies - No Variant)": {
      "main": [
        [
          {
            "node": "10. Typing Off (Add Cart No Variants)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```

---

#### **Branch: `Select Variant`**

*   **Connect from:** `9. Switch (Route by Intent)` -> `Select Variant` branch
*   **Purpose:** User has chosen a variant, now prompt for quantity.

```json
{
  "nodes": [
    {
      "parameters": {
        "functionCode": "const payload = $json.Payload; // e.g., SELECT_VARIANT_PRODUCT_X_VARIANT_Y\nconst parts = payload.split('_');\nconst productId = parts[2];\nconst variantId = parts[4];\nreturn { json: { PSID: $json.PSID, productId: productId, variantId: variantId } };"
      },
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [
        1920,
        1000
      ],
      "id": "f5125d80-49c0-482a-89a0-62e08e612f02_extract_variant",
      "name": "12. Code (Extract Product/Variant ID)"
    },
    {
      "parameters": {
        "method": "GET",
        "url": "https://yourstore.com/api/products/{{ $json.productId }}/variants/{{ $json.variantId }}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2160,
        1000
      ],
      "id": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f_fetch_variant",
      "name": "13. Fetch Variant Details"
    },
    {
      "parameters": {
        "functionCode": "const variant = $json.data;\nconst productId = $json.productId; // from previous node\nconst maxQuantity = Math.min(variant.stock_level || 5, 5); // Max 5 for quick replies, fallback to 5\n\nconst quickReplies = [];\nfor (let i = 1; i <= maxQuantity; i++) {\n  quickReplies.push({\n    content_type: \"text\",\n    title: i.toString(),\n    payload: `SET_QUANTITY_${productId}_${variant.id}_${i}`\n  });\n}\nquickReplies.push({\n    content_type: \"text\",\n    title: \"Other (Type)\",\n    payload: `SET_QUANTITY_OTHER_${productId}_${variant.id}` // Special payload for user input\n});\n\nreturn { json: { PSID: $json.PSID, text: `How many *${variant.name}* would you like?`, quick_replies: quickReplies, productId: productId, variantId: variant.id } };"
      },
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [
        2400,
        1000
      ],
      "id": "b1c2d3e4-f5a6-7b8c-9d0e-1f2a3b4c5d6e_variant_quantity",
      "name": "14. Code (Format Quantity Quick Replies)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"message\": {\n    \"text\": \"{{ $json.text }}\",\n    \"quick_replies\": {{ JSON.stringify($json.quick_replies) }}\n  },\n  \"messaging_type\": \"RESPONSE\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2640,
        1000
      ],
      "id": "d0e1f2a3-b4c5-6d7e-8f9a-0b1c2d3e4f5a_variant_quantity",
      "name": "15. Send (Quantity Quick Replies)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"sender_action\": \"typing_off\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2880,
        1000
      ],
      "id": "efb92aa3-2a9b-4d79-98cb-2b80e79d0ef1_variant_quantity",
      "name": "10. Typing Off (Variant Quantity)"
    }
  ],
  "connections": {
    "9. Switch (Route by Intent)": {
      "main": [
        [
          {
            "node": "12. Code (Extract Product/Variant ID)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "12. Code (Extract Product/Variant ID)": {
      "main": [
        [
          {
            "node": "13. Fetch Variant Details",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "13. Fetch Variant Details": {
      "main": [
        [
          {
            "node": "14. Code (Format Quantity Quick Replies)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "14. Code (Format Quantity Quick Replies)": {
      "main": [
        [
          {
            "node": "15. Send (Quantity Quick Replies)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "15. Send (Quantity Quick Replies)": {
      "main": [
        [
          {
            "node": "10. Typing Off (Variant Quantity)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```

---

#### **Branch: `Set Quantity`**

*   **Connect from:** `9. Switch (Route by Intent)` -> `Set Quantity` branch
*   **Purpose:** User has provided a quantity; add the item to the user's cart. This node also needs a mechanism to retrieve `productId` and `variantId` if the user typed "Other (Type)" and then a number. This requires user state management.

```json
{
  "nodes": [
    {
      "parameters": {
        "functionCode": "let productId = null;\nlet variantId = null;\nlet quantity = null;\nconst payload = $json.Payload;\nconst messageText = $json.MessageText;\n\nif (payload.startsWith(\"SET_QUANTITY_\")) {\n  const parts = payload.split('_');\n  if (parts[2] === 'OTHER') { \n    // This path is for when user clicked 'Other (Type)' and then typed a number.\n    // We need to retrieve product/variant from user's state. (Placeholder for DB call)\n    // For this example, let's assume `temp_product_info` exists from a previous state DB lookup\n    // productId = (await n8n.getNode('12. Fetch User State (Set Quantity)').getItem(0).json.temp_product_id;\n    // variantId = (await n8n.getNode('12. Fetch User State (Set Quantity)').getItem(0).json.temp_variant_id;\n    productId = parts[3]; // Placeholder, should come from state\n    variantId = parts[4]; // Placeholder, should come from state\n    quantity = parseInt(messageText);\n\n  } else { // Quantity from quick reply\n    productId = parts[2];\n    variantId = parts[3];\n    quantity = parseInt(parts[4]);\n  }\n} else { // AI detected quantity from raw message, needs last product context\n    // This is a complex scenario needing robust user state. \n    // For now, assume most quantity inputs are via quick replies.\n    // If AI route gets here without explicit payload, it's likely an error or needs context from user state.\n    quantity = parseInt(messageText);\n    // Add logic here to fetch last product/variant ID from user's state if `quantity` is valid and IDs are missing\n    if(isNaN(quantity) || quantity <= 0) {\n        return { json: { PSID: $json.PSID, error: \"Invalid quantity provided. Please enter a valid number.\" } };\n    }\n    // Placeholder: Retrieve `productId` and `variantId` from user session/state\n    // For demo purposes, we'll try to get it from a temporary global var if not set.\n    // In real scenario, use a DB lookup for `PSID` to get last context.\n    productId = productId || 'YOUR_FALLBACK_PRODUCT_ID'; // Replace with real lookup\n    variantId = variantId || 'NO_VARIANT'; // Replace with real lookup\n}\n\nif (isNaN(quantity) || quantity <= 0) {\n  return { json: { PSID: $json.PSID, error: \"Invalid quantity provided. Please enter a valid number.\" } };\n}\n\nreturn { json: { PSID: $json.PSID, productId: productId, variantId: variantId, quantity: quantity, error: null } };"
      },
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [
        1920,
        1250
      ],
      "id": "f5125d80-49c0-482a-89a0-62e08e612f02_extract_quantity",
      "name": "12. Code (Extract Quantity)"
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 2
          },
          "conditions": [
            {
              "id": "e6a1b2c3-d4e5-4f6g-7h8i-9j0k1l2m3n4o",
              "leftValue": "={{ $json.error }}",
              "rightValue": "null",
              "operator": {
                "type": "string",
                "operation": "equals",
                "name": "filter.operator.equals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.2,
      "position": [
        2160,
        1250
      ],
      "id": "a1b2c3d4-e5f6-4g7h-8i9j-0k1l2m3n4o5p_quantity_valid",
      "name": "13. If (Quantity Valid)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "https://yourstore.com/api/cart/add",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"user_id\": \"{{ $json.PSID }}\",\n  \"product_id\": \"{{ $json.productId }}\",\n  \"variant_id\": \"{{ $json.variantId === 'NO_VARIANT' ? null : $json.variantId }}\",\n  \"quantity\": {{ $json.quantity }}\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2400,
        1200
      ],
      "id": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f_add_cart",
      "name": "14. Add to Cart API"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"message\": {\n    \"text\": \"Item added to your cart! What next?\",\n    \"quick_replies\": [\n      { \"content_type\": \"text\", \"title\": \"View Cart\", \"payload\": \"VIEW_CART_PAYLOAD\" },\n      { \"content_type\": \"text\", \"title\": \"Continue Shopping\", \"payload\": \"VIEW_PRODUCTS_INITIAL_PAYLOAD\" }\n    ]\n  },\n  \"messaging_type\": \"RESPONSE\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2640,
        1200
      ],
      "id": "d0e1f2a3-b4c5-6d7e-8f9a-0b1c2d3e4f5a_cart_confirmation",
      "name": "15. Send Cart Confirmation"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"sender_action\": \"typing_off\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2880,
        1200
      ],
      "id": "efb92aa3-2a9b-4d79-98cb-2b80e79d0ef1_cart_confirmation",
      "name": "10. Typing Off (Cart Confirmation)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"message\": {\n    \"text\": \"{{ $json.error }}\"\n  },\n  \"messaging_type\": \"RESPONSE\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2400,
        1300
      ],
      "id": "d0e1f2a3-b4c5-6d7e-8f9a-0b1c2d3e4f5a_quantity_error",
      "name": "14. Send (Quantity Error Message)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"sender_action\": \"typing_off\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2640,
        1300
      ],
      "id": "efb92aa3-2a9b-4d79-98cb-2b80e79d0ef1_quantity_error",
      "name": "10. Typing Off (Quantity Error)"
    }
  ],
  "connections": {
    "9. Switch (Route by Intent)": {
      "main": [
        [
          {
            "node": "12. Code (Extract Quantity)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "12. Code (Extract Quantity)": {
      "main": [
        [
          {
            "node": "13. If (Quantity Valid)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "13. If (Quantity Valid)": {
      "main": [
        [
          {
            "node": "14. Add to Cart API",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "14. Send (Quantity Error Message)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "14. Add to Cart API": {
      "main": [
        [
          {
            "node": "15. Send Cart Confirmation",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "15. Send Cart Confirmation": {
      "main": [
        [
          {
            "node": "10. Typing Off (Cart Confirmation)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "14. Send (Quantity Error Message)": {
      "main": [
        [
          {
            "node": "10. Typing Off (Quantity Error)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```

---

#### **Branch: `View Cart`**

*   **Connect from:** `9. Switch (Route by Intent)` -> `View Cart` branch
*   **Purpose:** Display current cart items and offer checkout.

```json
{
  "nodes": [
    {
      "parameters": {
        "method": "GET",
        "url": "https://yourstore.com/api/cart?user_id={{ $json.PSID }}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        1920,
        1450
      ],
      "id": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f_fetch_cart",
      "name": "12. Fetch Cart Items"
    },
    {
      "parameters": {
        "functionCode": "const cartItems = $json.data || [];\nlet cartSummary = \"Your Cart:\\n\";\nlet total = 0;\n\nif (cartItems && cartItems.length > 0) {\n  for (const item of cartItems) {\n    cartSummary += `- ${item.product_name} (${item.variant_name || 'N/A'}) x ${item.quantity} @ ${item.currency || 'USD'} ${parseFloat(item.price).toFixed(2)} = ${item.currency || 'USD'} ${(item.quantity * parseFloat(item.price)).toFixed(2)}\\n`;\n    total += item.quantity * parseFloat(item.price);\n  }\n  cartSummary += `\\n*Total: ${cartItems[0].currency || 'USD'} ${total.toFixed(2)}*\\n\\n`; \n  cartSummary += \"Ready to checkout?\";\n\n  return { json: { PSID: $json.PSID, cart_text: cartSummary, has_items: true } };\n\n} else {\n  return { json: { PSID: $json.PSID, cart_text: \"Your cart is empty. Start shopping now!\", has_items: false } };\n}"
      },
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [
        2160,
        1450
      ],
      "id": "b1c2d3e4-f5a6-7b8c-9d0e-1f2a3b4c5d6e_format_cart",
      "name": "13. Code (Format Cart Display)"
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 2
          },
          "conditions": [
            {
              "id": "e6a1b2c3-d4e5-4f6g-7h8i-9j0k1l2m3n4o",
              "leftValue": "={{ $json.has_items }}",
              "rightValue": "true",
              "operator": {
                "type": "boolean",
                "operation": "equals",
                "name": "filter.operator.equals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.2,
      "position": [
        2400,
        1450
      ],
      "id": "a1b2c3d4-e5f6-4g7h-8i9j-0k1l2m3n4o5p_has_items",
      "name": "14. If (Has Items in Cart)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"message\": {\n    \"attachment\": {\n      \"type\": \"template\",\n      \"payload\": {\n        \"template_type\": \"button\",\n        \"text\": \"{{ $json.cart_text }}\",\n        \"buttons\": [\n          {\n            \"type\": \"postback\",\n            \"title\": \"Proceed to Checkout\",\n            \"payload\": \"PROCEED_TO_CHECKOUT_PAYLOAD\"\n          }\n        ]\n      }\n    }\n  },\n  \"messaging_type\": \"RESPONSE\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2640,
        1400
      ],
      "id": "c9e0f1g2-h3i4-5j6k-7l8m-9n0o1p2q3r4s_cart_summary",
      "name": "15. Send (Cart Summary + Checkout)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"sender_action\": \"typing_off\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2880,
        1400
      ],
      "id": "efb92aa3-2a9b-4d79-98cb-2b80e79d0ef1_cart_summary",
      "name": "10. Typing Off (Cart Summary)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"message\": {\n    \"text\": \"{{ $json.cart_text }}\"\n  },\n  \"messaging_type\": \"RESPONSE\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2640,
        1500
      ],
      "id": "c9e0f1g2-h3i4-5j6k-7l8m-9n0o1p2q3r4s_empty_cart",
      "name": "15. Send (Empty Cart Message)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"sender_action\": \"typing_off\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2880,
        1500
      ],
      "id": "efb92aa3-2a9b-4d79-98cb-2b80e79d0ef1_empty_cart",
      "name": "10. Typing Off (Empty Cart)"
    }
  ],
  "connections": {
    "9. Switch (Route by Intent)": {
      "main": [
        [
          {
            "node": "12. Fetch Cart Items",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "12. Fetch Cart Items": {
      "main": [
        [
          {
            "node": "13. Code (Format Cart Display)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "13. Code (Format Cart Display)": {
      "main": [
        [
          {
            "node": "14. If (Has Items in Cart)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "14. If (Has Items in Cart)": {
      "main": [
        [
          {
            "node": "15. Send (Cart Summary + Checkout)",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "15. Send (Empty Cart Message)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "15. Send (Cart Summary + Checkout)": {
      "main": [
        [
          {
            "node": "10. Typing Off (Cart Summary)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "15. Send (Empty Cart Message)": {
      "main": [
        [
          {
            "node": "10. Typing Off (Empty Cart)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```

---

#### **Branch: `Proceed to Checkout`**

*   **Connect from:** `9. Switch (Route by Intent)` -> `Proceed to Checkout` branch
*   **Purpose:** Prompt user for delivery address and update user state.

```json
{
  "nodes": [
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"message\": {\n    \"text\": \"Great! Please provide your full delivery address (Street, Apt/Suite, City, Postal Code, Country) so we can calculate shipping.\",\n    \"quick_replies\": [\n      {\n        \"content_type\": \"user_email\"\n      },\n      {\n        \"content_type\": \"user_phone_number\"\n      }\n    ]\n  },\n  \"messaging_type\": \"RESPONSE\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        1920,
        1650
      ],
      "id": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f_request_address",
      "name": "12. Send (Request Address + Quick Replies)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "https://yourstore.com/api/user_state/{{ $json.PSID }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"state\": \"awaiting_address_input\",\n  \"temp_data\": {}\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2160,
        1650
      ],
      "id": "a1b2c3d4-e5f6-4g7h-8i9j-0k1l2m3n4o5p_update_state_address",
      "name": "13. Update User State (Awaiting Address)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"sender_action\": \"typing_off\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2400,
        1650
      ],
      "id": "efb92aa3-2a9b-4d79-98cb-2b80e79d0ef1_request_address",
      "name": "10. Typing Off (Request Address)"
    }
  ],
  "connections": {
    "9. Switch (Route by Intent)": {
      "main": [
        [
          {
            "node": "12. Send (Request Address + Quick Replies)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "12. Send (Request Address + Quick Replies)": {
      "main": [
        [
          {
            "node": "13. Update User State (Awaiting Address)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "13. Update User State (Awaiting Address)": {
      "main": [
        [
          {
            "node": "10. Typing Off (Request Address)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```

---

#### **Branch: `Address Input`**

*   **Connect from:** `9. Switch (Route by Intent)` -> `Address Input` branch
*   **Purpose:** Parse the provided address, ask for confirmation, and update user state.
*   **Crucial:** This branch *must* first check if the user is in the `awaiting_address_input` state.

```json
{
  "nodes": [
    {
      "parameters": {
        "method": "GET",
        "url": "https://yourstore.com/api/user_state/{{ $json.PSID }}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        1920,
        1850
      ],
      "id": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f_fetch_user_state_address",
      "name": "12. Fetch User State (Address Input)"
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 2
          },
          "conditions": [
            {
              "id": "e6a1b2c3-d4e5-4f6g-7h8i-9j0k1l2m3n4o",
              "leftValue": "={{ $json.data.state }}",
              "rightValue": "awaiting_address_input",
              "operator": {
                "type": "string",
                "operation": "equals",
                "name": "filter.operator.equals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.2,
      "position": [
        2160,
        1850
      ],
      "id": "a1b2c3d4-e5f6-4g7h-8i9j-0k1l2m3n4o5p_if_awaiting_address",
      "name": "13. If (Awaiting Address Input)"
    },
    {
      "parameters": {
        "model": "gpt-4o",
        "authentication": "credentials",
        "credentials": {
          "credentialData": {
            "apiKey": "={{ $credentials.openAiApi.apiKey }}"
          }
        },
        "messages": [
          {
            "role": "system",
            "content": "You are an AI address parser. Extract the street, city, postal code, and country from the provided text. Handle multi-line inputs if needed. Output a JSON object. Example:\n{\"street\": \"123 Main St, Apt 4B\", \"city\": \"Anytown\", \"postal_code\": \"12345\", \"country\": \"USA\"}\nIf any part is missing or unclear, set it to null. Prioritize extracting all components."
          },
          {
            "role": "user",
            "content": "{{ $json.MessageText }}"
          }
        ],
        "options": {}
      },
      "type": "n8n-nodes-base.openAiChat",
      "typeVersion": 2,
      "position": [
        2400,
        1800
      ],
      "id": "e6f4a8e2-f542-4f3b-b27e-8c6f1a8e9e1f_address_parse",
      "name": "14. AI (Address Extraction)"
    },
    {
      "parameters": {
        "functionCode": "const address = JSON.parse($json.choices[0].message.content);\nconst confirmationText = `Is this your correct delivery address?\\n\\n` +\n                                     `Street: ${address.street || 'N/A'}\\n` +\n                                     `City: ${address.city || 'N/A'}\\n` +\n                                     `Postal Code: ${address.postal_code || 'N/A'}\\n` +\n                                     `Country: ${address.country || 'N/A'}\\n\\n` +\n                                     `Please 'Confirm' or 'Edit'.`;\nreturn { json: { PSID: $json.PSID, confirmationText: confirmationText, parsed_address: address } };"
      },
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [
        2640,
        1800
      ],
      "id": "b1c2d3e4-f5a6-7b8c-9d0e-1f2a3b4c5d6e_format_address",
      "name": "15. Code (Format Address Confirmation)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"message\": {\n    \"attachment\": {\n      \"type\": \"template\",\n      \"payload\": {\n        \"template_type\": \"button\",\n        \"text\": \"{{ $json.confirmationText }}\",\n        \"buttons\": [\n          {\n            \"type\": \"postback\",\n            \"title\": \"Confirm Address\",\n            \"payload\": \"CONFIRM_ADDRESS_PAYLOAD\"\n          },\n          {\n            \"type\": \"postback\",\n            \"title\": \"Edit Address\",\n            \"payload\": \"EDIT_ADDRESS_PAYLOAD\"\n          }\n        ]\n      }\n    }\n  },\n  \"messaging_type\": \"RESPONSE\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2880,
        1800
      ],
      "id": "c9e0f1g2-h3i4-5j6k-7l8m-9n0o1p2q3r4s_send_address_confirm",
      "name": "16. Send (Address Confirmation + Buttons)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "https://yourstore.com/api/user_state/{{ $json.PSID }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"state\": \"awaiting_address_confirmation\",\n  \"temp_data\": { \"address\": {{ JSON.stringify($json.parsed_address) }} }\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        3120,
        1800
      ],
      "id": "a1b2c3d4-e5f6-4g7h-8i9j-0k1l2m3n4o5p_update_state_confirm",
      "name": "17. Update User State (Awaiting Confirmation)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"sender_action\": \"typing_off\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        3360,
        1800
      ],
      "id": "efb92aa3-2a9b-4d79-98cb-2b80e79d0ef1_address_confirm",
      "name": "10. Typing Off (Address Confirm)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"message\": {\n    \"text\": \"Please provide your full delivery address. I didn't recognize your last message as an address, or you weren't in the correct step to provide one.\"\n  },\n  \"messaging_type\": \"RESPONSE\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2400,
        1900
      ],
      "id": "c9e0f1g2-h3i4-5j6k-7l8m-9n0o1p2q3r4s_address_reprompt",
      "name": "14. Send (Address Reprompt)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"sender_action\": \"typing_off\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2640,
        1900
      ],
      "id": "efb92aa3-2a9b-4d79-98cb-2b80e79d0ef1_address_reprompt",
      "name": "10. Typing Off (Address Reprompt)"
    }
  ],
  "connections": {
    "9. Switch (Route by Intent)": {
      "main": [
        [
          {
            "node": "12. Fetch User State (Address Input)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "12. Fetch User State (Address Input)": {
      "main": [
        [
          {
            "node": "13. If (Awaiting Address Input)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "13. If (Awaiting Address Input)": {
      "main": [
        [
          {
            "node": "14. AI (Address Extraction)",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "14. Send (Address Reprompt)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "14. AI (Address Extraction)": {
      "main": [
        [
          {
            "node": "15. Code (Format Address Confirmation)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "15. Code (Format Address Confirmation)": {
      "main": [
        [
          {
            "node": "16. Send (Address Confirmation + Buttons)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "16. Send (Address Confirmation + Buttons)": {
      "main": [
        [
          {
            "node": "17. Update User State (Awaiting Confirmation)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "17. Update User State (Awaiting Confirmation)": {
      "main": [
        [
          {
            "node": "10. Typing Off (Address Confirm)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "14. Send (Address Reprompt)": {
      "main": [
        [
          {
            "node": "10. Typing Off (Address Reprompt)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```

---

#### **Branch: `Confirm Address`**

*   **Connect from:** `9. Switch (Route by Intent)` -> `Confirm Address` branch
*   **Purpose:** User has confirmed address; create order and send receipt.

```json
{
  "nodes": [
    {
      "parameters": {
        "method": "GET",
        "url": "https://yourstore.com/api/user_state/{{ $json.PSID }}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        1920,
        2100
      ],
      "id": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f_fetch_user_state_confirm",
      "name": "12. Fetch User State (Confirm Address)"
    },
    {
      "parameters": {
        "method": "GET",
        "url": "https://yourstore.com/api/cart?user_id={{ $json.PSID }}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2160,
        2100
      ],
      "id": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f_fetch_cart_confirm",
      "name": "13. Fetch Cart Items (Confirm Address)"
    },
    {
      "parameters": {
        "functionCode": "const userState = $json.user_state_data;\nconst cartItems = $json.cart_items_data;\nconst deliveryAddress = userState.temp_data.address;\n\nif (!deliveryAddress || !cartItems || cartItems.length === 0) {\n    return { json: { PSID: $json.PSID, error: \"Checkout failed: Address or cart empty.\" } };\n}\n\nlet subtotal = 0;\nconst orderElements = cartItems.map(item => {\n    subtotal += item.quantity * parseFloat(item.price);\n    return {\n        title: item.product_name,\n        subtitle: item.variant_name || '',\n        quantity: item.quantity,\n        price: parseFloat(item.price),\n        currency: item.currency || 'USD',\n        image_url: item.image_url || 'https://example.com/placeholder.png'\n    };\n});\n\n// Placeholder for shipping/tax calculation from your API/service\nconst shipping_cost = 5.00; \nconst total_tax = subtotal * 0.05; // Example 5% tax\nconst total_cost = subtotal + shipping_cost + total_tax;\n\nreturn { json: { PSID: $json.PSID, order_details: {\n    user_id: $json.PSID,\n    cart_items: cartItems,\n    delivery_address: deliveryAddress,\n    subtotal: subtotal,\n    shipping_cost: shipping_cost,\n    total_tax: total_tax,\n    total_cost: total_cost,\n    order_elements: orderElements\n} } };"
      },
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [
        2400,
        2100
      ],
      "id": "b1c2d3e4-f5a6-7b8c-9d0e-1f2a3b4c5d6e_prepare_order",
      "name": "14. Code (Prepare Order Data)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "https://yourstore.com/api/orders",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"user_id\": \"{{ $json.PSID }}\",\n  \"items\": {{ JSON.stringify($json.order_details.cart_items) }},\n  \"delivery_address\": {{ JSON.stringify($json.order_details.delivery_address) }},\n  \"subtotal\": {{ $json.order_details.subtotal }},\n  \"shipping_cost\": {{ $json.order_details.shipping_cost }},\n  \"total_tax\": {{ $json.order_details.total_tax }},\n  \"total_cost\": {{ $json.order_details.total_cost }},\n  \"status\": \"pending\",\n  \"payment_method\": \"Messenger_Checkout\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2640,
        2100
      ],
      "id": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f_create_order",
      "name": "15. Create Order API"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "https://yourstore.com/api/cart/clear?user_id={{ $json.PSID }}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2880,
        2100
      ],
      "id": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f_clear_cart",
      "name": "16. Clear Cart API"
    },
    {
      "parameters": {
        "functionCode": "const orderConfirmation = $json.order_confirmation_data;\nconst orderDetails = $json.order_details;\nconst receiptTemplate = {\n  recipient_name: orderDetails.delivery_address.name || 'Customer',\n  order_number: orderConfirmation.order_id,\n  currency: orderDetails.cart_items[0].currency || 'USD',\n  payment_method: orderConfirmation.payment_method || 'Messenger Checkout',\n  order_url: orderConfirmation.order_url || `https://yourstore.com/orders/${orderConfirmation.order_id}`,\n  timestamp: Date.now().toString(),\n  address: {\n    street_1: orderDetails.delivery_address.street,\n    city: orderDetails.delivery_address.city,\n    postal_code: orderDetails.delivery_address.postal_code,\n    state: orderDetails.delivery_address.state || '', \n    country: orderDetails.delivery_address.country\n  },\n  summary: {\n    subtotal: parseFloat(orderDetails.subtotal).toFixed(2),\n    shipping_cost: parseFloat(orderDetails.shipping_cost).toFixed(2),\n    total_tax: parseFloat(orderDetails.total_tax).toFixed(2),\n    total_cost: parseFloat(orderDetails.total_cost).toFixed(2)\n  },\n  elements: orderDetails.order_elements\n};\n\nreturn { json: { PSID: $json.PSID, receipt_template: receiptTemplate } };"
      },
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [
        3120,
        2100
      ],
      "id": "b1c2d3e4-f5a6-7b8c-9d0e-1f2a3b4c5d6e_receipt_data",
      "name": "17. Code (Prepare Receipt Template Data)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"message\": {\n    \"attachment\": {\n      \"type\": \"template\",\n      \"payload\": {{ JSON.stringify($json.receipt_template) }}\n    }\n  },\n  \"messaging_type\": \"RESPONSE\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        3360,
        2100
      ],
      "id": "3bca2d33-cbe3-4dc0-b9d2-5c54e8930182_send_receipt",
      "name": "18. Send (Receipt Template)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "https://yourstore.com/api/user_state/{{ $json.PSID }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"state\": \"order_placed\",\n  \"temp_data\": {}\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        3600,
        2100
      ],
      "id": "a1b2c3d4-e5f6-4g7h-8i9j-0k1l2m3n4o5p_update_state_order_placed",
      "name": "19. Update User State (Order Placed)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"sender_action\": \"typing_off\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        3840,
        2100
      ],
      "id": "efb92aa3-2a9b-4d79-98cb-2b80e79d0ef1_order_success",
      "name": "10. Typing Off (Order Success)"
    }
  ],
  "connections": {
    "9. Switch (Route by Intent)": {
      "main": [
        [
          {
            "node": "12. Fetch User State (Confirm Address)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "12. Fetch User State (Confirm Address)": {
      "main": [
        [
          {
            "node": "13. Fetch Cart Items (Confirm Address)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "13. Fetch Cart Items (Confirm Address)": {
      "main": [
        [
          {
            "node": "14. Code (Prepare Order Data)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "14. Code (Prepare Order Data)": {
      "main": [
        [
          {
            "node": "15. Create Order API",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "15. Create Order API": {
      "main": [
        [
          {
            "node": "16. Clear Cart API",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "16. Clear Cart API": {
      "main": [
        [
          {
            "node": "17. Code (Prepare Receipt Template Data)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "17. Code (Prepare Receipt Template Data)": {
      "main": [
        [
          {
            "node": "18. Send (Receipt Template)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "18. Send (Receipt Template)": {
      "main": [
        [
          {
            "node": "19. Update User State (Order Placed)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "19. Update User State (Order Placed)": {
      "main": [
        [
          {
            "node": "10. Typing Off (Order Success)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```

---

#### **Branch: `Order Cancel`**

*   **Connect from:** `9. Switch (Route by Intent)` -> `Order Cancel` branch
*   **Purpose:** Allow users to cancel orders.

```json
{
  "nodes": [
    {
      "parameters": {
        "model": "gpt-4o",
        "authentication": "credentials",
        "credentials": {
          "credentialData": {
            "apiKey": "={{ $credentials.openAiApi.apiKey }}"
          }
        },
        "messages": [
          {
            "role": "system",
            "content": "You are an AI assistant. Extract the order number from the user's message. If found, output ONLY the number. If not found, output 'null'."
          },
          {
            "role": "user",
            "content": "{{ $json.MessageText }}"
          }
        ],
        "options": {}
      },
      "type": "n8n-nodes-base.openAiChat",
      "typeVersion": 2,
      "position": [
        1920,
        2350
      ],
      "id": "e6f4a8e2-f542-4f3b-b27e-8c6f1a8e9e1f_order_id_cancel",
      "name": "12. AI (Order ID Extraction)"
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 2
          },
          "conditions": [
            {
              "id": "e6a1b2c3-d4e5-4f6g-7h8i-9j0k1l2m3n4o",
              "leftValue": "={{ $json.choices[0].message.content }}",
              "rightValue": "null",
              "operator": {
                "type": "string",
                "operation": "notEqual",
                "name": "filter.operator.notEquals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.2,
      "position": [
        2160,
        2350
      ],
      "id": "a1b2c3d4-e5f6-4g7h-8i9j-0k1l2m3n4o5p_if_order_id_cancel",
      "name": "13. If (Order ID Found)"
    },
    {
      "parameters": {
        "method": "GET",
        "url": "https://yourstore.com/api/orders/{{ $json.choices[0].message.content }}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2400,
        2300
      ],
      "id": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f_fetch_order_status_cancel",
      "name": "14. Fetch Order Status"
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 2
          },
          "conditions": [
            {
              "id": "e6a1b2c3-d4e5-4f6g-7h8i-9j0k1l2m3n4o",
              "leftValue": "={{ $json.data.status }}",
              "rightValue": "pending",
              "operator": {
                "type": "string",
                "operation": "equals",
                "name": "filter.operator.equals"
              }
            },
            {
              "id": "e6a1b2c3-d4e5-4f6g-7h8i-9j0k1l2m3n4o_2",
              "leftValue": "={{ $json.data.status }}",
              "rightValue": "processing",
              "operator": {
                "type": "string",
                "operation": "equals",
                "name": "filter.operator.equals"
              }
            }
          ],
          "combinator": "or"
        },
        "options": {}
      },
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.2,
      "position": [
        2640,
        2300
      ],
      "id": "a1b2c3d4-e5f6-4g7h-8i9j-0k1l2m3n4o5p_if_cancellable",
      "name": "15. If (Order Cancellable)"
    },
    {
      "parameters": {
        "method": "PUT",
        "url": "https://yourstore.com/api/orders/{{ $json.choices[0].message.content }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"status\": \"cancelled\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2880,
        2250
      ],
      "id": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f_update_order_cancel",
      "name": "16. Update Order Status (Cancelled)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"message\": {\n    \"text\": \"Your order #`{{ $json.choices[0].message.content }}` has been successfully cancelled.\"\n  },\n  \"messaging_type\": \"RESPONSE\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        3120,
        2250
      ],
      "id": "c9e0f1g2-h3i4-5j6k-7l8m-9n0o1p2q3r4s_send_cancel_confirm",
      "name": "17. Send (Cancel Confirmation)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"sender_action\": \"typing_off\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        3360,
        2250
      ],
      "id": "efb92aa3-2a9b-4d79-98cb-2b80e79d0ef1_cancel_confirm",
      "name": "10. Typing Off (Cancel Confirm)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"message\": {\n    \"text\": \"I'm sorry, your order #`{{ $json.choices[0].message.content }}` cannot be cancelled as it's already `{{ $json.data.status }}`. Please contact support if you need further assistance.\"\n  },\n  \"messaging_type\": \"RESPONSE\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2880,
        2350
      ],
      "id": "c9e0f1g2-h3i4-5j6k-7l8m-9n0o1p2q3r4s_send_cancel_error",
      "name": "16. Send (Cancel Error Message)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"sender_action\": \"typing_off\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        3120,
        2350
      ],
      "id": "efb92aa3-2a9b-4d79-98cb-2b80e79d0ef1_cancel_error",
      "name": "10. Typing Off (Cancel Error)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"message\": {\n    \"text\": \"Please provide the order number you wish to cancel.\"\n  },\n  \"messaging_type\": \"RESPONSE\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2400,
        2450
      ],
      "id": "c9e0f1g2-h3i4-5j6k-7l8m-9n0o1p2q3r4s_request_order_id_cancel",
      "name": "14. Send (Request Order ID)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"sender_action\": \"typing_off\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2640,
        2450
      ],
      "id": "efb92aa3-2a9b-4d79-98cb-2b80e79d0ef1_request_order_id_cancel",
      "name": "10. Typing Off (Request Order ID)"
    }
  ],
  "connections": {
    "9. Switch (Route by Intent)": {
      "main": [
        [
          {
            "node": "12. AI (Order ID Extraction)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "12. AI (Order ID Extraction)": {
      "main": [
        [
          {
            "node": "13. If (Order ID Found)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "13. If (Order ID Found)": {
      "main": [
        [
          {
            "node": "14. Fetch Order Status",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "14. Send (Request Order ID)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "14. Fetch Order Status": {
      "main": [
        [
          {
            "node": "15. If (Order Cancellable)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "15. If (Order Cancellable)": {
      "main": [
        [
          {
            "node": "16. Update Order Status (Cancelled)",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "16. Send (Cancel Error Message)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "16. Update Order Status (Cancelled)": {
      "main": [
        [
          {
            "node": "17. Send (Cancel Confirmation)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "17. Send (Cancel Confirmation)": {
      "main": [
        [
          {
            "node": "10. Typing Off (Cancel Confirm)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "16. Send (Cancel Error Message)": {
      "main": [
        [
          {
            "node": "10. Typing Off (Cancel Error)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "14. Send (Request Order ID)": {
      "main": [
        [
          {
            "node": "10. Typing Off (Request Order ID)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```

---

#### **Branch: `Delivery Info`**

*   **Connect from:** `9. Switch (Route by Intent)` -> `Delivery Info` branch
*   **Purpose:** Provide real-time order tracking.

```json
{
  "nodes": [
    {
      "parameters": {
        "model": "gpt-4o",
        "authentication": "credentials",
        "credentials": {
          "credentialData": {
            "apiKey": "={{ $credentials.openAiApi.apiKey }}"
          }
        },
        "messages": [
          {
            "role": "system",
            "content": "You are an AI assistant. Extract the order number from the user's message. If found, output ONLY the number. If not found, output 'null'."
          },
          {
            "role": "user",
            "content": "{{ $json.MessageText }}"
          }
        ],
        "options": {}
      },
      "type": "n8n-nodes-base.openAiChat",
      "typeVersion": 2,
      "position": [
        1920,
        2600
      ],
      "id": "e6f4a8e2-f542-4f3b-b27e-8c6f1a8e9e1f_order_id_track",
      "name": "12. AI (Order ID Extraction)"
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 2
          },
          "conditions": [
            {
              "id": "e6a1b2c3-d4e5-4f6g-7h8i-9j0k1l2m3n4o",
              "leftValue": "={{ $json.choices[0].message.content }}",
              "rightValue": "null",
              "operator": {
                "type": "string",
                "operation": "notEqual",
                "name": "filter.operator.notEquals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.2,
      "position": [
        2160,
        2600
      ],
      "id": "a1b2c3d4-e5f6-4g7h-8i9j-0k1l2m3n4o5p_if_order_id_track",
      "name": "13. If (Order ID Found)"
    },
    {
      "parameters": {
        "method": "GET",
        "url": "https://yourstore.com/api/tracking?order_id={{ $json.choices[0].message.content }}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2400,
        2550
      ],
      "id": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f_fetch_tracking",
      "name": "14. Fetch Order Tracking"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"message\": {\n    \"attachment\": {\n      \"type\": \"template\",\n      \"payload\": {\n        \"template_type\": \"button\",\n        \"text\": \"Your order #`{{ $json.choices[0].message.content }}` is currently *`{{ $json.data.status || 'in transit' }}`*.\\nEstimated delivery: `{{ $json.data.eta || 'not available' }}`.\",\n        \"buttons\": [\n          {\n            \"type\": \"web_url\",\n            \"url\": \"{{ $json.data.tracking_url || 'https://yourstore.com/track' }}\",\n            \"title\": \"Track Live\"\n          }\n        ]\n      }\n    }\n  },\n  \"messaging_type\": \"RESPONSE\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2640,
        2550
      ],
      "id": "c9e0f1g2-h3i4-5j6k-7l8m-9n0o1p2q3r4s_send_tracking",
      "name": "15. Send (Tracking Info + URL Button)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"sender_action\": \"typing_off\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2880,
        2550
      ],
      "id": "efb92aa3-2a9b-4d79-98cb-2b80e79d0ef1_tracking_info",
      "name": "10. Typing Off (Tracking Info)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"message\": {\n    \"text\": \"Please provide your order number to track your delivery.\"\n  },\n  \"messaging_type\": \"RESPONSE\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2400,
        2650
      ],
      "id": "c9e0f1g2-h3i4-5j6k-7l8m-9n0o1p2q3r4s_request_order_id_track",
      "name": "14. Send (Request Order ID)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/me/messages?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"recipient\": {\n    \"id\": \"{{ $json.PSID }}\"\n  },\n  \"sender_action\": \"typing_off\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2640,
        2650
      ],
      "id": "efb92aa3-2a9b-4d79-98cb-2b80e79d0ef1_request_order_id_track",
      "name": "10. Typing Off (Request Order ID)"
    }
  ],
  "connections": {
    "9. Switch (Route by Intent)": {
      "main": [
        [
          {
            "node": "12. AI (Order ID Extraction)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "12. AI (Order ID Extraction)": {
      "main": [
        [
          {
            "node": "13. If (Order ID Found)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "13. If (Order ID Found)": {
      "main": [
        [
          {
            "node": "14. Fetch Order Tracking",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "14. Send (Request Order ID)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "14. Fetch Order Tracking": {
      "main": [
        [
          {
            "node": "15. Send (Tracking Info + URL Button)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "15. Send (Tracking Info + URL Button)": {
      "main": [
        [
          {
            "node": "10. Typing Off (Tracking Info)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "14. Send (Request Order ID)": {
      "main": [
        [
          {
            "node": "10. Typing Off (Request Order ID)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```

---

### **3. One-Time Setup Workflow: Persistent Menu**

This workflow should be executed *once* to set the persistent menu for your Facebook Page. It is not part of the per-message response logic.

```json
{
  "name": "Setup Persistent Menu",
  "nodes": [
    {
      "parameters": {},
      "type": "n8n-nodes-base.start",
      "typeVersion": 1,
      "position": [
        240,
        140
      ],
      "id": "start-node-persistent-menu",
      "name": "Start (Manual Trigger)"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "https://graph.facebook.com/v19.0/me/messenger_profile?access_token={{ $credentials.facebookMessenger.accessToken }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"persistent_menu\": [\n    {\n      \"locale\": \"default\",\n      \"composer_input_disabled\": false,\n      \"call_to_actions\": [\n        {\n          \"type\": \"postback\",\n          \"title\": \"Shop Products\",\n          \"payload\": \"VIEW_PRODUCTS_INITIAL_PAYLOAD\"\n        },\n        {\n          \"type\": \"postback\",\n          \"title\": \"My Cart\",\n          \"payload\": \"VIEW_CART_PAYLOAD\"\n        },\n        {\n          \"type\": \"postback\",\n          \"title\": \"Track Order\",\n          \"payload\": \"TRACK_ORDER_INITIAL_PAYLOAD\"\n        },\n        {\n          \"type\": \"postback\",\n          \"title\": \"Help & FAQ\",\n          \"payload\": \"BUSINESS_INFO_INITIAL_PAYLOAD\"\n        },\n        {\n          \"type\": \"web_url\",\n          \"title\": \"Visit Our Website\",\n          \"url\": \"https://yourstore.com\",\n          \"webview_height_ratio\": \"full\"\n        },\n        {\n          \"type\": \"phone_number\",\n          \"title\": \"Call Us\",\n          \"payload\": \"+1-555-123-4567\" \n        }\n      ]\n    }\n  ]\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        480,
        140
      ],
      "id": "87320b94-fe5c-47f8-84e4-cbe43eeafb14_persistent_menu_setup",
      "name": "Set Persistent Menu"
    },
    {
      "parameters": {
        "message": "Persistent menu setup completed.",
        "options": {}
      },
      "type": "n8n-nodes-base.noOp",
      "typeVersion": 1,
      "position": [
        720,
        140
      ],
      "id": "no-op-persistent-menu-success",
      "name": "Success"
    }
  ],
  "connections": {
    "Start (Manual Trigger)": {
      "main": [
        [
          {
            "node": "Set Persistent Menu",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Set Persistent Menu": {
      "main": [
        [
          {
            "node": "Success",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  },
  "active": false,
  "settings": {
    "executionOrder": "v1"
  },
  "versionId": "setup-persistent-menu-v1",
  "meta": {
    "instanceId": "persistent-menu-setup-instance"
  }
}
```

---

### **Consolidating the Workflow**

You will combine all the individual workflow JSON snippets into one large JSON. In n8n, you typically paste each "branch" onto the canvas and then connect the starting node of each branch to the appropriate output of the "9. Switch (Route by Intent)" node.

**To implement this:**

1.  **Open a new n8n workflow.**
2.  **Import the "Core Workflow" JSON.** This will give you the Webhook to Switch nodes.
3.  **For each "Intent-Specific Branch":**
    *   Copy the JSON for that branch.
    *   In your n8n canvas, paste the JSON.
    *   Drag a connection from the corresponding output of the **"9. Switch (Route by Intent)"** node to the first node of that branch (e.g., "General Greeting AI Response").
4.  **Configure Credentials:** For all `HTTP Request` and `OpenAI Chat` nodes, ensure you set up and select your n8n credentials for Facebook Messenger (Page Access Token) and OpenAI (API Key). **Do not hardcode sensitive tokens in the workflow itself.**
5.  **Update Placeholders:** Replace `https://yourstore.com/api/...` with your actual API endpoints. Adjust `{{ $json.data.user_state }}` and similar expressions to match your database's response structure.
6.  **User State Management:** This is critical. For the `Set Quantity`, `Address Input`, and `Confirm Address` branches, you *must* implement a backend system (e.g., a simple database or a dedicated user state service) that stores the user's current conversational state based on their `PSID`. The `HTTP Request` nodes interacting with `https://yourstore.com/api/user_state/{{ $json.PSID }}` are placeholders for this.
7.  **Activate the Workflow:** Once configured and connected, activate the main workflow.
8.  **Run Persistent Menu Setup:** Execute the "Setup Persistent Menu" workflow once.

This detailed breakdown provides the architectural overview and the n8n workflow JSON snippets needed to build your sophisticated F-Commerce Messenger automation. Remember to test each branch thoroughly!
