# Pattern: create-or-update (with nested sub-record row IDs)

Real, working workflow structure exported from appse ai, trimmed to its
structural fields only (node UI position/size and repeated app-catalog
metadata removed — see `conventions.md` for the full envelope shape those
omitted fields belong to). Read for structure only — never reuse these
literal field values, node names, or app-specific mappings.

**Demonstrates:** a second, independent confirmation of create-vs-update —
Trigger (Shopify customer updated) → search SAP B1 by email →
`DecisionNode` → `false` → `create_businesspartner`, `true` →
`update_businesspartner`. Different app pair from the customer example,
same confirmed shape — strengthens confidence this pattern generalizes
rather than being a one-off.

**Additional detail this file reveals that the customer example didn't:**
when the update touches a nested array field (`BPAddresses`), the update
payload references the _original_ record's row identifier —
`{{$payload.BPAddresses[idx].RowNum}}`, pulled from the search result — to
correctly target the existing sub-record. Without this, an update could
silently create a duplicate address entry instead of updating the existing
one. This is API-specific behavior (SAP B1 treats addresses as a row-
identified sub-collection; Magento2's `update_customer` in the other
pattern just replaces the whole array by `customerId`, no row IDs needed) —
**when building an update that touches a nested array, check whether the
target API needs this row-preservation pattern rather than assuming
Magento2's simpler replace-the-array behavior is universal.**

**Status: approved reference for create-vs-update; use this file
specifically when nested/sub-record fields are involved.**

```json
{
  "nodes": [
    {
      "id": "2c768c95-b32c-4e0e-96bc-2f91850b50c6",
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
        "properties": { "limit": "10", "since_from": "2025-12-11T04:10:00.000Z" },
        "action_code": "on_new_customers_updated",
        "action_name": "Customers updated",
        "current_name": "Shopify",
        "original_name": "Shopify",
        "next_data_from_template": "={{date_max_by($payload[*],&updatedAt) }}"
      }
    },
    {
      "id": "964b98bb-24e7-4606-a2f4-3e01e7b32275",
      "type": "AppNode",
      "data": {
        "app": { "code": "sapbusinessone" },
        "idx": 2,
        "settings": {
          "on_error": "stop",
          "execute_once": false,
          "retry_on_fail": false,
          "always_output_data": true
        },
        "properties": { "EmailAddress": "={{$payload.defaultEmailAddress.emailAddress}}" },
        "action_code": "get_businesspartner_by_email",
        "action_name": "Get a BusinessPartner by EmailAddress",
        "current_name": "SAP Business One",
        "original_name": "SAP Business One"
      }
    },
    {
      "id": "8192e43a-6d6e-4f97-bc21-4e6d086a3c70",
      "type": "DecisionNode",
      "data": {
        "idx": 3,
        "settings": { "disable_case_sensitivity": false },
        "properties": {
          "advance_filter": [
            [
              {
                "operator": { "type": "string", "operation": "equal" },
                "leftValue": "={{$payload.EmailAddress}}",
                "rightValue": "={{$('Shopify').payload.defaultEmailAddress.emailAddress}}"
              }
            ]
          ]
        },
        "current_name": "Decision",
        "original_name": "Decision"
      }
    },
    {
      "id": "ddfc3ffe-6efd-4dbc-9f64-91f677fc7a46",
      "type": "AppNode",
      "data": {
        "app": { "code": "sapbusinessone" },
        "idx": 4,
        "settings": {
          "on_error": "stop",
          "execute_once": false,
          "retry_on_fail": false,
          "always_output_data": true
        },
        "properties": {
          "Phone1": "={{$('Shopify').payload.defaultPhoneNumber.phoneNumber}}",
          "CardCode": "={{$payload.CardCode}}",
          "CardName": "={{$('Shopify').payload.firstName}}{{$('Shopify').payload.lastName}}",
          "BPAddresses": [
            {
              "City": "={{$('Shopify').payload.addresses[].city}}",
              "State": "={{$('Shopify').payload.addresses[].provinceCode}}",
              "BPCode": "={{$payload.CardCode}}",
              "RowNum": "={{$payload.BPAddresses[idx].RowNum}}",
              "Street": "={{$('Shopify').payload.addresses[].address1}}",
              "Country": "={{$('Shopify').payload.addresses[].countryCodeV2}}",
              "ZipCode": "={{$('Shopify').payload.addresses[].zip}}",
              "AddressName": "={{$('Shopify').payload.addresses[].name}}",
              "AddressType": "bo_ShipTo"
            }
          ]
        },
        "action_code": "update_businesspartner",
        "action_name": "Update a businesspartner",
        "current_name": "SAP Business One 2",
        "original_name": "SAP Business One"
      }
    },
    {
      "id": "c8704890-ae44-4836-a084-4b2340201759",
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
          "Phone1": "={{$('Shopify').payload.defaultPhoneNumber.phoneNumber}}",
          "CardCode": "={{split($('Shopify').payload.id, '/')[4]}}",
          "CardName": "={{$('Shopify').payload.firstName}} {{$('Shopify').payload.lastName}}",
          "BPAddresses": [
            {
              "City": "={{$('Shopify').payload.addresses[].city}}",
              "State": "={{$('Shopify').payload.addresses[].provinceCode}}",
              "Street": "={{$('Shopify').payload.addresses[].address1}}",
              "Country": "={{$('Shopify').payload.addresses[].countryCodeV2}}",
              "ZipCode": "={{$('Shopify').payload.addresses[].zip}}",
              "AddressName": "={{$('Shopify').payload.addresses[].name}}",
              "AddressType": "bo_ShipTo"
            }
          ],
          "EmailAddress": "={{$('Shopify').payload.defaultEmailAddress.emailAddress}}"
        },
        "action_code": "create_businesspartner",
        "action_name": "Create a new businesspartner",
        "current_name": "SAP Business One 1",
        "original_name": "SAP Business One"
      }
    }
  ],
  "edges": [
    { "id": "554ec1d8-237e-4dfa-a1c9-03b6605959ab", "source": "2c768c95-b32c-4e0e-96bc-2f91850b50c6", "target": "964b98bb-24e7-4606-a2f4-3e01e7b32275", "sourceHandle": "default", "targetHandle": "default" },
    { "id": "d048edab-c588-4b8d-b973-4a00f3dfab02", "source": "964b98bb-24e7-4606-a2f4-3e01e7b32275", "target": "8192e43a-6d6e-4f97-bc21-4e6d086a3c70", "sourceHandle": "default", "targetHandle": "default" },
    { "id": "caab7dd7-4c7d-40f8-8f96-b3785fb8ff8b", "source": "8192e43a-6d6e-4f97-bc21-4e6d086a3c70", "target": "c8704890-ae44-4836-a084-4b2340201759", "sourceHandle": "false", "targetHandle": "default" },
    { "id": "4dec1c6c-ae5b-4771-ac66-8935c91f77a5", "source": "8192e43a-6d6e-4f97-bc21-4e6d086a3c70", "target": "ddfc3ffe-6efd-4dbc-9f64-91f677fc7a46", "sourceHandle": "true", "targetHandle": "default" }
  ]
}
```

Compare the `false`-branch create node (no `RowNum`, address is brand new)
against the `true`-branch update node (`RowNum` pulled from the search
result) — that's the detail this file exists to show.
