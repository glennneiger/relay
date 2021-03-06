---
id: mutations
title: Mutations
layout: docs
category: Relay Modern
permalink: docs/mutations.html
next: subscriptions
---

Relay exposes the following APIs to perform mutations.

```javascript
const {commitMutation} = require('react-relay');

type Variables = {[name: string]: any};

commitMutation(
  environment: Environment,
  config: {
    mutation: GraphQLTaggedNode,
    variables: Variables,
    onCompleted?: ?(response: ?Object, errors: ?[Error]) => void,
    onError?: ?(error: Error) => void,
    optimisticResponse?: Object,
    optimisticUpdater?: ?(store: RecordSourceSelectorProxy) => void,
    updater?: ?(store: RecordSourceSelectorProxy, data: SelectorData) => void,
    configs?: Array<RelayMutationConfig>,
  },
);
```
First, let's take a look at the `environment` input. To perform the mutation on the correct `environment` with the relevant data, it's a good idea to use the `environment` used to render the components. It's accessible at `this.props.relay.environment` from the component.

Now let's take a closer look at the `config`:

* `mutation`: the `graphql` tagged mutation query.
* `variables`: an object that contains the variables needed for the mutation.
* `onCompleted`: a callback function executed with the 'raw' response and errors from the server after the in-memory Relay store is updated with the `updater`.
* `onError`: a callback function executed when Relay encounters an error.
* `optimisticResponse`: an object conforming to the mutation's response type definition. If provided, the optimistic response will be normalized to the proxy store before `optimisticUpdater` is executed. We suggest you provide an `optimisticResponse` for two benefits:
 * Like `updater`, there is no need to provide `optimisticUpdater` for simple mutations (field change).
 * For more complicated mutations, `optimisticUpdater` and `updater` can be the same function.
* `optimisticUpdater`: a function that takes in a proxy of the in-memory Relay store. In this function, the client defines 'how to' update the store through the proxy in an imperative way.
* `updater`: a function that updates the in-memory Relay store based on the **real** server response. When the server response comes back, Relay first reverts any changes introduced by `optimisticUpdater` or `optimisticResponse` and then applies the `updater` to the store.
* `configs`:  an array containing the different optimisticUpdater/updater configurations. It provides a convenient way to specify the `updater` behavior.

## Example

In a simple mutation, you only need `mutation` and `variables`:

```javascript
const {
  commitMutation,
  graphql,
} = require('react-relay');

const mutation = graphql`
  mutation MarkReadNotificationMutation(
    $input: MarkReadNotificationData!
  ) {
    markReadNotification(data: $input) {
      notification {
        seenState
      }
    }
  }
`;

function markNotificationAsRead(environment, source, storyID) {
  const variables = {
    input: {
      source,
      storyID,
    },
  };

  commitMutation(
    environment,
    {
      mutation,
      variables,
      onCompleted: (response, errors) => {
        console.log('Response received from server.')
      },
      onError: err => console.error(err),
    },
  );
}
```

# Updating the client optimistically

To improve perceived responsiveness, you may wish to perform an "optimistic update", in which the client immediately updates to reflect the anticipated new value even before the response from the server has come back. We do this by providing an `optimisticResponse` and adding it to the `config` that we pass into `commitMutation`:

```javascript
const optimisticResponse = {
  markReadNotification: {
    notification: {
      seenState: SEEN,
    },
  },
};

commitMutation(
  environment,
  {
    mutation,
    optimisticResponse,
    variables,
  },
);
```

# Configs

We can give Relay instructions in the form of a config array on how to use the response from each mutation to update the client-side store. We do this by configuring the mutation with one or more of the following mutation types:

## NODE_DELETE
Given a deletedIDFieldName, Relay will remove the node(s) from the connection.

### Arguments
* `deletedIDFieldName: string`: The field name in the response that contains the DataID of the deleted node

### Example
```javascript
const mutation = graphql`
  mutation DestroyShipMutation($input: DestroyShipData!) {
    destroyShip(input: $input) {
      destroyedShipId
      faction {
        ships {
          id
        }
      }
    }
  }
`;

const configs = [{
  type: 'NODE_DELETE',
  deletedIDFieldName: 'destroyedShipId',
}];
```

## RANGE_ADD
Given a parent, information about the connection, and the name of the newly created edge in the response payload Relay will add the node to the store and attach it to the connection according to the range behavior(s) specified in the connectionInfo.

### Arguments
* `parentID: string`: The DataID of the parent node that contains the
connection.
* `connectionInfo: Array<{key: string, filters?: Variables, rangeBehavior:
string}>`: An array of objects containing a connection key, an object
containing optional filters, and a range behavior depending on what behavior we expect (append, prepend, or ignore).
  * `filters`: An object containing GraphQL calls e.g. `const filters = {'orderby': 'chronological'};`.
* `edgeName: string`: The field name in the response that represents the newly created edge

