---
title: Merging type definitions in Apollo Server
date: 2020-01-07
categories:
---

As you finished the hello world lessons of GraphQL using the Apollo Toolkit and want to do some serious stuff with this amazing piece of technology, you will find yourself struggling with organizing your code so that it can be more modular. Nobody likes a file with 1000 line of code.

Same happened with me when I was trying to do a POC for my current organization about putting a GraphQL wrapper over our existing RESTish APIs.

I think it is a nice quick article to write about how I managed to organize my code.

When you look at the documentation of GraphQL Apollo Server, you will see in most if not all examples where they write the type definition in one file such as `schema.js`.

However with some experiments I managed to settle with the following structure.

```
.
└── src
    ├── schema
    │   ├── user.ts
    │   ├── post.ts
    │   └── index.ts
    └── index.ts
```

Once you have organized your code this manner the next thing is to define your specific type definitions in their respective files. So for example I can write all my `Queries`, `Mutations`, `Types`, `Inputs` for User entity in `user.ts` file.

```typescript
import { gql } from "apollo-server-express";

export const typeDefs = gql`
	type User {
		id: Int
		name: String
		active: Boolean
	}

	type Query {
		getAllUsers(offset: Int, limit: Int): [User]! @cacheControl(maxAge: 60)
	}
`;
```

And same goes for `post.ts`

```typescript
import { gql } from "apollo-server-express";

export const typeDefs = gql`
	type Post {
		id: Int
		title: String
		body: String
		created: String
	}

	type Mutation {
		createPost(title: String!, body: String!, created: String!): Post!
	}

	type Subscription {
		postCreated: Post!
	}
`;
```

Now in order to combine these type definitions into a single schema we have to install the following package

```
npm install merge-graphql-schemas
```

In order to export all the schema it is better to merge them in the `index.ts` file within the `schema` directory so that we can simply export the merged definiations for our Apollo Server

```typescript
import { typeDefs as userSchema } from "./user";
import { typeDefs as postSchema } from "./post";
import { mergeTypes } from "merge-graphql-schemas";

const types = [userSchema, postSchema];

export const typeDefs = mergeTypes(types, { all: true });
```

And thats it. All you need to do is to import `typeDefs` from this file in the Apollo Server creation and use it in the respective parameter.
