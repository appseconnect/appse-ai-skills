# Pattern: SKU/item reconciliation + multi-branch order creation

Real, working workflow structure exported from appse ai, trimmed to its
structural fields only (node UI position/size and repeated app-catalog
metadata removed — see `conventions.md` for the full envelope shape those
omitted fields belong to). Read for structure only — never reuse these
literal field values, node names, or app-specific mappings.

**Demonstrates three things in one workflow, all branching off the same
Shopify "new order" trigger:**

1. **SKU/item reconciliation:** `SplitterNode` (fans out order line items) →
   `Get Item by ItemCode` → `FilterNode` (item exists) → `Create New Items`.
   **Approved reference for SKU/item reconciliation** — for the *shape*
   only. **Don't copy its `$('Splitter')` references** in the Filter and
   Create nodes: they read the Splitter from past a Filter. When a Filter
   sits in between, read the element from the **last Filter** instead
   (e.g. `$('Has SKU').payload.sku`) — live builds got `null` by copying
   this.
2. **Single-criterion entity resolution** (email only) for Business
   Partner — dedupe-and-skip style, not a fallback cascade. Still no
   confirmed example of a true email→phone→name cascade.
3. **An AI node (`get_chat_completions`, GPT-4.1) used mid-workflow** for
   order-total reconciliation, feeding a `JsonConverterNode`, feeding an
   `Update SalesOrder` call. Real and working, but **not something the
   partner originally asked for — do not treat this as an approved pattern
   for the skill to build until confirmed with the senior.** Kept in this
   file only because it's real data; the skill should not replicate the
   AI-node portion of this shape without explicit sign-off (see `SKILL.md`
   — never build it unless the partner explicitly asks for AI-assisted
   reconciliation).

**Status: approved for the SKU-reconciliation and dedupe-and-skip portions;
the AI-node portion requires sign-off before building.**

