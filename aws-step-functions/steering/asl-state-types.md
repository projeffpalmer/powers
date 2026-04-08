# ASL Structure and State Types (JSONata Mode)

Quick reference for the eight state types in AWS Step Functions. Reference variables-and-data.md for details about the fields available inside each state.
---

## Pass State

Passes input to output, optionally transforming it with JSONata. Useful for injecting or transforming data. Without `Output`, the Pass state copies input to output unchanged.

```json
"SetupAndGreet": {
  "Type": "Pass",
  "Assign": {
    "retryCount": 0,
    "maxRetries": 3,
    "config": "{% $states.input.configuration %}"
  },
  "Output": {
    "greeting": "{% 'Hello, ' & $states.input.name %}",
    "timestamp": "{% $now() %}"
  },
  "Next": "ProcessItem"
}
```

---

## Task State

Executes work via AWS service integrations, activities, or HTTP APIs. Reference service-integrations.md for full details.

### Required Fields
- `Resource`: ARN identifying the task to execute

### Optional Fields
- `Arguments`: Input to the task (replaces JSONPath `Parameters`)
- `Output`: Transform the result
- `Assign`: Store variables from input or result
- `TimeoutSeconds`: Max task duration (default 99999999, accepts JSONata expression)
- `HeartbeatSeconds`: Heartbeat interval (must be < TimeoutSeconds)
- `Retry`: Retry policy array
- `Catch`: Error handler array
- `Credentials`: Cross-account role assumption

---

## Choice State

Uses `Choices` and `Condition` fields with JSONata boolean expressions to implement branching logic.

Key points:
- `Condition` must evaluate to a boolean.
- Each Choice Rule can have its own `Assign` and `Output`.
- If a rule matches, its `Assign`/`Output` are used (not the state-level ones).
- If no rule matches, the state-level `Assign` is evaluated and `Default` is followed.
- `Default` is optional but recommended — without it, `States.NoChoiceMatched` is thrown.
- Choice states cannot be terminal (no `End` field).

### Structure

```json
"RouteOrder": {
  "Type": "Choice",
  "Choices": [
    {
      "Condition": "{% $states.input.orderType = 'express' %}",
      "Next": "ExpressShipping"
    },
    {
      "Condition": "{% $states.input.total > 100 %}",
      "Assign": {
        "discount": "{% $states.input.total * 0.1 %}"
      },
      "Output": {
        "total": "{% $states.input.total * 0.9 %}"
      },
      "Next": "ApplyDiscount"
    },
    {
      "Condition": "{% $states.input.priority >= 5 and $states.input.category = 'urgent' %}",
      "Next": "PriorityQueue"
    }
  ],
  "Default": "StandardProcessing",
  "Assign": {
    "routedDefault": true
  }
}
```

JSONata supports rich boolean logic:

```json
"Condition": "{% $states.input.age >= 18 and $states.input.age <= 65 %}"
"Condition": "{% $states.input.status = 'active' or $states.input.override = true %}"
"Condition": "{% $not($exists($states.input.error)) %}"
"Condition": "{% $contains($states.input.email, '@') %}"
"Condition": "{% $count($states.input.items) > 0 %}"
"Condition": "{% $states.input.score >= $threshold %}"
```

---

## Wait State

Delays execution for a specified duration or until a timestamp.

### Wait with Dynamic Seconds

```json
"DynamicWait": {
  "Type": "Wait",
  "Seconds": "{% $states.input.delaySeconds %}",
  "Next": "Continue"
}
```

### Wait Until Timestamp

```json
"WaitUntilDate": {
  "Type": "Wait",
  "Timestamp": "{% $states.input.scheduledTime %}",
  "Next": "Execute"
}
```

Timestamps must conform to RFC3339 (e.g., `"2026-03-14T01:59:00Z"`).

A Wait state must contain exactly one of `Seconds` or `Timestamp`.

---

## Succeed State

Terminates the state machine (or a Parallel branch / Map iteration) successfully.

```json
"Done": {
  "Type": "Succeed",
  "Output": {
    "status": "completed",
    "processedAt": "{% $now() %}"
  }
}
```

---

## Fail State

Terminates the state machine with an error.

```json
"DynamicFail": {
  "Type": "Fail",
  "Error": "{% $states.input.errorCode %}",
  "Cause": "{% $states.input.errorMessage %}"
}
```
Reference error-handling.md for more information.

---

## Parallel State

Executes multiple branches concurrently. All branches receive the same input.

