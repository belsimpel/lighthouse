# Authorization

Not every user in your application may be allowed to see all data or do any action.
You can control what they can do by enforcing authorization rules.

Before you can apply authorization, make sure you cover [authentication](authentication.md) first - it's
a prerequisite to have your users logged in before checking what they can do.

## Utilize the Viewer pattern

A common pattern is to allow users to only access entries that belong to them.
For example, a user may only be able to see notes they created.
You can utilize the nested nature of GraphQL queries to naturally limit access to such fields.

Begin with a field that represents the currently authenticated user, commonly called `me` or `viewer`.
You can resolve that field quite easily by using the [@auth](../api-reference/directives.md#auth) directive.

```graphql
type Query {
  me: User! @auth
}

type User {
  name: String!
}
```

Now, add related entities that are present as relationships onto the `User` type.

```graphql
type User {
  name: String!
  notes: [Note!]!
}

type Note {
  title: String!
  content: String!
}
```

Now, authenticated users can query for items that belong to them and are naturally
limited to seeing just those.

```graphql
{
  me {
    name
    notes {
      title
      content
    }
  }
}
```

## Restrict fields through policies

Lighthouse allows you to restrict field operations to a certain group of users.
Use the [@can](../api-reference/directives.md#can) directive
to leverage [Laravel Policies](https://laravel.com/docs/authorization) for authorization.

Starting from Laravel 5.7, [authorization of guest users](https://laravel.com/docs/authorization#guest-users) is supported.
Because of this, Lighthouse does **not** validate that the user is authenticated before passing it along to the policy.

### Protect mutations

As an example, you might want to allow only admin users of your application to create posts.
Start out by defining [@can](../api-reference/directives.md#can) upon a mutation you want to protect:

```graphql
type Mutation {
  createPost(input: PostInput): Post @can(ability: "create")
}
```

The `create` ability that is referenced in the example above is backed by a Laravel policy:

```php
final class PostPolicy
{
    public function create(User $user): bool
    {
        return $user->is_admin;
    }
}
```

### Protect specific model instances

For some models, you may want to restrict access for specific instances of a model.
Set the `resolved` argument to `true` to have Lighthouse check permissions against
the resolved model instances.

> This will actually run the field before checking permissions, do not use in mutations.

```graphql
type Query {
  post(id: ID! @eq): Post
    @can(ability: "view", resolved: true)
    @find
    @softDeletes
}
```

```php
final class PostPolicy
{
    public function view(User $user, Post $post): bool
    {
        return $user->id === $post->author_id;
    }
}
```

### Passing additional arguments

You can pass additional arguments to the policy checks by specifying them as `args`:

```graphql
type Mutation {
  createPost(input: PostInput): Post
    @can(ability: "create", args: ["FROM_GRAPHQL"])
}
```

```php
final class PostPolicy
{
    public function create(User $user, array $args): bool
    {
        // $args will be the PHP representation of what is in the schema: [0 => 'FROM_GRAPHQL']
    }
}
```

You can pass along the client given input data as arguments to the policy checks
with the `injectArgs` argument:

```graphql
type Mutation {
  createPost(title: String!): Post @can(ability: "create", injectArgs: true)
}
```

```php
final class PostPolicy
{
    public function create(User $user, array $injected): bool
    {
        // $injected will hold the args given by the client: ['title' => string(?)]
    }
}
```

When you combine both ways of passing arguments, the policy will be passed the `injectArgs` as
the second parameter and the static `args` as the third parameter:

```php
final class PostPolicy
{
    public function create($user, array $injectedArgs, array $staticArgs): bool { ... }
}
```

## Custom field restrictions

For applications with role management, it is common to hide some model attributes from a
certain group of users. At the moment, Laravel and Lighthouse offer no canonical solution
for this.

A great way to implement something that fits your use case is to create
[a custom `FieldMiddleware` directive](../custom-directives/field-directives.md#fieldmiddleware).
Field middleware allows you to intercept field access and conditionally hide them.
You can hide a field by returning `null` instead of calling the final resolver, or maybe even
abort execution by throwing an error.

The following directive `@canAccess` is an example implementation, make sure to adapt it to your needs.
It assumes a simple role system where a `User` has a single attribute `$role`.

```php
namespace App\GraphQL\Directives;

use Nuwave\Lighthouse\Execution\ResolveInfo;
use Nuwave\Lighthouse\Exceptions\DefinitionException;
use Nuwave\Lighthouse\Schema\Directives\BaseDirective;
use Nuwave\Lighthouse\Schema\Values\FieldValue;
use Nuwave\Lighthouse\Support\Contracts\FieldMiddleware;
use Nuwave\Lighthouse\Support\Contracts\GraphQLContext;

final class CanAccessDirective extends BaseDirective implements FieldMiddleware
{
    public static function definition(): string
    {
        return /** @lang GraphQL */ <<<GRAPHQL
"""
Limit field access to users of a certain role.
"""
directive @canAccess(
  """
  The name of the role authorized users need to have.
  """
  requiredRole: String!
) on FIELD_DEFINITION
GRAPHQL;
    }

    public function handleField(FieldValue $fieldValue): void
    {
        $fieldValue->wrapResolver(fn (callable $resolver) => function (mixed $root, array $args, GraphQLContext $context, ResolveInfo $resolveInfo) use ($resolver) {
            $requiredRole = $this->directiveArgValue('requiredRole');
            // Throw in case of an invalid schema definition to remind the developer
            if ($requiredRole === null) {
                throw new DefinitionException("Missing argument 'requiredRole' for directive '@canAccess'.");
            }

            $user = $context->user();
            if (
                // Unauthenticated users don't get to see anything
                ! $user
                // The user's role has to match have the required role
                || $user->role !== $requiredRole
            ) {
                return null;
            }

            return $resolver($root, $args, $context, $resolveInfo);
        });
    }
}
```

Here is how you would use it in your schema:

```graphql
type Post {
  #...
  secret_admin_note: String @canAccess(requiredRole: "admin")
}
```
