# Pattern: create-or-update (customer)

Real, working workflow structure exported from appse ai, trimmed to its
structural fields only (node UI position/size and repeated app-catalog
metadata removed — see `conventions.md` for the full envelope shape those
omitted fields belong to). Read for structure only — never reuse these
literal field values, node names, or app-specific mappings.

**Demonstrates:** the real create-vs-update pattern. Trigger (D365 BC
customer created) → search Magento2 by email → `DecisionNode` "Customers
exist" → **both branches wired**: `false` → `Create a customer`, `true` →
`Update a customer` (with a `customerId` pulled from the search result).

**Status: approved reference for create-vs-update.**

```json
{
  "nodes": [
    {
      "id": "7cc628fe-3ef2-4fac-815f-40696fad8cfb",
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
        "properties": { "limit": "40", "since_from": "2026-02-09T09:00:00.000Z" },
        "action_code": "on_new_customers_created",
        "action_name": "New customers created",
        "current_name": "Dynamics 365 Business Central",
        "original_name": "Dynamics 365 Business Central",
        "next_data_from_template": "={{date_max_by($payload[*],&lastModifiedDateTime) }}"
      }
    },
    {
      "id": "ee681cc0-f960-4efb-bc12-f3a70858a1b1",
      "type": "AppNode",
      "data": {
        "app": { "code": "magento2" },
        "idx": 2,
        "settings": {
          "on_error": "stop",
          "execute_once": false,
          "retry_on_fail": false,
          "always_output_data": false
        },
        "properties": { "email": "={{$payload.email}}" },
        "action_code": "get_customer_by_email",
        "action_name": "Get customer by email address",
        "current_name": "Search customers by email",
        "original_name": "Magento2"
      }
    },
    {
      "id": "401f753f-42bd-4475-a0df-93be26d2b20b",
      "type": "DecisionNode",
      "data": {
        "idx": 3,
        "settings": { "disable_case_sensitivity": false },
        "properties": {
          "advance_filter": [
            [
              {
                "operator": { "type": "string", "operation": "equal" },
                "leftValue": "={{$payload.items[].email}}",
                "rightValue": "={{$('Dynamics 365 Business Central').payload.email}}"
              }
            ]
          ]
        },
        "current_name": "Customers exist",
        "original_name": "Decision"
      }
    },
    {
      "id": "b5fa7308-fd20-4566-9be6-2c76caf150e3",
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
          "customer": {
            "lastname": "={{split($('Dynamics 365 Business Central').payload.displayName, ' ')[1]}}",
            "addresses": [
              {
                "city": "={{$('Dynamics 365 Business Central').payload.city}}",
                "street": [
                  "={{$('Dynamics 365 Business Central').payload.addressLine1}}",
                  "={{$('Dynamics 365 Business Central').payload.addressLine2}}"
                ],
                "lastname": "={{split($('Dynamics 365 Business Central').payload.displayName, ' ')[1]}}",
                "postcode": "={{$('Dynamics 365 Business Central').payload.postalCode}}",
                "firstname": "={{split($('Dynamics 365 Business Central').payload.displayName, ' ')[0]}}",
                "telephone": "={{$('Dynamics 365 Business Central').payload.phoneNumber}}",
                "country_id": "={{$('Dynamics 365 Business Central').payload.country}}",
                "defaultBilling": "Yes",
                "defaultShipping": "Yes"
              }
            ],
            "firstname": "={{split($('Dynamics 365 Business Central').payload.displayName, ' ')[0]}}",
            "website_id": "1"
          },
          "customerId": "={{$payload.items[idx].id}}"
        },
        "action_code": "update_customer",
        "action_name": "Update a customer",
        "current_name": "Update existing customers",
        "original_name": "Magento2"
      }
    },
    {
      "id": "fc4dbc02-1346-40e5-9e12-ce7812608d2e",
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
          "customer": {
            "email": "={{$('Dynamics 365 Business Central').payload.email}}",
            "lastname": "={{split($('Dynamics 365 Business Central').payload.displayName,' ')[1]}}",
            "addresses": [
              {
                "city": "={{$('Dynamics 365 Business Central').payload.city}}",
                "street": [
                  "={{$('Dynamics 365 Business Central').payload.addressLine1}}",
                  "={{$('Dynamics 365 Business Central').payload.addressLine2}}"
                ],
                "lastname": "={{split($('Dynamics 365 Business Central').payload.displayName,' ')[1]}}",
                "postcode": "={{$('Dynamics 365 Business Central').payload.postalCode}}",
                "firstname": "={{split($('Dynamics 365 Business Central').payload.displayName,' ')[0]}}",
                "telephone": "={{$('Dynamics 365 Business Central').payload.phoneNumber}}",
                "country_id": "={{$('Dynamics 365 Business Central').payload.country}}",
                "defaultBilling": "Yes",
                "defaultShipping": "Yes"
              }
            ],
            "firstname": "={{split($('Dynamics 365 Business Central').payload.displayName,' ')[0]}}"
          }
        },
        "action_code": "create_customer",
        "action_name": "Create a customer",
        "current_name": "Create new customers",
        "original_name": "Magento2"
      }
    }
  ],
  "edges": [
    { "id": "d952a989-033c-4ebf-b988-af0e10cdf875", "source": "7cc628fe-3ef2-4fac-815f-40696fad8cfb", "target": "ee681cc0-f960-4efb-bc12-f3a70858a1b1", "sourceHandle": "default", "targetHandle": "default" },
    { "id": "e8d746d2-463a-4084-a3c0-7a7d1e1acf6c", "source": "ee681cc0-f960-4efb-bc12-f3a70858a1b1", "target": "401f753f-42bd-4475-a0df-93be26d2b20b", "sourceHandle": "default", "targetHandle": "default" },
    { "id": "d7dca838-ed05-4f43-8946-e8e4beb53243", "source": "401f753f-42bd-4475-a0df-93be26d2b20b", "target": "fc4dbc02-1346-40e5-9e12-ce7812608d2e", "sourceHandle": "false", "targetHandle": "default" },
    { "id": "9f853075-cdbf-4473-a182-dcefb5a83fab", "source": "401f753f-42bd-4475-a0df-93be26d2b20b", "target": "b5fa7308-fd20-4566-9be6-2c76caf150e3", "sourceHandle": "true", "targetHandle": "default" }
  ]
}
```

Both `DecisionNode` handles are wired here, unlike the dedupe-and-skip
pattern — `false` creates, `true` updates, each to its own action node.
