# Pattern: parallel branches off one trigger

Real, working workflow structure exported from appse ai, trimmed to its
structural fields only (node UI position/size and repeated app-catalog
metadata removed — see `conventions.md` for the full envelope shape those
omitted fields belong to). Read for structure only — never reuse these
literal field values, node names, or app-specific mappings.

**Demonstrates:** independent parallel branches off a single trigger (not
sequential) — one branch updates inventory directly on a stable key, a
second, separate `DecisionNode` (dual condition: exists AND value = 0)
triggers a Microsoft Teams notification. Useful reference for "notify on a
business condition" as a distinct action type, and for multiple independent
branches fanning out from one trigger node.

**Status: approved reference for parallel branching + notification actions.**

```json
{
  "nodes": [
    {
      "id": "fecc3429-f2fe-4528-8d07-3cbd1ca18260",
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
        "properties": { "limit": "2", "since_from": "2026-02-19T10:00:00.000Z" },
        "action_code": "on_items_updated",
        "action_name": "Items updated",
        "current_name": "Dynamics 365 Business Central",
        "original_name": "Dynamics 365 Business Central",
        "next_data_from_template": "={{date_max_by($payload[*],&lastModifiedDateTime) }}"
      }
    },
    {
      "id": "a8b94b3f-cd66-4a3f-bbce-1cd0cd664a6b",
      "type": "DecisionNode",
      "data": {
        "idx": 2,
        "settings": { "disable_case_sensitivity": false },
        "properties": {
          "advance_filter": [
            [
              {
                "operator": { "type": "basic", "operation": "exist" },
                "leftValue": "={{$payload.gtin}}",
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
      "id": "45f7e503-d49b-40cb-b37f-fa21da074178",
      "type": "AppNode",
      "data": {
        "app": { "code": "bigcommerce" },
        "idx": 3,
        "settings": {
          "on_error": "stop",
          "execute_once": false,
          "retry_on_fail": false,
          "always_output_data": false
        },
        "properties": {
          "product_id": "={{$payload.gtin}}",
          "inventory_level": "={{$payload.inventory}}",
          "inventory_tracking": "product"
        },
        "action_code": "update_product_inventory",
        "action_name": "Update product inventory",
        "current_name": "BigCommerce",
        "original_name": "BigCommerce"
      }
    },
    {
      "id": "9fddeda6-7147-4841-a817-c8fde0896b9c",
      "type": "DecisionNode",
      "data": {
        "idx": 4,
        "settings": { "disable_case_sensitivity": false },
        "properties": {
          "advance_filter": [
            [
              {
                "operator": { "type": "basic", "operation": "exist" },
                "leftValue": "={{$payload.gtin}}",
                "rightValue": ""
              },
              {
                "operator": { "type": "number", "operation": "equal" },
                "leftValue": "={{$payload.inventory}}",
                "rightValue": "0"
              }
            ]
          ]
        },
        "current_name": "Decision 1",
        "original_name": "Decision"
      }
    },
    {
      "id": "65c6d4cc-2b29-483e-85f0-fedae772bb2a",
      "type": "AppNode",
      "data": {
        "app": { "code": "microsoftteams" },
        "idx": 5,
        "settings": {
          "on_error": "stop",
          "execute_once": false,
          "retry_on_fail": false,
          "always_output_data": false
        },
        "properties": {
          "body": {
            "content": "=<html, built from $payload.number and a fixed stock-alert message — omitted here, structure only: an HTML string with the item number interpolated twice>",
            "contentType": "html"
          },
          "chat_id": "=48:notes"
        },
        "action_code": "send_message_to_chat",
        "action_name": "Send message to a chat group",
        "current_name": "Microsoft Teams",
        "original_name": "Microsoft Teams"
      }
    }
  ],
  "edges": [
    { "id": "7e0e4c4b-d054-4e73-b2b6-18ea3c9b2093", "source": "fecc3429-f2fe-4528-8d07-3cbd1ca18260", "target": "a8b94b3f-cd66-4a3f-bbce-1cd0cd664a6b", "sourceHandle": "default", "targetHandle": "default" },
    { "id": "a65a8f68-582e-4823-aee7-6cb663470183", "source": "a8b94b3f-cd66-4a3f-bbce-1cd0cd664a6b", "target": "45f7e503-d49b-40cb-b37f-fa21da074178", "sourceHandle": "true", "targetHandle": "default" },
    { "id": "e3241767-955c-4d94-a7bc-82dc3bfa8190", "source": "fecc3429-f2fe-4528-8d07-3cbd1ca18260", "target": "9fddeda6-7147-4841-a817-c8fde0896b9c", "sourceHandle": "default", "targetHandle": "default" },
    { "id": "4736331c-8f1f-49f5-b284-ba700bbb9303", "source": "9fddeda6-7147-4841-a817-c8fde0896b9c", "target": "65c6d4cc-2b29-483e-85f0-fedae772bb2a", "sourceHandle": "true", "targetHandle": "default" }
  ]
}
```

The structural point: **two separate edges leave the trigger node directly**
(to `a8b94b3f...` and to `9fddeda6...`) — that's what makes these branches
parallel rather than sequential. Each `DecisionNode` here is independent;
they don't feed each other. The second Decision shows two conditions
AND-ed in one `advance_filter` inner array (exists AND equals 0).