```json
"LookupCustomerInfo": {
  "Type": "Parallel",
  "Arguments": {
    "customerId": "{% $states.input.customerId %}"
  },
  "Branches": [
    {
      "StartAt": "GetAddress",
      "States": {
        "GetAddress": {
          "Type": "Task",
          "Resource": "arn:aws:states:::lambda:invoke",
          "Arguments": {
            "FunctionName": "arn:aws:lambda:us-east-1:123456789012:function:GetAddress:$LATEST",
            "Payload": "{% $states.input %}"
          },
          "Output": "{% $states.result.Payload %}",
          "End": true
        }
      }
    },
    {
      "StartAt": "GetOrders",
      "States": {
        "GetOrders": {
          "Type": "Task",
          "Resource": "arn:aws:states:::lambda:invoke",
          "Arguments": {
            "FunctionName": "arn:aws:lambda:us-east-1:123456789012:function:GetOrders:$LATEST",
            "Payload": "{% $states.input %}"
          },
          "Output": "{% $states.result.Payload %}",
          "End": true
        }
      }
    }
  ],
  "Assign": {
    "address": "{% $states.result[0] %}",
    "orders": "{% $states.result[1] %}"
  },
  "Output": {
    "address": "{% $states.result[0] %}",
    "orders": "{% $states.result[1] %}"
  },
  "Next": "ProcessResults"
}
```

Key points:
- `Arguments` provides input to each branch's StartAt state (optional, defaults to state input).
- Result is an array with one element per branch, in the same order as `Branches`.
- If any branch fails, the entire Parallel state fails (unless caught).
- States inside branches can only transition to other states within the same branch.
- Branch variables are scoped — branches cannot access each other's variables.
- Use `Output` on terminal states within branches to pass data back to the outer scope.

---

## Map State

Iterates over an array, processing each element (potentially in parallel).

### Basic Map

```json
"ProcessItems": {
  "Type": "Map",
  "Items": "{% $states.input.orders %}",
  "MaxConcurrency": 10,
  "ItemProcessor": {
    "StartAt": "ProcessOrder",
    "States": {
      "ProcessOrder": {
        "Type": "Task",
        "Resource": "arn:aws:states:::lambda:invoke",
        "Arguments": {
          "FunctionName": "arn:aws:lambda:us-east-1:123456789012:function:ProcessOrder:$LATEST",
          "Payload": "{% $states.input %}"
        },
        "Output": "{% $states.result.Payload %}",
        "End": true
      }
    }
  },
  "Output": "{% $states.result %}",
  "Next": "AllDone"
}
```

### Map with ItemSelector

Use `ItemSelector` to reshape each item before processing:

```json
"ProcessItems": {
  "Type": "Map",
  "Items": "{% $states.input.detail.shipped %}",
  "ItemSelector": {
    "parcel": "{% $states.context.Map.Item.Value %}",
    "index": "{% $states.context.Map.Item.Index %}",
    "courier": "{% $states.input.detail.`delivery-partner` %}"
  },
  "MaxConcurrency": 0,
  "ItemProcessor": {
    "StartAt": "Ship",
    "States": {
      "Ship": {
        "Type": "Task",
        "Resource": "arn:aws:states:::lambda:invoke",
        "Arguments": {
          "FunctionName": "arn:aws:lambda:us-east-1:123456789012:function:ShipItem:$LATEST",
          "Payload": "{% $states.input %}"
        },
        "Output": "{% $states.result.Payload %}",
        "End": true
      }
    }
  },
  "Next": "Done"
}
```

### Map Context Variables

Inside `ItemSelector`, you can access:
- `$states.context.Map.Item.Value` — the current array element
- `$states.context.Map.Item.Index` — the zero-based index

### Key Map Fields

| Field | Description |
|-------|-------------|
| `Items` | JSON array or JSONata expression evaluating to an array |
| `ItemProcessor` | State machine to run for each item (has `StartAt` and `States`) |
| `ItemSelector` | Reshape each item before processing |
| `MaxConcurrency` | Max parallel iterations (0 = unlimited, 1 = sequential) |
| `ToleratedFailurePercentage` | 0-100, percentage of items allowed to fail |
| `ToleratedFailureCount` | Number of items allowed to fail |
| `ItemReader` | Read items from an external resource |
| `ItemBatcher` | Batch items into sub-arrays |
| `ResultWriter` | Write results to an external resource |

### Map ProcessorConfig

The `ItemProcessor` can include a `ProcessorConfig` to control execution mode.
- `INLINE` (default) — iterations run within the parent execution. Use for most cases.
- `DISTRIBUTED` — iterations run as child executions. Use for large-scale processing (thousands+ items), items read from S3, or when you need per-iteration execution history.

```json
"ItemProcessor": {
  "ProcessorConfig": {
    "Mode": "INLINE"
  },
  "StartAt": "ProcessOrder",
  "States": { ... }
}
```

### Map Failure Tolerance

```json
"ProcessWithTolerance": {
  "Type": "Map",
  "Items": "{% $states.input.records %}",
  "ToleratedFailurePercentage": 10,
  "ToleratedFailureCount": 5,
  "ItemProcessor": { ... },
  "Next": "Done"
}
```

The Map state fails if either threshold is breached.

---