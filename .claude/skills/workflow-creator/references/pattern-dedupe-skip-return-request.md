# Pattern: dedupe-and-skip

Real, working workflow structure exported from appse ai, trimmed to its
structural fields only (node UI position/size and repeated app-catalog
metadata removed — see `conventions.md` for the full envelope shape those
omitted fields belong to). Read for structure only — never reuse these
literal field values, node names, or app-specific mappings.

**Demonstrates:** dedupe via search-before-write, skip silently if found.
Trigger (WooCommerce order cancelled) → search SAP for an existing return
request (stable key: `U_WEBORDID`) → `FilterNode` → `DecisionNode` on
`DocEntry exist` → **only the `false` branch has an outgoing edge** (create).
The `true` branch (already exists) is a dead end — no action taken.

**Status: approved reference for dedupe-and-skip.**

```json
{
  "nodes": [
    {
      "id": "2f72799b-0e09-4c94-8f8e-6c27b2114ddc",
      "type": "AppTriggerNode",
      "data": {
        "app": { "code": "woocommerce" },
        "idx": 1,
        "cron": "*/3 * * * *",
        "settings": {
          "on_error": "stop",
          "execute_once": false,
          "retry_on_fail": false,
          "always_output_data": false
        },
        "properties": {
          "limit": "10",
          "status": "cancelled",
          "since_from": "2026-03-25T05:34:00.000Z"
        },
        "action_code": "on_orders_updated",
        "action_name": "Orders updated",
        "current_name": "WooCommerce",
        "original_name": "WooCommerce",
        "next_data_from_template": "={{date_max_by($payload[*],&date_modified) }}"
      }
    },
    {
      "id": "f3a0cf34-a1f8-40a2-ba3e-7315910fdb58",
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
        "properties": {
          "$top": "10",
          "$filter": "=U_WEBORDID eq '{{$payload.id}}' and NumAtCard eq 'U_{{$payload.id}}'",
          "objectName": "DeliveryNotes"
        },
        "action_code": "get_filtered_records",
        "action_name": "Search Records",
        "current_name": "SAP Business One",
        "original_name": "SAP Business One"
      }
    },
    {
      "id": "725c0441-d797-482d-8235-9f1044a7dcf3",
      "type": "FilterNode",
      "data": {
        "idx": 3,
        "settings": { "disable_case_sensitivity": false },
        "properties": {
          "advance_filter": [
            [
              {
                "operator": { "type": "basic", "operation": "exist" },
                "leftValue": "={{$payload.DocEntry}}",
                "rightValue": ""
              },
              {
                "operator": { "type": "string", "operation": "equal" },
                "leftValue": "={{$payload.DocumentStatus}}",
                "rightValue": "bost_Open"
              }
            ]
          ]
        },
        "current_name": "Filter",
        "original_name": "Filter"
      }
    },
    {
      "id": "4ff138b1-6cf3-4ec2-bafc-7304d11f71af",
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
          "$top": "10",
          "$filter": "=U_WEBORDID eq '{{$('WooCommerce').payload.id}}' and NumAtCard eq 'U_{{$('WooCommerce').payload.id}}'",
          "objectName": "ReturnRequest"
        },
        "action_code": "get_filtered_records",
        "action_name": "Search Records",
        "current_name": "SAP Business One 2",
        "original_name": "SAP Business One"
      }
    },
    {
      "id": "611e486b-2789-4deb-be74-31afd6546b08",
      "type": "DecisionNode",
      "data": {
        "idx": 5,
        "settings": { "disable_case_sensitivity": false },
        "properties": {
          "advance_filter": [
            [
              {
                "operator": { "type": "basic", "operation": "exist" },
                "leftValue": "={{$payload.DocEntry}}",
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
      "id": "409fa0c1-e63b-4cc6-af18-0ebc02a5f76c",
      "type": "AppNode",
      "data": {
        "app": { "code": "sapbusinessone" },
        "idx": 6,
        "settings": {
          "on_error": "stop",
          "execute_once": false,
          "retry_on_fail": false,
          "always_output_data": false
        },
        "properties": {
          "DocDate": "={{$('SAP Business One').payload.DocDate}}",
          "CardCode": "={{$('SAP Business One').payload.CardCode}}",
          "NumAtCard": "={{$('SAP Business One').payload.NumAtCard}}",
          "DocDueDate": "={{$('SAP Business One').payload.DocDueDate}}",
          "DocumentLines": [
            {
              "TaxCode": "={{$('SAP Business One').payload.DocumentLines[].TaxCode}}",
              "BaseLine": "={{$('SAP Business One').payload.DocumentLines[].BaseLine}}",
              "BaseType": "15",
              "ItemCode": "={{$('SAP Business One').payload.DocumentLines[].ItemCode}}",
              "Quantity": "={{$('SAP Business One').payload.DocumentLines[].Quantity}}",
              "BaseEntry": "={{$('SAP Business One').payload.DocEntry}}",
              "UnitPrice": "={{$('SAP Business One').payload.DocumentLines[].Price}}"
            }
          ]
        },
        "action_code": "create_return_request",
        "action_name": "Create Return Request",
        "current_name": "SAP Business One 1",
        "original_name": "SAP Business One"
      }
    }
  ],
  "edges": [
    { "id": "7ca5b359-333a-46b1-bd0b-cfc3b050cc42", "source": "2f72799b-0e09-4c94-8f8e-6c27b2114ddc", "target": "f3a0cf34-a1f8-40a2-ba3e-7315910fdb58", "sourceHandle": "default", "targetHandle": "default" },
    { "id": "b8000edb-6b90-4ace-ab82-2f398d46bc54", "source": "f3a0cf34-a1f8-40a2-ba3e-7315910fdb58", "target": "725c0441-d797-482d-8235-9f1044a7dcf3", "sourceHandle": "default", "targetHandle": "default" },
    { "id": "40d8e0d0-f88a-4d12-a54b-4b2676062a25", "source": "725c0441-d797-482d-8235-9f1044a7dcf3", "target": "4ff138b1-6cf3-4ec2-bafc-7304d11f71af", "sourceHandle": "default", "targetHandle": "default" },
    { "id": "6c227c7b-86be-4b4c-9c88-5cdc10be2021", "source": "4ff138b1-6cf3-4ec2-bafc-7304d11f71af", "target": "611e486b-2789-4deb-be74-31afd6546b08", "sourceHandle": "default", "targetHandle": "default" },
    { "id": "ad351ecc-9b7c-4567-8590-52589988e2a1", "source": "611e486b-2789-4deb-be74-31afd6546b08", "target": "409fa0c1-e63b-4cc6-af18-0ebc02a5f76c", "sourceHandle": "false", "targetHandle": "default" }
  ]
}
```

Note the last edge: the `DecisionNode`'s `false` handle is the only one wired
to a next node. The `true` branch (return request already exists) has no
outgoing edge at all — that silence *is* "skip."
