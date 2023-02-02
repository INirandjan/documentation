---
title: Error handling
displayed_sidebar: devDocsSidebar
description: With Strapi's error handling feature it's easy to send and receive errors in your application.
canonicalUrl: https://docs.strapi.io/developer-docs/latest/developer-resources/error-handling.html
---

# Error handling

Strapi is natively handling errors with a standard format.

There are 3 use cases for error handling:

- As a developer querying content through the [REST](/dev-docs/api/rest) or [GraphQL](/dev-docs/api/graphql) APIs, you might [receive errors](#receiving-errors) in response to the requests.
- As a developer customizing the backend of your Strapi application, you could use controllers and services to [throw errors](#throwing-errors).
- As a developer customizing the backend you can rollback database entries from actions that result in an error using database transactions.

## Receiving errors

Errors are included in the response object with the `error` key and include information such as the HTTP status code, the name of the error, and additional information.

### REST errors

Errors thrown by the REST API are included in the [response](/dev-docs/api/rest#requests) that has the following format:

```json
{
  "data": null,
  "error": {
    "status": "", // HTTP status
    "name": "", // Strapi error name ('ApplicationError' or 'ValidationError')
    "message": "", // A human readable error message
    "details": {
      // error info specific to the error type
    }
  }
}
```

### GraphQL errors

Errors thrown by the GraphQL API are included in the [response](/dev-docs/api/graphql#unified-response-format) that has the following format:

```json
{ "errors": [
    {
      "message": "", // A human reable error message
      "extensions": {
        "error": {
          "name": "", // Strapi error name ('ApplicationError' or 'ValidationError'),
          "message": "", // A human reable error message (same one as above);
          "details": {}, // Error info specific to the error type
        },
        "code": "" // GraphQL error code (ex: BAD_USER_INPUT)
      }
    }
  ],
  "data": {
    "graphQLQueryName": null
  }
}
```

## Throwing errors

### Controllers and middlewares

The recommended way to throw errors when developing any custom logic with Strapi is to have the [controller](/dev-docs/backend-customization/controllers) or [middleware](/dev-docs/backend-customization/middlewares) respond with the correct status and body.

This can be done by calling an error function on the context (i.e. `ctx`). Available error functions are listed in the [http-errors documentation](https://github.com/jshttp/http-errors#list-of-all-constructors) but their name should be lower camel-cased to be used by Strapi (e.g. `badRequest`).

Error functions accept 2 parameters that correspond to the `error.message` and `error.details` attributes [received](#receiving-errors) by a developer querying the API:

- the first parameter of the function is the error `message`
- and the second one is the object that will be set as `details` in the response received

<Tabs groupId="js-ts">

<TabItem value="javascript" label="JavaScript">

```js
// path: ./src/api/[api-name]/controllers/my-controller.js

module.exports = {
  renameDog: async (ctx, next) => {
    const newName = ctx.request.body.name;
    if (!newName) {
      return ctx.badRequest('name is missing', { foo: 'bar' })
    }
    ctx.body = strapi.service('api::dog.dog').rename(newName);
  }
}

// path: ./src/api/[api-name]/middlewares/my-middleware.js

module.exports = async (ctx, next) => {
  const newName = ctx.request.body.name;
  if (!newName) {
    return ctx.badRequest('name is missing', { foo: 'bar' })
  }
  await next();
}
```

</TabItem>

<TabItem value="typescript" label="TypeScript">

```ts
// path: ./src/api/[api-name]/controllers/my-controller.ts

export default {
  renameDog: async (ctx, next) => {
    const newName = ctx.request.body.name;
    if (!newName) {
      return ctx.badRequest('name is missing', { foo: 'bar' })
    }
    ctx.body = strapi.service('api::dog.dog').rename(newName);
  }
}

// path: ./src/api/[api-name]/middlewares/my-middleware.ts

export default async (ctx, next) => {
  const newName = ctx.request.body.name;
  if (!newName) {
    return ctx.badRequest('name is missing', { foo: 'bar' })
  }
  await next();
}
```

</TabItem>

</Tabs>

### Services and models lifecycles

Once you are working at a deeper layer than the controllers or middlewares there are dedicated error classes that can be used to throw errors. These classes are extensions of [Node `Error` class](https://nodejs.org/api/errors.html#errors_class_error) and are specifically targeted for certain use-cases.

These error classes are imported through the `@strapi/utils` package and can be called from several different layers. The following examples use the service layer but error classes are not just limited to services and model lifecycles. When throwing errors in the model lifecycle layer, it's recommended to use the `ApplicationError` class so that proper error messages are shown in the admin panel.

:::note
See the [default error classes](#default-error-classes) section for more information on the error classes provided by Strapi.
:::

<details>
<summary>Example: Throwing an error in a service</summary>
This example shows wrapping a [core service](/dev-docs/backend-customization/services#extending-core-services) and doing a custom validation on the `create` method:

<Tabs groupId="js-ts">

<TabItem value="javascript" label="JavaScript">

```js title="path: ./src/api/restaurant/services/restaurant.js"

const utils = require('@strapi/utils');
const { ApplicationError } = utils.errors;
const { createCoreService } = require('@strapi/strapi').factories;

module.exports = createCoreService('api::restaurant.restaurant', ({ strapi }) =>  ({
  async create(params) {
    let okay = false;

    // Throwing an error will prevent the restaurant from being created
    if (!okay) {
      throw new ApplicationError('Something went wrong', { foo: 'bar' });
    }
  
    const result = await super.create(params);

    return result;
  }
});
```

</TabItem>

<TabItem value="typescript" label="TypeScript">

```ts title="path: ./src/api/[api-name]/policies/my-policy.ts"

import utils from '@strapi/utils';
import { factories } from '@strapi/strapi';

const { ApplicationError } = utils.errors;

export default factories.createCoreService('api::restaurant.restaurant', ({ strapi }) =>  ({
  async create(params) {
    let okay = false;

    // Throwing an error will prevent the restaurant from being created
    if (!okay) {
      throw new ApplicationError('Something went wrong', { foo: 'bar' });
    }
  
    const result = await super.create(params);

    return result;
  }
}));

```

</TabItem>

</Tabs>

</details>

<details>
<summary>Example: Throwing an error in a model lifecycle</summary>

This example shows building a [custom model lifecycle](/dev-docs/backend-customization/models#lifecycle-hooks) and being able to throw an error that stops the request and will return proper error messages to the admin panel. Generally you should only throw an error in `beforeX` lifecycles, not `afterX` lifecycles.

<Tabs groupId="js-ts">

<TabItem value="javascript" label="JavaScript">

```js title="path: ./src/api/[api-name]/content-types/[api-name]/lifecycles.js"

const utils = require('@strapi/utils');
const { ApplicationError } = utils.errors;

module.exports = {
  beforeCreate(event) {
    let okay = false;

    // Throwing an error will prevent the entity from being created
    if (!okay) {
      throw new ApplicationError('Something went wrong', { foo: 'bar' });
    }
  },
};

```

</TabItem>

<TabItem value="typescript" label="TypeScript">

```ts title="path: ./src/api/[api-name]/content-types/[api-name]/lifecycles.ts"

import utils from '@strapi/utils';
const { ApplicationError } = utils.errors;

export default {
  beforeCreate(event) {
    let okay = false;

    // Throwing an error will prevent the entity from being created
    if (!okay) {
      throw new ApplicationError('Something went wrong', { foo: 'bar' });
    }
  },
};
```

</TabItem>

</Tabs>

</details>

### Policies

[Policies](/dev-docs/backend-customization/policies) are a special type of middleware that are executed before a controller. They are used to check if the user is allowed to perform the action or not. If the user is not allowed to perform the action and a `return false` is used then a generic error will be thrown. As an alternative, you can throw a custom error message using a nested class extensions from the Strapi `ForbiddenError` class, `ApplicationError` class (see [Default error classes](#default-error-classes) for both classes), and finally the [Node `Error` class](https://nodejs.org/api/errors.html#errors_class_error).

The `PolicyError` class is available from `@strapi/utils` package and accepts 2 parameters:

- the first parameter of the function is the error `message`
- (optional) the second parameter is the object that will be set as `details` in the response received; a best practice is to set a `policy` key with the name of the policy that threw the error.

<details>
<summary>Example: Throwing a PolicyError in a custom policy</summary>

This example shows building a [custom policy](/dev-docs/backend-customization/policies) that will throw a custom error message and stop the request.

<Tabs groupId="js-ts">

<TabItem value="javascript" label="JavaScript">

```js title="path: ./src/api/[api-name]/policies/my-policy.js"

const utils = require('@strapi/utils');
const { PolicyError } = utils.errors;

module.exports = (policyContext, config, { strapi }) => {
  let isAllowed = false;

  if (isAllowed) {
    return true;
  } else {
    throw new PolicyError('You are not allowed to perform this action', {
      policy: 'my-policy',
      myCustomKey: 'myCustomValue',
    });
  }
}

```

</TabItem>

<TabItem value="typescript" label="TypeScript">

```ts title="path: ./src/api/[api-name]/policies/my-policy.ts"

const utils = require('@strapi/utils');
const { PolicyError } = utils.errors;

export default (policyContext, config, { strapi }) => {
  let isAllowed = false;

  if (isAllowed) {
    return true;
  } else {
    throw new PolicyError('You are not allowed to perform this action', {
      policy: 'my-policy',
      myCustomKey: 'myCustomValue',
    });
  }
};
```

</TabItem>

</Tabs>

</details>

### Default error classes

The default error classes are available from the `@strapi/utils` package and can be imported and used in your code. Any of the default error classes can be extended to create a custom error class. The custom error class can then be used in your code to throw errors.

<Tabs> 

<TabItem value="Application" label="Application">

The `ApplicationError` class is a generic error class for application errors and is generally recommended as the default error class. This class is specifically designed to throw proper error messages that the admin panel can read and show to the user. It accepts the following parameters:

| Parameter | Type | Description | Default |
| --- | --- | --- | --- |
| `message` | `string` | The error message | `An application error occured` |
| `details` | `object` | Object to define additional details | `{}` |

```js
throw new ApplicationError('Something went wrong', { foo: 'bar' });
```

</TabItem>

<!-- Not sure if it's worth keeping this tab or not as it's very specific to Strapi internal use-cases -->
<!-- ::: tab Validation

The `ValidationError` and `YupValidationError` classes are specific error classes designed to be used with the built in validations system and specifically format the errors coming from [Yup](https://www.npmjs.com/package/yup). The `ValidationError` does not accept any parameters but the `YupValidationError` accepts the following parameters:

| Parameter | Type | Description | Default |
| --- | --- | --- | --- |
| `message` | `string` | The error message | - |
| `details` | `object` | Object to define additional details | `{ yupErrors }` |

```js

```js
throw new PolicyError('Something went wrong', { policy: 'my-policy' });
```

::: -->

<TabItem value="Pagination" label="Pagination">

The `PaginationError` class is a specific error class that is typically used when parsing the pagination information from [REST](/dev-docs/api/rest/sort-pagination#pagination), [GraphQL](/dev-docs/api/graphql#pagination), or the [Entity Service](/dev-docs/api/entity-service/order-pagination#pagination). It accepts the following parameters:

| Parameter | Type | Description | Default |
| --- | --- | --- | --- |
| `message` | `string` | The error message | `Invalid pagination` |

```js
throw new PaginationError('Exceeded maximum pageSize limit');
```

</TabItem>

<TabItem value="NotFound" label="NotFound">

The `NotFoundError` class is a generic error class for throwing `404` status code errors. It accepts the following parameters:

| Parameter | Type | Description | Default |
| --- | --- | --- | --- |
| `message` | `string` | The error message | `Entity not found` |

```js
throw new NotFoundError('These are not the droids you are looking for');
```

</TabItem>

<TabItem value="Forbidden" label="Forbidden">

The `ForbiddenError` class is a specific error class used when a user either doesn't provide any or the correct authentication credentials. It accepts the following parameters:

| Parameter | Type | Description | Default |
| --- | --- | --- | --- |
| `message` | `string` | The error message | `Forbidden access` |

```js
throw new ForbiddenError('Ah ah ah, you didn\'t say the magic word');
```

</TabItem>

<TabItem value="Unauthorized" label="Unauthorized">

The `UnauthorizedError` class is a specific error class used when a user doesn't have the proper role or permissions to perform a specific action, but has properly authenticated. It accepts the following parameters:

| Parameter | Type | Description | Default |
| --- | --- | --- | --- |
| `message` | `string` | The error message | `Unauthorized` |

```js
throw new UnauthorizedError('You shall not pass!');
```

</TabItem>

<TabItem value="PayloadTooLarge" label="PayloadTooLarge">

The `PayloadTooLargeError` class is a specific error class used when the incoming request body or attached files exceed the limits of the server. It accepts the following parameters:

| Parameter | Type | Description | Default |
| --- | --- | --- | --- |
| `message` | `string` | The error message | `Entity too large` |

```js
throw new PayloadTooLargeError('Uh oh, the file too big!');
```

</TabItem>

<TabItem value="Policy" label="Policy">

The `PolicyError` class is a specific error designed to be used with [route policies](/dev-docs/backend-customization/policies). The best practice recommendation is to ensure the name of the policy is passed in the `details` parameter. It accepts the following parameters:

| Parameter | Type | Description | Default |
| --- | --- | --- | --- |
| `message` | `string` | The error message | `Policy Failed` |
| `details` | `object` | Object to define additional details | `{}` |

```js
throw new PolicyError('Something went wrong', { policy: 'my-policy' });
```

</TabItem>

</Tabs>

## Rollback with database transactions

Queries using the [Entity Service API](/dev-docs/api/entity-service) and [Query Engine API](/dev-docs/api/query-engine) have access to database transactions in v4.7.0 and later versions of Strapi. Database transactions allow you to rollback incomplete actions due to an error. For example, if a controller contains multiple database queries with interspersed business logic an error thrown at any of the steps wrapped in a database transaction will return the database to its state prior to the controller starting.

In the callback param you have access to the transaction instance (from knex) if needed, and a rollback and commit function if needed as well"

### Wrapping queries with database transactions

Everything that is inside a the transaction can be:

- sent through the callback,
- is automatically rolled back if there is a failure and throws an error, 
- otherwise commits the transaction.

```js

strapi.db.transaction(async ({trx,rollback,commit}) => {

await  strapi.entityService.create();
await  strapi.entityService.create();

});

The transaction is managed by passing an object with three functions as a parameter to `strapi.db.transaction`: `trx`, `rollback`, and `commit`. `trx` is a transaction object that is used to perform database operations, `rollback` is a function that can be used to undo any changes made during the transaction if something goes wrong, and `commit` is a function that can be used to persist the changes made during the transaction.
