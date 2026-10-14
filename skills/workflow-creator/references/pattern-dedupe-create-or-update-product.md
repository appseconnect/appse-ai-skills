# Pattern: create-or-update (product, ERP → store)

Real, working workflow structure exported from appse ai, trimmed to its
structural fields only (node UI position/size and repeated app-catalog
metadata removed — see `conventions.md` for the full envelope shape those
omitted fields belong to). Read for structure only — never reuse these
literal field values, node names, or app-specific mappings. **Supplied
2026-09-24; tested status not stated — confirm with the team.**

**Demonstrates:** create-or-update for **products**, in the usual direction —
ERP is the product master, pushing to the store. Trigger (D365 BC items
updated) → Magento2 `Get product by SKU` (`sku: {{$payload.number}}`,
`always_output_data: true`) → `DecisionNode` "SKU exists in Magento"
(`$payload.sku` **equal** `$('Dynamics 365 Business Central').payload.number`)
→ `true` → `Update a product` (SKU in the URL field `sku`, fields in a
`product` object), `false` → `Create a product` (everything inside
`product`).

**What it confirms:**
- The lookup straight after the trigger reads `$payload` (same as every
  other reference).
- Create/update nodes after a lookup + Decision read source fields from the
  trigger **by its `current_name`** (`$('Dynamics 365 Business Central')`) —
  with **no Filter in between**. Consistent with the other working
  references; it does not tell us what happens when a Filter sits in the
  path.
- Magento2 wraps product fields in a `product` object; update takes the SKU
  outside it.

**Values not to copy blindly:**
- `type_id: "simple"`, `status: "1"` (enabled), `visibility: "4"` (catalog
  and search) are standard Magento codes — reasonable defaults, but still
  list them under "Mappings I worked out" in Step 9.
- `attribute_set_id: "4"` is Magento's default attribute set, but many
  stores use their own — treat it as a **company-specific setting** (ask).

**Gaps an expert build should close (not edited here — this is the export
as supplied):**
- No guard for an empty item number before the lookup, and the Decision
  only checks `equal` — add the not-empty checks per "Think Like an
  Integration Expert".
- The update re-sends `status: "1"` and `visibility: "4"`, which would
  re-enable a product someone deliberately disabled in Magento. On updates,
  leave out fields the store team manages unless the partner wants them
  synced.
- Trigger is "Items **updated**" with a start date of 2026-02-19 — confirm
  it also fires for newly created items, and start new builds from now.

**Status: reference for create-or-update of products (ERP → e-commerce).**

```json
{
  "nodes": [
    {
      "id": "ccdd0213-db83-4102-a07a-231433f471d5",
      "type": "AppTriggerNode",
      "data": {
        "app": { "code": "dynamics365businesscentral" },
        "idx": 1,
        "cron": "*/3 * * * *",
        "settings": {
          "on_error": "stop",
          "execute_once": false,
          "retry_on_fail": false,
          "always_output_data": false
        },
        "properties": { "limit": "10", "since_from": "2026-02-19T12:51:00.000Z" },
        "action_code": "on_items_updated",
        "action_name": "Items updated",
        "current_name": "Dynamics 365 Business Central",
        "original_name": "Dynamics 365 Business Central",
        "next_data_from_template": "={{date_max_by($payload[*],&lastModifiedDateTime) }}"
      }
    },
    {
      "id": "6ed4c325-6fa1-453a-9d79-9cdac670a3d7",
      "type": "AppNode",
      "data": {
        "app": { "code": "magento2" },
        "idx": 2,
        "settings": {
          "on_error": "stop",
          "execute_once": false,
          "retry_on_fail": false,
          "always_output_data": true
        },
        "properties": { "sku": "={{$payload.number}}" },
        "action_code": "get_product_by_sku",
        "action_name": "Get product by SKU",
        "current_name": "Search item by input sku",
        "original_name": "Magento2"
      }
    },
    {
      "id": "689716ac-621c-4fbf-8916-1b13c116443b",
      "type": "DecisionNode",
      "data": {
        "idx": 3,
        "settings": { "disable_case_sensitivity": false },
        "properties": {
          "advance_filter": [
            [
              {
                "operator": { "type": "string", "operation": "equal" },
                "leftValue": "={{$payload.sku}}",
                "rightValue": "={{$('Dynamics 365 Business Central').payload.number}}"
              }
            ]
          ]
        },
        "current_name": "SKU exists in Magento",
        "original_name": "Decision"
      }
    },
    {
      "id": "e1a9050f-4441-4268-9dd7-a149e5307e66",
      "type": "AppNode",
      "data": {
        "app": { "code": "magento2" },
        "idx": 4,
        "settings": {
          "on_error": "stop",
          "execute_once": false,
          "retry_on_fail": false,
          "always_output_data": false
        },
        "properties": {
          "sku": "={{$('Dynamics 365 Business Central').payload.number}}",
          "product": {
            "name": "={{$('Dynamics 365 Business Central').payload.displayName}}",
            "price": "={{$('Dynamics 365 Business Central').payload.unitPrice}}",
            "status": "1",
            "type_id": "simple",
            "visibility": "4",
            "attribute_set_id": "4"
          }
        },
        "action_code": "update_product",
        "action_name": "Update a product",
        "current_name": "Update Existing Products",
        "original_name": "Magento2"
      }
    },
    {
      "id": "ea123c3e-dc67-4742-a6a2-144b25735a32",
      "type": "AppNode",
      "data": {
        "app": { "code": "magento2" },
        "idx": 5,
        "settings": {
          "on_error": "stop",
          "execute_once": false,
          "retry_on_fail": false,
          "always_output_data": false
        },
        "properties": {
          "product": {
            "sku": "={{$('Dynamics 365 Business Central').payload.number}}",
            "name": "={{$('Dynamics 365 Business Central').payload.displayName}}",
            "price": "={{$('Dynamics 365 Business Central').payload.unitPrice}}",
            "status": "1",
            "type_id": "simple",
            "visibility": "4",
            "attribute_set_id": "4"
          }
        },
        "action_code": "create_product",
        "action_name": "Create a product",
        "current_name": "Create new products",
        "original_name": "Magento2"
      }
    }
  ],
  "edges": [
    { "id": "df42157c-caf4-44aa-abec-0da1497b05de", "source": "ccdd0213-db83-4102-a07a-231433f471d5", "target": "6ed4c325-6fa1-453a-9d79-9cdac670a3d7", "sourceHandle": "default", "targetHandle": "default" },
    { "id": "2f20f9f5-9515-4398-a8ee-8c4db008495d", "source": "6ed4c325-6fa1-453a-9d79-9cdac670a3d7", "target": "689716ac-621c-4fbf-8916-1b13c116443b", "sourceHandle": "default", "targetHandle": "default" },
    { "id": "f7567ed2-aeb5-458d-88b9-763aea133db9", "source": "689716ac-621c-4fbf-8916-1b13c116443b", "target": "ea123c3e-dc67-4742-a6a2-144b25735a32", "sourceHandle": "false", "targetHandle": "default" },
    { "id": "a7d51ff5-da8c-4826-9991-af0d2d463c22", "source": "689716ac-621c-4fbf-8916-1b13c116443b", "target": "e1a9050f-4441-4268-9dd7-a149e5307e66", "sourceHandle": "true", "targetHandle": "default" }
  ]
}
```

Note the field placement difference between the two Magento2 actions:
`create_product` puts every field inside `product`; `update_product` pulls
`sku` out to the top level (it identifies the record) and keeps the rest
inside `product`.