```json
{
  "nodes": [
    {
      "id": "ef291040-fb8d-40ab-8eba-2b3d376a6c8e",
      "type": "AppTriggerNode",
      "data": {
        "app": { "code": "shopify" },
        "idx": 1,
        "cron": "*/3 * * * *",
        "settings": {
          "on_error": "stop",
          "execute_once": false,
          "retry_on_fail": false,
          "always_output_data": false
        },
        "properties": { "limit": "10", "since_from": "2026-03-05T04:10:00.000Z" },
        "action_code": "on_new_orders_created",
        "action_name": "New orders created",
        "current_name": "Fetch Newly Created Orders",
        "original_name": "Shopify",
        "next_data_from_template": "={{date_max_by($payload[*],&createdAt) }}"
      }
    },

    { "_comment": "--- Branch 1: SKU/item reconciliation ---" },
    {
      "id": "c5adc6ca-6469-4eb4-9419-e98829e697a2",
      "type": "SplitterNode",
      "data": {
        "idx": 2,
        "include": "no_other_fields",
        "current_name": "Splitting the Items",
        "original_name": "Splitter",
        "fields_to_split": "lineItems.nodes"
      }
    },
    {
      "id": "b2d75436-4b2d-4659-b039-6e2e7640ee97",
      "type": "AppNode",
      "data": {
        "app": { "code": "sapbusinessone" },
        "idx": 3,
        "settings": {
          "on_error": "stop",
          "execute_once": false,
          "retry_on_fail": false,
          "always_output_data": true
        },
        "properties": { "ItemCode": "={{$payload.sku}}" },
        "action_code": "get_item_by_itemcode",
        "action_name": "Get a Item by ItemCode",
        "current_name": "Get Item by ItemCode",
        "original_name": "SAP Business One"
      }
    },
    {
      "id": "e848a5ba-fd9e-4821-af9c-99eaef76d22e",
      "type": "FilterNode",
      "data": {
        "idx": 4,
        "settings": { "disable_case_sensitivity": false },
        "properties": {
          "advance_filter": [
            [
              {
                "operator": { "type": "string", "operation": "not_equal" },
                "leftValue": "={{$payload.value[].ItemCode}}",
                "rightValue": "={{$('Splitter').payload.sku}}"
              }
            ]
          ]
        },
        "current_name": "Item Exists",
        "original_name": "Filter"
      }
    },
    {
      "id": "bb9b4cb1-5203-41a3-a8b5-fe8f1b581cc4",
      "type": "AppNode",
      "data": {
        "app": { "code": "sapbusinessone" },
        "idx": 5,
        "settings": {
          "on_error": "stop",
          "execute_once": false,
          "retry_on_fail": false,
          "always_output_data": false
        },
        "properties": {
          "ItemCode": "={{$('Splitter').payload.sku}}",
          "ItemName": "={{$('Splitter').payload.name}}",
          "ItemPrices": [
            {
              "Price": "={{$('Splitter').payload.originalUnitPriceSet.presentmentMoney.amount}}",
              "Currency": "$$",
              "PriceList": "1"
            }
          ]
        },
        "action_code": "create_item",
        "action_name": "Create a new item",
        "current_name": "Create New Items",
        "original_name": "SAP Business One"
      }
    },

    { "_comment": "--- Branch 2: Business Partner dedupe-and-skip ---" },
    {
      "id": "a3ba8d08-fbe2-4785-883c-b0ecf8d19bc4",
      "type": "AppNode",
      "data": {
        "app": { "code": "sapbusinessone" },
        "idx": 6,
        "settings": {
          "on_error": "stop",
          "execute_once": false,
          "retry_on_fail": false,
          "always_output_data": true
        },
        "properties": { "EmailAddress": "={{$payload.customer.email}}" },
        "action_code": "get_businesspartner_by_email",
        "action_name": "Get a BusinessPartner by EmailAddress",
        "current_name": "Fetch Business Partner by Email",
        "original_name": "SAP Business One"
      }
    },
    {
      "id": "5b140f3d-40f2-43f3-b0f9-7f5e73ab168d",
      "type": "FilterNode",
      "data": {
        "idx": 7,
        "settings": { "disable_case_sensitivity": false },
        "properties": {
          "advance_filter": [
            [
              {
                "operator": { "type": "string", "operation": "not_equal" },
                "leftValue": "={{$payload.CardCode}}",
                "rightValue": "={{split($('Shopify').payload.customer.id,'/')[4]}}"
              }
            ]
          ]
        },
        "current_name": "Business Partner Exists",
        "original_name": "Filter"
      }
    },
    {
      "id": "98c3792c-23d5-4bad-92ce-640f425fe87b",
      "type": "AppNode",
      "data": {
        "app": { "code": "sapbusinessone" },
        "idx": 8,
        "settings": {
          "on_error": "stop",
          "execute_once": false,
          "retry_on_fail": false,
          "always_output_data": false
        },
        "properties": {
          "Phone1": "={{$('Shopify').payload.customer.phone}}",
          "CardCode": "={{split($('Shopify').payload.customer.id,'/')[4]}}",
          "CardName": "={{$('Shopify').payload.customer.firstName}} {{$('Shopify').payload.customer.lastName}}",
          "BPAddresses": [
            {
              "City": "={{$('Shopify').payload.billingAddress.city}}",
              "Block": "={{$('Shopify').payload.billingAddress.address2}}",
              "State": "={{$('Shopify').payload.billingAddress.provinceCode}}",
              "Street": "={{$('Shopify').payload.billingAddress.address1}}",
              "Country": "={{$('Shopify').payload.billingAddress.countryCodeV2}}",
              "ZipCode": "={{$('Shopify').payload.billingAddress.zip}}",
              "AddressName": "={{$('Shopify').payload.billingAddress.firstName}} {{$('Shopify').payload.billingAddress.lastName}}",
              "AddressType": "bo_BillTo"
            },
            {
              "City": "={{$('Shopify').payload.shippingAddress.city}}",
              "Block": "={{$('Shopify').payload.shippingAddress.address2}}",
              "State": "={{$('Shopify').payload.shippingAddress.provinceCode}}",
              "Street": "={{$('Shopify').payload.shippingAddress.address1}}",
              "Country": "={{$('Shopify').payload.shippingAddress.countryCodeV2}}",
              "ZipCode": "={{$('Shopify').payload.shippingAddress.zip}}",
              "AddressName": "={{$('Shopify').payload.shippingAddress.firstName}} {{$('Shopify').payload.shippingAddress.lastName}}",
              "AddressType": "bo_ShipTo"
            }
          ],
          "EmailAddress": "={{$('Shopify').payload.customer.email}}"
        },
        "action_code": "create_businesspartner",
        "action_name": "Create a new businesspartner",
        "current_name": "Create new Business Partners",
        "original_name": "SAP Business One"
      }
    },

    { "_comment": "--- Branch 3: order creation + AI-assisted total reconciliation (sign-off required) ---" },
    {
      "id": "81b18cf2-6188-4457-8e54-62888145e415",
      "type": "AppNode",
      "data": {
        "app": { "code": "sapbusinessone" },
        "idx": 9,
        "settings": {
          "on_error": "stop",
          "execute_once": false,
          "retry_on_fail": false,
          "always_output_data": true
        },
        "properties": { "NumAtCard": "={{split($payload.id, '/')[4]}}" },
        "action_code": "get_orders_by_numatcard",
        "action_name": "Get an sales order by customer reference no",
        "current_name": "Fetch SalesOrder by Reference Number",
        "original_name": "SAP Business One"
      }
    },
    {
      "id": "c6ab4122-4200-4261-b433-bc77d39ca7bb",
      "type": "DecisionNode",
      "data": {
        "idx": 10,
        "settings": { "disable_case_sensitivity": false },
        "properties": {
          "advance_filter": [
            [
              {
                "operator": { "type": "string", "operation": "equal" },
                "leftValue": "={{$payload.NumAtCard}}",
                "rightValue": "={{split($('Shopify').payload.id, '/')[4]}}"
              }
            ]
          ]
        },
        "current_name": "SalesOrder Exists",
        "original_name": "Decision"
      }
    },
    {
      "id": "4b3ae57a-0002-4d57-8a20-003a92fca177",
      "type": "AppNode",
      "data": {
        "app": { "code": "sapbusinessone" },
        "idx": 11,
        "settings": {
          "on_error": "stop",
          "execute_once": false,
          "retry_on_fail": false,
          "always_output_data": false
        },
        "properties": { "EmailAddress": "={{$('Shopify').payload.customer.email}}" },
        "action_code": "get_businesspartner_by_email",
        "action_name": "Get a BusinessPartner by EmailAddress",
        "current_name": "Fetch Business Partners by Email",
        "original_name": "SAP Business One"
      }
    },
    {
      "id": "728415b1-a0e7-45bc-ad2c-be8dabd55aa2",
      "type": "AppNode",
      "data": {
        "app": { "code": "sapbusinessone" },
        "idx": 12,
        "settings": {
          "on_error": "stop",
          "execute_once": false,
          "retry_on_fail": false,
          "always_output_data": true
        },
        "properties": {
          "DocDate": "={{$('Shopify').payload.createdAt}}",
          "CardCode": "={{$payload.CardCode}}",
          "DocTotal": "={{$('Shopify').payload.transactions[].amountSet.presentmentMoney.amount}}",
          "NumAtCard": "={{split($('Shopify').payload.id, '/')[4]}}",
          "DocDueDate": "={{$('Shopify').payload.createdAt}}",
          "DocObjectCode": "17",
          "DocumentLines": [
            {
              "TaxCode": "={{$('Shopify').payload.lineItems.nodes[0].taxLines[0].title}}",
              "ItemCode": "={{$('Shopify').payload.lineItems.nodes[].sku}}",
              "Quantity": "={{$('Shopify').payload.lineItems.nodes[].quantity}}",
              "UnitPrice": "={{$('Shopify').payload.lineItems.nodes[].originalUnitPriceSet.presentmentMoney.amount}}",
              "DiscountPercent": "={{$('Shopify').payload.discountApplications.nodes[0].value.percentage}}"
            }
          ],
          "DocumentAdditionalExpenses": [
            {
              "LineTotal": "={{$('Shopify').payload.shippingLines.nodes[].originalPriceSet.presentmentMoney.amount}}",
              "ExpenseCode": "1"
            }
          ]
        },
        "action_code": "create_order",
        "action_name": "Create a new sales order",
        "current_name": "Create New SalesOrder",
        "original_name": "SAP Business One"
      }
    },
    {
      "id": "24cb931b-a6a4-41ff-a264-7e318f5392ab",
      "type": "AppNode",
      "data": {
        "app": { "code": "openai" },
        "idx": 13,
        "settings": {
          "on_error": "stop",
          "execute_once": false,
          "retry_on_fail": false,
          "always_output_data": false
        },
        "properties": {
          "model": "gpt-4.1",
          "messages": [
            { "role": "system", "content": "<reconciliation instructions, omitted here — structure only>" },
            { "role": "user", "content": "<order totals from both systems, interpolated — omitted here>" }
          ],
          "response_format": "={ \"type\": \"json_object\" }"
        },
        "action_code": "get_chat_completions",
        "action_name": "Create chat completions",
        "current_name": "Smart Order Total Reconciliation",
        "original_name": "OpenAI"
      }
    },
    {
      "id": "11265dd1-c5fe-4faf-a3a7-e9868c96e8f8",
      "type": "JsonConverterNode",
      "data": {
        "idx": 14,
        "properties": { "fieldsToConvert": "={{$payload.choices[].message.content}}" },
        "current_name": "JSON Converter",
        "original_name": "JSON Converter",
        "conversionType": "string_to_json"
      }
    },
    {
      "id": "2e7043b1-945b-4b55-839e-f0d52306052c",
      "type": "AppNode",
      "data": {
        "app": { "code": "sapbusinessone" },
        "idx": 15,
        "settings": {
          "on_error": "stop",
          "execute_once": false,
          "retry_on_fail": false,
          "always_output_data": false
        },
        "properties": {
          "DocEntry": "={{$('SAP Business One 2').payload.DocEntry}}",
          "Rounding": "tYES",
          "NumAtCard": "={{$('SAP Business One 2').payload.NumAtCard}}",
          "RoundingDiffAmount": "={{$payload.Difference}}"
        },
        "action_code": "update_salesorder_customerrefno",
        "action_name": "Update salaesorder customer reference no",
        "current_name": "Update SalesOrder",
        "original_name": "SAP Business One"
      }
    }
  ],
  "edges": [
    { "id": "66161454-bbb5-4b66-853f-49c0c403afbf", "source": "ef291040-fb8d-40ab-8eba-2b3d376a6c8e", "target": "c5adc6ca-6469-4eb4-9419-e98829e697a2", "sourceHandle": "default", "targetHandle": "default" },
    { "id": "f9405178-e1e2-430b-a051-ae97500d385a", "source": "c5adc6ca-6469-4eb4-9419-e98829e697a2", "target": "b2d75436-4b2d-4659-b039-6e2e7640ee97", "sourceHandle": "default", "targetHandle": "default" },
    { "id": "de3396c6-1c72-4be6-81e3-61caf69a759b", "source": "b2d75436-4b2d-4659-b039-6e2e7640ee97", "target": "e848a5ba-fd9e-4821-af9c-99eaef76d22e", "sourceHandle": "default", "targetHandle": "default" },
    { "id": "07638133-8176-4206-a413-a2c3e33cd21c", "source": "e848a5ba-fd9e-4821-af9c-99eaef76d22e", "target": "bb9b4cb1-5203-41a3-a8b5-fe8f1b581cc4", "sourceHandle": "default", "targetHandle": "default" },

    { "id": "1b5580a0-562a-417a-909f-97273ca7f642", "source": "ef291040-fb8d-40ab-8eba-2b3d376a6c8e", "target": "a3ba8d08-fbe2-4785-883c-b0ecf8d19bc4", "sourceHandle": "default", "targetHandle": "default" },
    { "id": "e58c2292-a4ad-478f-b767-179e1f162b37", "source": "a3ba8d08-fbe2-4785-883c-b0ecf8d19bc4", "target": "5b140f3d-40f2-43f3-b0f9-7f5e73ab168d", "sourceHandle": "default", "targetHandle": "default" },
    { "id": "a318b82a-86b1-461e-91e2-c64dcd9748f0", "source": "5b140f3d-40f2-43f3-b0f9-7f5e73ab168d", "target": "98c3792c-23d5-4bad-92ce-640f425fe87b", "sourceHandle": "default", "targetHandle": "default" },

    { "id": "d6f77c52-7362-4adb-b791-325e5aae8dba", "source": "ef291040-fb8d-40ab-8eba-2b3d376a6c8e", "target": "81b18cf2-6188-4457-8e54-62888145e415", "sourceHandle": "default", "targetHandle": "default" },
    { "id": "311475b3-7620-4274-8c15-6f2e7746005d", "source": "81b18cf2-6188-4457-8e54-62888145e415", "target": "c6ab4122-4200-4261-b433-bc77d39ca7bb", "sourceHandle": "default", "targetHandle": "default" },
    { "id": "47993958-1456-4f97-804a-c21d42887326", "source": "c6ab4122-4200-4261-b433-bc77d39ca7bb", "target": "4b3ae57a-0002-4d57-8a20-003a92fca177", "sourceHandle": "false", "targetHandle": "default" },
    { "id": "e9f1ce0e-3642-4b0c-bcc9-8fb24c12ca27", "source": "4b3ae57a-0002-4d57-8a20-003a92fca177", "target": "728415b1-a0e7-45bc-ad2c-be8dabd55aa2", "sourceHandle": "default", "targetHandle": "default" },
    { "id": "3d36c055-4583-4339-9e99-159e6b6319f3", "source": "728415b1-a0e7-45bc-ad2c-be8dabd55aa2", "target": "24cb931b-a6a4-41ff-a264-7e318f5392ab", "sourceHandle": "default", "targetHandle": "default" },
    { "id": "be22f2d0-2527-4011-8b3b-8f32c42fc411", "source": "24cb931b-a6a4-41ff-a264-7e318f5392ab", "target": "11265dd1-c5fe-4faf-a3a7-e9868c96e8f8", "sourceHandle": "default", "targetHandle": "default" },
    { "id": "2503adb8-add7-4f5c-9d4b-8af64b167139", "source": "11265dd1-c5fe-4faf-a3a7-e9868c96e8f8", "target": "2e7043b1-945b-4b55-839e-f0d52306052c", "sourceHandle": "default", "targetHandle": "default" }
  ]
}
```

**Reading this one:** three branches leave the trigger node directly (to
the Splitter, to the Business Partner lookup, and to the SalesOrder
lookup) — they run independently, same as the parallel-branch pattern.
The `_comment` keys above are added by this file to mark where each
branch starts; they are not part of the real saved JSON (which has no
comment syntax) — remove them if copying structure into an actual build.
The `FilterNode`/`JsonConverterNode` references to `'Splitter'` and the AI
nodes are exactly the parts flagged above as **don't copy**.
