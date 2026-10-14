# Pattern: find-or-create parent, then create child

Real, working workflow structure exported from appse ai, trimmed to its
structural fields only (node UI position/size and repeated app-catalog
metadata removed — see `conventions.md` for the full envelope shape those
omitted fields belong to). Read for structure only — never reuse these
literal field values, node names, or app-specific mappings.

**Demonstrates:** find-or-create a parent record, then create a child record
that depends on it. Trigger (Shopify new order) → search D365 BC customer by
email → `DecisionNode` on customer `number exist` → **both branches end in
the same child action (create sales order), but get the customer number
differently:**

- `true` (customer exists) → `Create a new sales order`, with
  `customerNumber` taken from the **search result**
  (`$('<search node>').payload.number`)
- `false` (customer missing) → `Create a new customer` → **then, in
  sequence,** `Create a new sales order`, with `customerNumber` set to the
  same value the new customer was just created with

Use when the child record (order, invoice, shipment) cannot be created
without a parent (customer, business partner) that may not exist yet. This
differs from create-or-update: the "found" branch does not update the
parent, it just reuses it.

**No guard on a blank source email — the same failure class as a blank
search key returning every record. Expert review must add a Filter before
the customer search.**

**Corrected from the original export (2026-09-24), not unmodified:**
- The `true`-branch order line's `unitPrice` pointed at the order-level
  `currentSubtotalPriceSet` (the whole order's subtotal) instead of the
  per-line `lineItems.nodes[].originalUnitPriceSet` used on the `false`
  branch. That would have priced every line at the full order subtotal for
  returning customers. Corrected to match the `false` branch (the snippet
  below reflects the corrected version).
- The final edge (create customer → create sales order) was missing
  `"targetHandle": "default"`; added for consistency with every other edge.

**Not confirmed by this file:** whether `salesOrderLines` with a single
object using `nodes[]` projections expands to one line per Shopify line
item at runtime, or needs a `SplitterNode`. Confirm before relying on it
for multi-line orders.

**Status: reference for find-or-create-then-create-child.** The app-specific
mappings (Shopify GID stripping via `substringAfter`, `type: "Person"`) are
field values — never copy them; resolve them fresh per Steps 6–7.

```json
{
  "nodes": [
    {
      "id": "d0c3e827-5924-4526-8f18-dc068ef34715",
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
        "properties": { "limit": "10", "since_from": "2026-03-19T11:38:00.000Z" },
        "action_code": "on_new_orders_created",
        "action_name": "New orders created",
        "current_name": "Shopify",
        "original_name": "Shopify",
        "next_data_from_template": "={{date_max_by($payload[*],&createdAt) }}"
      }
    },
    {
      "id": "e3ed575a-d68f-4a37-be93-b2ecc58389f3",
      "type": "AppNode",
      "data": {
        "app": { "code": "dynamics365businesscentral" },
        "idx": 2,
        "settings": {
          "on_error": "stop",
          "execute_once": false,
          "retry_on_fail": false,
          "always_output_data": true
        },
        "properties": { "email": "={{$payload.customer.email}}" },
        "action_code": "get_customer_by_email",
        "action_name": "Get customer by email",
        "current_name": "Dynamics 365 Business Central 1",
        "original_name": "Dynamics 365 Business Central"
      }
    },
    {
      "id": "6f7260d5-4a42-40c8-bcba-136e87647132",
      "type": "DecisionNode",
      "data": {
        "idx": 3,
        "settings": { "disable_case_sensitivity": false },
        "properties": {
          "advance_filter": [
            [
              {
                "operator": { "type": "basic", "operation": "exist" },
                "leftValue": "={{$payload.number}}",
                "rightValue": ""
              }
            ]
          ]
        },
        "current_name": "Decision",
        "original_name": "Decision"
      }
    },
    {
      "id": "56a181c4-bb59-4474-be14-418f0ecd6aad",
      "type": "AppNode",
      "data": {
        "app": { "code": "dynamics365businesscentral" },
        "idx": 4,
        "settings": {
          "on_error": "stop",
          "execute_once": false,
          "retry_on_fail": false,
          "always_output_data": false
        },
        "properties": {
          "number": "={{substringAfter($('Shopify').payload.id,'gid://shopify/Order/')}}",
          "orderDate": "={{substringBefore($('Shopify').payload.createdAt,'T')}}",
          "customerNumber": "={{$('Dynamics 365 Business Central 1').payload.number}}",
          "salesOrderLines": [
            {
              "lineType": "Item",
              "quantity": "={{$('Shopify').payload.lineItems.nodes[].quantity}}",
              "unitPrice": "={{$('Shopify').payload.lineItems.nodes[].originalUnitPriceSet.presentmentMoney.amount}}",
              "lineObjectNumber": "={{$('Shopify').payload.lineItems.nodes[].sku}}"
            }
          ]
        },
        "action_code": "create_salesorder",
        "action_name": "Create a new sales order",
        "current_name": "Dynamics 365 Business Central",
        "original_name": "Dynamics 365 Business Central"
      }
    },
    {
      "id": "74b0b667-d799-4342-bdf5-a1c17b385164",
      "type": "AppNode",
      "data": {
        "app": { "code": "dynamics365businesscentral" },
        "idx": 5,
        "settings": {
          "on_error": "stop",
          "execute_once": false,
          "retry_on_fail": false,
          "always_output_data": false
        },
        "properties": {
          "type": "Person",
          "email": "={{$('Shopify').payload.customer.email}}",
          "number": "={{substringAfter($('Shopify').payload.customer.id,'gid://shopify/Customer/')}}",
          "displayName": "={{$('Shopify').payload.billingAddress.firstName}} {{$('Shopify').payload.billingAddress.lastName}}"
        },
        "action_code": "create_customer",
        "action_name": "Create a new customer",
        "current_name": "Dynamics 365 Business Central 3",
        "original_name": "Dynamics 365 Business Central"
      }
    },
    {
      "id": "3cf5c145-5b0e-4054-9679-3d685cf6d103",
      "type": "AppNode",
      "data": {
        "app": { "code": "dynamics365businesscentral" },
        "idx": 6,
        "settings": {
          "on_error": "stop",
          "execute_once": false,
          "retry_on_fail": false,
          "always_output_data": false
        },
        "properties": {
          "number": "={{substringAfter($('Shopify').payload.id,'gid://shopify/Order/')}}",
          "orderDate": "={{substringBefore($('Shopify').payload.createdAt,'T')}}",
          "customerNumber": "={{substringAfter($('Shopify').payload.customer.id,'gid://shopify/Customer/')}}",
          "salesOrderLines": [
            {
              "lineType": "Item",
              "quantity": "={{$('Shopify').payload.lineItems.nodes[].quantity}}",
              "unitPrice": "={{$('Shopify').payload.lineItems.nodes[].originalUnitPriceSet.presentmentMoney.amount}}",
              "lineObjectNumber": "={{$('Shopify').payload.lineItems.nodes[].sku}}"
            }
          ]
        },
        "action_code": "create_salesorder",
        "action_name": "Create a new sales order",
        "current_name": "Dynamics 365 Business Central 2",
        "original_name": "Dynamics 365 Business Central"
      }
    }
  ],
  "edges": [
    { "id": "75708518-d838-4ab5-84a7-1b5dd39db254", "source": "d0c3e827-5924-4526-8f18-dc068ef34715", "target": "e3ed575a-d68f-4a37-be93-b2ecc58389f3", "sourceHandle": "default", "targetHandle": "default" },
    { "id": "e1193304-59a7-4cf9-9967-373b6430fafe", "source": "e3ed575a-d68f-4a37-be93-b2ecc58389f3", "target": "6f7260d5-4a42-40c8-bcba-136e87647132", "sourceHandle": "default", "targetHandle": "default" },
    { "id": "3b711c41-5860-4def-a086-4476b3b4b2a0", "source": "6f7260d5-4a42-40c8-bcba-136e87647132", "target": "56a181c4-bb59-4474-be14-418f0ecd6aad", "sourceHandle": "true", "targetHandle": "default" },
    { "id": "cd14341f-1b6b-4dd6-ae51-216d9e10e51e", "source": "6f7260d5-4a42-40c8-bcba-136e87647132", "target": "74b0b667-d799-4342-bdf5-a1c17b385164", "sourceHandle": "false", "targetHandle": "default" },
    { "id": "857a5a58-944a-4e35-af63-24e25277c434", "source": "74b0b667-d799-4342-bdf5-a1c17b385164", "target": "3cf5c145-5b0e-4054-9679-3d685cf6d103", "sourceHandle": "default", "targetHandle": "default" }
  ]
}
```

Note the `true` branch's sales-order node is named `"Dynamics 365 Business
Central"` (no suffix) while the `false`-branch chain uses `...3` then
`...2` — a reminder that a reference file's `current_name`s are that
workflow's own history, not a naming convention to copy.
