---
title: Composable networking for GraphQL
sidebar_title: Introduction
description: Apollo Link is a standard interface for modifying control flow of GraphQL requests and fetching GraphQL results.
---

This is the official guide for getting started with Apollo Link in your application. Apollo Link is a simple yet powerful way to describe how you want to get the result of a GraphQL operation, and what you want to do with the results. You've probably come across "middleware" that might transform a request and its result: Apollo Link is an abstraction that's meant to solve similar problems in a much more flexible and elegant way. 

You can use Apollo Link with Apollo Client, `graphql-tools` schema stitching, GraphiQL, and even as a standalone client, allowing you to reuse the same authorization, error handling, and control flow across all of your GraphQL fetching.

<h2 id="introduction">Introduction</h2>

In a few words, Apollo Links are chainable "units" that you can snap together to define how each GraphQL request is handled by your GraphQL client. When you fire a GraphQL request, each Link's functionality is applied one after another. This allows you to control the request lifecycle in a way that makes sense for your application. For example, Links can provide [retrying](https://github.com/apollographql/apollo-link/tree/master/packages/apollo-link-retry), [polling](https://github.com/apollographql/apollo-link/tree/master/packages/apollo-link-polling), [batching](https://github.com/apollographql/apollo-link/tree/master/packages/apollo-link-batch), and more!  

If you're new to Apollo Link, you should read the [concepts guide](https://www.apollographql.com/docs/link/overview.html) which explains the motivation behind the package and its different pieces. The original [blog post](https://dev-blog.apollodata.com/apollo-link-the-modular-graphql-network-stack-3b6d5fcf9244) for the project is also a great first resource. 

<h2 id="installation">Installation</h2>

```bash
npm install apollo-link
```

Apollo Link has two main exports, the `ApolloLink` interface and the `execute` function. The `ApolloLink` interface is used to create custom links, compose multiple links together, and can be extended to support more powerful use cases. The `execute` function allows you to create a request with a link and an operation. For a deeper dive on how to use links in your application, check out our Apollo Link [concepts guide](./overview.html).

<h2 id="usage">Usage</h2>

Apollo Link is easy to use with a variety of GraphQL libraries. It's designed to go anywhere you need to fetch GraphQL results.

<h3 id="apollo-client">Apollo Client</h3>

