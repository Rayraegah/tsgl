# TSGQL

A utility to enhance the developer experience of writing GraphQL in TypeScript.

## Installlation

```bash
npm install --save tsgql

```

## Usage

Defines GraphQL-like JS Object:

```typescript
import { params, types } from 'tsgql'

const getUserQuery = {
    user: params(
        { id: 1 },
        {
            id: types.number,
            name: types.string,
            bankAccount: {
                id: types.number,
                branch: types.optional.string,
            },
        },
    ),
}
```

`types` helper defines types in the result, and the `params` helper defines the parameters.

Converts the JS Object to GraphQL (string):

```typescript
import { query } from 'tsgql'

const gqlString = query('getUser', getUserQuery)

console.log(gqlString)
// =>
//   query getUser {
//     user(id: 1) {
//       id
//       name
//       bankAccount {
//         id
//         branch
//       }
//     }
//   }
```

Executes the GraphQL:

```typescript
import { executeGraphql } from 'some-graphql-request-library'

// We would like to type this!
const result: typeof getUser = await executeGraphql(gqlString)

// As we cast `result` to `typeof getUser`,
// Now, `result` type looks like this:
// interface result {
//   user: {
//     id: number
//     name: string
//     bankAccount: {
//       id: number
//       branch?: string
//     }
//   }
// }
```

## Features

Currently `tsgql` can convert these GraphQL features:

-   Operations
    -   Query
    -   Mutation
    -   Subscription
-   Inputs
    -   Variables
    -   Parameters
-   Data structures
    -   Nested object query
    -   Array query
-   Scalar types
    -   `number`
    -   `string`
    -   `boolean`
    -   Enum
    -   Constant
    -   Custom type
    -   Optional types, e.g.) `number | undefined`
-   Fragments
-   Inline Fragments

## Examples

### Basic Query

```graphql
query getUser {
    user {
        id
        name
        isActive
    }
}
```

```typescript
import { query, types } from 'tsgql'

query('getUser', {
    user: {
        id: types.number,
        name: types.string,
        isActive: types.boolean,
    },
})
```

### Without Query Name

```graphql
query {
    user {
        id
        name
        isActive
    }
}
```

```typescript
import { query, types } from 'tsgql'

query({
    user: {
        id: types.number,
        name: types.string,
        isActive: types.boolean,
    },
})
```

### Basic Mutation

```graphql
mutation updateUserMutation($input: UserInput!) {
    updateUser(input: $input) {
        id
        name
    }
}
```

```typescript
import { mutation, params } from 'tsgql'

mutation('updateUserMutation', params({ $input: 'UserInput!' }, {
  updateUser: params({ input: '$input' }, {
    id: types.number,
    name: types.string,
  }),
})
```

### Nested Query

```graphql
query getUser {
    user {
        id
        name
        parent {
            id
            name
            grandParent {
                id
                name
                children {
                    id
                    name
                }
            }
        }
    }
}
```

```typescript
import { query, types } from 'tsgql'

query('getUser', {
    user: {
        id: types.number,
        name: types.string,
        parent: {
            id: types.number,
            name: types.string,
            grandParent: {
                id: types.number,
                name: types.string,
                children: {
                    id: types.number,
                    name: types.string,
                },
            },
        },
    },
})
```

### Array Field

```graphql
query getUsers {
  users(status: 'active') {
    id
    name
  }
}
```

```typescript
import { params, query, types } from 'tsgql'

query('users', {
    users: params({ status: 'active' }, [
        {
            id: types.number,
            name: types.string,
        },
    ]),
})
```

### Optional Field

```typescript
import { optional, query, types } from 'tsgql'

query('getUser', {
  user: {
    id: types.number,
    name: types.optional.string, // <-- user.name is `string | undefined`
    bankAccount: optional({      // <-- user.bankAccount is `{ id: number } | undefined`
      id: types.number,
    }),
  },
}
```

### Constant field

```graphql
query getUser {
    user {
        id
        name
        __typename # <-- Always `User`
    }
}
```

