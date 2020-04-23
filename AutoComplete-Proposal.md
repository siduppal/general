# Proposal for bringing auto-complete functionality to `Input.ChoiceSet` in Adaptive Cards.

## Summary 

When there are thousands of choices in a menu (like assigning a task to someone, when there's thousands of people in your org), you need to have a searchable select menu that dynamically loads data as the user searches (since including 1,000 or more choices statically in the card/data isn't a good option).

![](https://cdn.productboard.com/production/attachments/d9a78e01fb63f8502884f5361982c48d4d913acb491fef0ed623d958d6356bb2/portal_cover/DynamicSearchable.gif)


## Status

This feature is currently **blocked** while we work on a formalized transport contract that allows Adaptive Cards to make an authenticated connection to a backend.

In the interim we can work on a proposal based on a Microsoft Teams feature that enables this functionality in Messaging Extensions. 

Please take a look at the [Teams Search Command](https://docs.microsoft.com/en-us/microsoftteams/platform/messaging-extensions/how-to/search-commands/respond-to-search?tabs=json) for reference.

## Open Issues

- [ ] Need to discuss the combinations of `style` and `isMultiSelect` with this

## Requirements

1. The input can dynamically fetch data from a remote backend as the user types
2. The backend returns data in the correct shape (described below)
3. Single-select only for v1
4. An Action may be invoked when a choice is selected

## Spec/schema

Coming soon

## Example

### Step 1: Card Payload

The following payload is a normal `Input.ChoiceSet` with a property `autoComplete` that describes how to query the bot to dynamically retreive values for the choice-set. 

```json
{
    "type": "Input.ChoiceSet",
    "id": "selectedUser",
    "choices": [
        { "title": "Matt", "value": "1" },
	{ "title": "David", "value": "2" }
    ],
    "value": "2",
    "autoComplete": {
        "commandId": "searchUserCmd",
        "pageSize": 25,
        "initialRun": false,
        "parameters": [
            {
                "name": "searchKeywords",
                "value": "{selectedUser.value}"
            }
        ]
    }
}
```
#### Properties of `autoComplete`
| Property name | Purpose |
| --- | --- |
| commandId | The name of the command to query the bot |
| pageSize | The number of elements to fetch in a single query |
| initialRun | If `initialRun` is true, bot receives the query as soon as the choice-set has focus |
| parameters | Array of paramters to query the bot. Each parameter object contains the parameter name, along with the value |

### Step 2: Fetch the data as the user types

As the user types, an HTTP request is sent to the Bot in the format below. 

```json
{
  "type": "invoke",
  "name": "autoComplete/query",
  "value": {
    "commandId": "searchUserCmd",
    "parameters": [
      {
        "name": "searchKeywords",
        "value": "<VALUE-AS-TYPED-BY-USER>"
      }
    ],
    "queryOptions": {
      "skip": 0,
      "count": 25
    }
  }
}
```

As the user scrolls-down in the choice-set, additional queries are made to the bot while incrementing the value of `skip`.

If `initialRun` was set to `true` for the command, then the default-query contains the parameter `initialRun` set to true as shown below.

```json
{
  "type": "invoke",
  "name": "autoComplete/query",
  "value": {
    "commandId": "searchUserCmd",
    "parameters": [
      {
        "name": "initialRun",
        "value": "true"
      }
    ],
    "queryOptions": {
      "skip": 0,
      "count": 25
    }
  }
}
```

### Step 3: Backend bot responds with the matching results

The backend sees that this is a `autoComplete/query`, and needs to return a set of items based on the query. It can inspect the `commandId` and `parameters` values to know what type of data to return, and what the user has currently typed.

E.g., if the user had typed "Ma" it would return something like:

```json
{
    "autoCompleteResponse": {
        "type": "result",
        "values": [
            { "title": "Matt", "id": "1" },
            { "title": "Mark", "id": "2" }
            { "title": "Mack", "id": "3" }
            { "title": "May", "id": "4" }
        ]
    }
}
```

Each `autoCompleteResponse` has a type. The following types are supported.

| Type | Purpose |
| --- | --- |
| result | Results of the query |
| auth | Asks the user to authenticate |
| config | Asks the user to setup the bot |
| message | Displays a plain text message |

#### Responding with sign-in action

To prompt an unauthenticated user to sign-in, respond with an `Action.OpenUrl` action as shown below.

```json
{
    "autoCompleteResponse": {
        "type": "auth",
        "values": [
            { "type": "Action.OpenUrl", "url": "https://example.com/auth", "title": "Please sign-in to continue" }
        ]
    }
}
```

#### Responding with config action

To prompt the user to configure the bot so the query can be performed, respond with an `Action.OpenUrl` action as shown below.

```json
{
    "autoCompleteResponse": {
        "type": "config",
        "values": [
            { "type": "Action.OpenUrl", "url": "https://example.com/config" }
        ]
    }
}
```

#### Responding with a message

To show a message to the user, respond with a `TextBlock` as shown below.

```json
{
    "autoCompleteResponse": {
        "type": "message",
        "values": [
            { "type": "TextBlock", "text": "No matches found" }
        ]
    }
}
```


### Step 4: Display the results 

The Input `choices` property is set to the array returned by the backend.


## Step 5 (optional): Trigger an action when a choice is selected

To allow a card refresh when an item is selected, we will include a property (`selectAction`) to specify an action to be invoked.

```json
{
    "type": "Input.ChoiceSet",
    "id": "selectedUser",
    "choices": [
        { "title": "Matt", "value": "1" },
	{ "title": "David", "value": "2" }
    ],
    "value": "2",
    "autoComplete": {
        "commandId": "searchUserCmd",
        "pageSize": 25,
        "paramName": "searchKeywords"
    },
    "selectAction": {
        "type": "Action.Submit",
        "id": "userSelected"
    }
}
```

When a user selects a choice (with keyboard or mouse) the action should be invokved, allowing the card to be refreshed.