Apollo Client works seamlessly with Apollo Link. A Link is one of the required items when creating an [Apollo Client instance](/docs/react/reference/index.html#apollo-client). For simple HTTP requests, we recommend using [`apollo-link-http`](./links/http.html):

```js
import { ApolloLink } from 'apollo-link';
import { ApolloClient } from 'apollo-client';
import { InMemoryCache } from 'apollo-cache-inmemory';
import { HttpLink } from 'apollo-link-http';

const client = new ApolloClient({
  link: new HttpLink({ uri: 'http://api.githunt.com/graphql' }),
  cache: new InMemoryCache()
});
```

The `HttpLink` is a replacement for `createNetworkInterface` from Apollo Client 1.0. For more information on how to upgrade from 1.0 to 2.0, including examples for using middleware and setting headers, please check out our [upgrade guide](https://github.com/apollographql/apollo-link/tree/master/packages/apollo-link-http#upgrading-from-apollo-fetch--apollo-client).

<h3 id="graphql-tools">graphql-tools</h3>

You can also use Apollo Link with `graphql-tools` to facilitate schema stitching by using `node-fetch` as your request link's fetcher function and passing it to `makeRemoteExecutableSchema`.

```js
import { HttpLink } from 'apollo-link-http';
import fetch from 'node-fetch';

const link = new HttpLink({ uri: 'http://api.githunt.com/graphql', fetch });

const schema = await introspectSchema(link);

const executableSchema = makeRemoteExecutableSchema({
  schema,
  link,
});
```

You can read more about schema stitching with `graphql-tools` [here](https://www.apollographql.com/docs/graphql-tools/schema-stitching.html).

<h3 id="graphiql">GraphiQL</h3>

GraphiQL is a great way to document and explore your GraphQL API. In this example, we're setting up GraphiQL's fetcher function by using the `execute` function exported from Apollo Link. This function takes a link and an operation to create a GraphQL request. 

```js
import React from 'react';
import ReactDOM from 'react-dom';
import '../node_modules/graphiql/graphiql.css'
import GraphiQL from 'graphiql';
import { parse } from 'graphql';

import { execute } from 'apollo-link';
import { HttpLink } from 'apollo-link-http';

const link = new HttpLink({
  uri: 'http://api.githunt.com/graphql'
});

const fetcher = (operation) => {
  operation.query = parse(operation.query);
  return execute(link, operation);
};

ReactDOM.render(
  <GraphiQL fetcher={fetcher}/>,
  document.body,
);
```

With this setup, we're able to construct an arbitrarily complicated set of links (e.g. with polling, batching, etc.) and test it out using GraphiQL. This is incredibly useful for debugging as you're building a Link-based application.

<h3 id="relay-modern">Relay Modern</h3>

You can use Apollo Link as a network layer with Relay Modern.

```js
import {Environment, Network, RecordSource, Store} from 'relay-runtime';
import {execute, makePromise} from 'apollo-link';
import {HttpLink} from 'apollo-link-http';
import {parse} from 'graphql';

const link = new HttpLink({
  uri: 'http://api.githunt.com/graphql'
});

const source = new RecordSource();
const store = new Store(source);
const network = Network.create(
  (operation, variables) => makePromise(
    execute(link, {
      query: parse(operation.text),
      variables
    })
  )
);

const environment = new Environment({
    network,
    store
});
```

<h3 id="standalone">Standalone</h3>

You can also use Apollo Link as a standalone client. That is, you can use it to fire requests and receive responses from a GraphQL server. However, unlike a full client implementation such as Apollo Client, Apollo Link doesn't come with a reactive cache, UI bindings, etc. To use Apollo Link as a standalone client, we're using the `execute` function exported by Apollo Link in the following code sample:

```js
import { execute, makePromise } from 'apollo-link';
import { HttpLink } from 'apollo-link-http';

const uri = 'http://api.githunt.com/graphql';
const link = new HttpLink({ uri });

// execute returns an Observable so it can be subscribed to
execute(link, operation).subscribe({
  next: data => console.log(`received data ${data}`),
  error: error => console.log(`received error ${error}`),
  complete: () => console.log(`complete`),
})

// For single execution operations, a Promise can be used
makePromise(execute(link, operation))
  .then(data => console.log(`received data ${data}`))
  .catch(error => console.log(`received error ${error}`))
```

`execute` accepts a standard GraphQL request and returns an [Observable](https://github.com/tc39/proposal-observable) that allows subscribing. A GraphQL request is an object with a `query` which is a GraphQL document AST, `variables` which is an object to be sent to the server, an optional `operationName` string to make it easy to debug a query on the server, and a `context` object to send data directly to a link in the chain.
Links use observables to support GraphQL subscriptions, live queries, and polling, in addition to single response queries and mutations.

`makePromise` is similar to execute, except it returns a Promise. You can use `makePromise` for single response operations such as queries and mutations.

If you want to control how you handle errors, `next` will receive GraphQL errors, while `error` be called on a network error. We recommend using [`apollo-link-error`](https://github.com/apollographql/apollo-link/tree/master/packages/apollo-link-error) instead.

<h2 id="linkslist">Available Links</h2>

There are a number of useful links that have already been implemented that may be useful for your application.

[`apollo-link-http`](https://github.com/apollographql/apollo-link/tree/master/packages/apollo-link-http)
Get the results for a GraphQL query over HTTP. 

[`apollo-link-state`](https://github.com/apollographql/apollo-link-state)
Allows you to manage your application's non-data state and interact with it via GraphQL.

[`apollo-link-error`](https://github.com/apollographql/apollo-link/tree/master/packages/apollo-link-error)
Handle and inspect errors within your GraphQL stack.

[`apollo-link-retry`](https://github.com/apollographql/apollo-link/tree/master/packages/apollo-link-retry)
Attempts an operation multiple times if it fails due to network or server errors.

[`apollo-link-batch`](https://github.com/apollographql/apollo-link/tree/master/packages/apollo-link-batch)
Batches and manipulates a grouping of multiple GraphQL operations.

[`apollo-link-batch-http`](https://github.com/apollographql/apollo-link/tree/master/packages/apollo-link-batch-http)
Batches multiple GraphQL operations into a single HTTP request as an array of operations.

[`apollo-link-context`](https://github.com/apollographql/apollo-link/tree/master/packages/apollo-link-context)
Sets a context on your operation, which can be used other links further down the chain.

[`apollo-link-dedup`](https://github.com/apollographql/apollo-link/tree/master/packages/apollo-link-dedup)
Deduplicates requests before sending them down the wire. This link is included by default on Apollo Client.

[`apollo-link-schema`](https://github.com/apollographql/apollo-link/tree/master/packages/apollo-link-schema)
Assists with mocking and server-side rendering.

[`apollo-link-ws`](https://github.com/apollographql/apollo-link/tree/master/packages/apollo-link-ws)
Send GraphQL operations over a WebSocket. Works with GraphQL subscriptions.

<h2 id="customization">Customizing your own links</h2>

The links documented here and provided by the community have you covered for the most common use cases, but we've designed things so that it's easy to write your own. If you need your own versions of offline support or persisted queries, the `ApolloLink` interface was designed to be as flexible as possible to fit your application's needs.

To get started, first read our [concepts guide](./overview.html) and then learn how to write your own [stateless link](./stateless.html).