```typescript
import { query, types } from 'tsgql'

query('getUser', {
    user: {
        id: types.number,
        name: types.string,
        __typename: types.constant('User'),
    },
})
```

### Enum field

```graphql
query getUser {
    user {
        id
        name
        type # <-- `Student` or `Teacher`
    }
}
```

```typescript
import { query, types } from 'tsgql'

enum UserType {
    'Student',
    'Teacher',
}

query('getUser', {
    user: {
        id: types.number,
        name: types.string,
        type: types.oneOf(UserType),
    },
})
```

### Multiple Queries

```graphql
query getFatherAndMother {
    father {
        id
        name
    }
    mother {
        id
        name
    }
}
```

```typescript
import { query, types } from 'tsgql'

query('getFatherAndMother', {
    father: {
        id: types.number,
        name: types.string,
    },
    mother: {
        id: types.number,
        name: types.number,
    },
})
```

### Query Alias

via a dynamic property.

```graphql
query getMaleUser {
    maleUser: user {
        id
        name
    }
}
```

```typescript
import { alias, query, types } from 'tsgql'

query('getMaleUser', {
  [alias('maleUser', 'user')]: {
    id: types.number,
    name: types.string,
  },
}
```

### Standard fragments

```graphql
query {
    user(id: 1) {
        ...userFragment
    }
    maleUsers: users(sex: MALE) {
        ...userFragment
    }
}

fragment userFragment on User {
    id
    name
    bankAccount {
        ...bankAccountFragment
    }
}

fragment bankAccountFragment on BankAccount {
    id
    branch
}
```

```typescript
import { alias, fragment, params, query } from 'tsgql'

const bankAccountFragment = fragment('bankAccountFragment', 'BankAccount', {
  id: types.number,
  branch: types.string,
})

const userFragment = fragment('userFragment', 'User', {
  id: types.number,
  name: types.string,
  bankAccount: {
    ...bankAccountFragment,
  },
})

query({
  user: params({ id: 1 }, {
    ...userFragment,
  }),
  [alias('maleUsers', 'users')]: params({ sex: 'MALE' }, {
    ...userFragment,
  }),
}
```

### Inline Fragment

```graphql
query getHeroForEpisode {
    hero {
        id
        ... on Droid {
            primaryFunction
        }
        ... on Human {
            height
        }
    }
}
```

```typescript
import { on, query, types } from 'tsgql'

query('getHeroForEpisode', {
    hero: {
        id: types.number,
        ...on('Droid', {
            primaryFunction: types.string,
        }),
        ...on('Human', {
            height: types.number,
        }),
    },
})
```

#### discriminated union pattern

```graphql
query getHeroForEpisode {
    hero {
        id
        ... on Droid {
            kind
            primaryFunction
        }
        ... on Human {
            kind
            height
        }
    }
}
```

```typescript
import { onUnion, query, types } from 'tsgql'

query('getHeroForEpisode', {
    hero: {
        id: types.number,
        ...onUnion({
            Droid: {
                kind: types.constant('Droid'),
                primaryFunction: types.string,
            },
            Human: {
                kind: types.constant('Human'),
                height: types.number,
            },
        }),
    },
})
```

returns a type of `A | B`

```typescript
const droidOrHuman = queryResult.hero
if (droidOrHuman.kind === 'Droid') {
    const droid = droidOrHuman
    // ... handle droid
} else if (droidOrHument.kind === 'Human') {
    const human = droidOrHuman
    // ... handle human
}
```

## Usage w/ React Native

```typescript
import 'babel-polyfill' // polyfill `Symbol` and `Map`
import * as React from 'react'
import { View, Text } from 'react-native'
import { query, types } from 'tsgql'

const queryString = query({
    getUser: {
        user: {
            id: types.number,
        },
    },
})

export class App extends React.Component<{}> {
    render() {
        return (
            <View>
                <Text>{queryString}</Text>
            </View>
        )
    }
}
```
