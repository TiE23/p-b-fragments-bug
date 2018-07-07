# Prisma-Binding(?) Fragments Bug Reproducer
Made with graphql-boilerplate node-graphql-server (advanced)
https://github.com/graphql-boilerplates/node-graphql-server/tree/master/advanced

This is a project to reproduce a fragments issue I'm having as of 2018-07-07.

## What did I do?
What I did is I defined a new type in the `schema.graphql` named `PrivateUser`, where it has the user's email address and I removed the user's email address from the original type `User`. This is in order to hide a user's email address from being accessed by random people in methods such as looking at a `Post` and looking at the `Author` and gaining instant access to the user's email address.

Nothing complicated. Just a simple and common thing.

Look at the git logs to see my change.

## Steps to get started
* Clone this repo.
* Install modules with `npm install`.
* Start the server
    * You can just run the server with `npm run start` then point your Playground at http://localhost:4000
    * You can run the server and a browser Playground with `npm run dev`

## Let's repro this!
### 1 - Login
Log in the example user:
```
mutation login {
  login(
    email: "developer@example.com",
    password: "nooneknows"
  ) {
    token
    user {
      id
    }
  }
}
```

Copy the token and add the proper header to your playground:
```
{
  "Authorization": "Bearer TOKEN_HERE"
}
```

### 2 - me query
Run the basic me() query, which returns PrivateUser.
```
query me {
  me {
    id
    name
    email
  }
}
```
This works.

### 3 - Attempt to use a Fragment
Now, let's define a fragment and try to use it.
```
query me {
  me {
    ...PrivateUserFragment
  }
}

fragment PrivateUserFragment on PrivateUser {
  id
  name
  email
}
```

**ERROR** output:
```
{
  "data": {
    "me": null
  },
  "errors": [
    {
      "message": "Field 'user' of type 'User' must have a sub selection. (line 2, column 3):\n  user(where: $_v0_where)\n  ^",
      "locations": [],
      "path": [
        "user"
      ]
    }
  ]
}
```

### 4 - Sanity check
Let's make sure that fragments are even working. Let's show off a version of me() query working with fragments so long as the type is `User`:
```
query me2 {
  me2 {
    ...UserFragment
  }
}

fragment UserFragment on User {
  id
  name
}
```

This totally works.

### 5 - Causes?
It is my theory that perhaps the binding will not match the fragment because the type's name `PrivateUser` does not match Prisma's query's type name `User`?

#### Additional check
No surprise that also fails to work in a more complicated situation where I attempt to use the Fragment on mutation Login that returns a `AuthPayload` type. So it's not just some weird edge-case with how `info` argument works in `src/resolvers/AuthPayload.js` as I personally originally came across the issue.
```
mutation login {
  login(
    email: "developer@example.com",
    password: "nooneknows"
  ) {
    token
    user {
      ...PrivateUserFragment
    }
  }
}

fragment PrivateUserFragment on PrivateUser {
  id
  name
  email
}
```
**Error** output:
```
{
  "data": null,
  "errors": [
    {
      "message": "Field 'user' of type 'User' must have a sub selection. (line 2, column 3):\n  user(where: $_v0_where)\n  ^",
      "locations": [],
      "path": [
        "user"
      ]
    }
  ]
}
```