### Example
```javascript
const mutation = graphql`
  mutation AddShipMutation($input: AddShipData!) {
    addShip(input: $input) {
      faction {
        ships {
          id
        }
      }
      newShipEdge
    }
  }
`;

const configs = [{
  type: 'RANGE_ADD',
  parentID: 'shipId',
  connectionInfo: [{
    key: 'AddShip_ships',
    rangeBehavior: 'append',
  }],
  edgeName: 'newShipEdge',
}];
```

## RANGE_DELETE
Given a parent, connectionKeys, one or more DataIDs in the response payload, and
a path between the parent and the connection, Relay will remove the node(s)
from the connection but leave the associated record(s) in the store.

### Arguments
* `parentID: string`: The DataID of the parent node that contains the
connection.
* `connectionKeys: Array<{key: string, filters?: Variables}>`: An array of
objects containing a connection key and optionally filters.
  * `filters`: An object containing GraphQL calls e.g. `const filters = {'orderby': 'chronological'};`.
* `pathToConnection: Array<string>`: An array containing the field names between the parent and the connection, including the parent and the connection.
* `deletedIDFieldName: string | Array<string>`: The field name in the response that contains the DataID of the removed node, or the path to the node removed from the connection

### Example
```javascript
const mutation = graphql`
  mutation RemoveTagsMutation($input: RemoveTagsData!) {
    removeTags(input: $input) {
      todo {
        tags {
          id
        }
      }
      removedTagId
    }
  }
`;

const configs = [{
  type: 'RANGE_DELETE',
  parentID: 'todoId',
  connectionKeys: [{
    key: RemoveTags_tags,
  }],
  pathToConnection: ['todo', 'tags'],
  deletedIDFieldName: removedTagId
}];
```

For examples of more complex optimistic updates, including adding and removing from a list, see the [Relay Modern Todo example app](https://github.com/relayjs/relay-examples/tree/master/todo).

# Updating the store programatically (advanced)

The Relay store can be mutated programatically in advanced edge cases when optimistic updates need more granular control. The following API methods are useful for mutating your connections and fragments.

## RelayRecordStore
### getSource(): [RecordSource](https://github.com/facebook/relay/blob/d0310d69012bba615dacf614319bcf47ee2a0f3f/packages/relay-runtime/ARCHITECTURE.md)
Returns a read-only view of the store's internal RecordSource that holds all records.

### getRootField(fieldName: string): ?RecordProxy
Returns a proxy class for manipulating records from a record source, for example a query, mutation, or the store.


## RelayRecordProxy
### getDataID(): [DataID](https://github.com/facebook/relay/blob/d0310d69012bba615dacf614319bcf47ee2a0f3f/packages/relay-runtime/ARCHITECTURE.md)
Returns the globally unique identifier string for a record.

### getType(): RelayQLType
Returns the GraphQL type name for a given record.

### getValue(name: string, args?: ?Variables): mixed
Reads the value of an attribute on a record by the field name and an object representing pre-defined argument values.

### setValue(value: mixed, name: string, args?: ?Variables): RecordProxy
Updates the value of a mutable record's attribute given by the field name and an object representing pre-defined argument values.

### getLinkedRecord(name: string, args?: ?Variables): ?RecordProxy
### getLinkedRecords(name: string, args?: ?Variables): ?Array<?RecordProxy>
Retrieves record(s) associated with the given record, transversing the source by field name and an object representing pre-defined argument values.

### setLinkedRecord(record: RecordProxy, name: string, args?: ?Variables): RecordProxy
### setLinkedRecords(records: Array<?RecordProxy>, name: string, args?: ?Variables ): RecordProxy
Updates the records associated with a mutable record, transversing the source by field name and an object representing pre-defined argument values.

### getOrCreateLinkedRecord(name: string, typeName: string, args?: ?Variables ): RecordProxy
Finds or creates a single record associated with a mutable record.
This is a shortcut to `RelayRecordProxy.getLinkedRecord` with `RelayRecordProxy.setLinkedRecord` should the associated record be non-existant.

## Advanced Mutation Example

```javascript
const sharedUpdater = (source, todoItem) => {
    const sourceRecord = source.getRootField('todoList');

    const todoItems = sourceRecord.getLinkedRecords('todoItems');
    if (todoItems) {
      sourceRecord.setLinkedRecords(todoItems.concat(todoItem), 'todoItems');
    }
};

const variables = {
  todoItem: {
    task: 'Finish this example!',
    dueDate: null,
  }
}

commitMutation(store, {
  mutation,
  variables,
  updater: (store) => {
    const mutationRoot = store.getRootField('addTodoItem');
    const todoItem = mutationRoot.getLinkedRecord('todoItem');
    sharedUpdater(store, todoItem);
  },
  optimisticUpdater: (store) => {
    const todoItem = mutationRoot.getLinkedRecord('todoItem');
    sharedUpdater(store, variables.todoItem);
  }
});
```
