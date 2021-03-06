---
layout: page
path: /postgraphile/custom-mutations/
title: Custom Mutations
---

## Custom Mutations

PostGraphile automatically generates [CRUD
Mutations](/postgraphile/crud-mutations/) for you; but it's rare that these
will cover all your needs - and many people just disable them outright.
Custom mutations enable you to write exactly the business logic you need with
access to all of your data all wrapped up in a PostgreSQL function. You can
even bypass the RLS and GRANT checks, should you so choose, by tagging your
function as `SECURITY DEFINER` - but be very careful when you do so!

To create a function that PostGraphile will recognise as a custom mutation,
it must obey the following rules:

- adhere to [common PostGraphile function restrictions](/postgraphile/function-restrictions/)
- must be marked as `VOLATILE` (which is the default for PostgreSQL functions)
- must be defined in one of the introspected schemas

Functions matching these requirements will be represented in GraphQL in a way that is compatible with the [Relay Input Object Mutations Specification](https://facebook.github.io/relay/graphql/mutations.htm). For example the function

```sql
CREATE FUNCTION my_function(a int, b int) RETURNS text AS $$ … $$ LANGUAGE sql VOLATILE;
```

could be called from GraphQL like this:

```graphql
mutation {
  myFunction(input: { a: 1, b: 2 }) {
    text
  }
}
```

Look at the documentation in GraphiQL to find the parameters you may use!

### Example

Here's an example of a custom mutation, which will generate the graphql `acceptTeamInvite` mutation:

```sql
CREATE FUNCTION app_public.accept_team_invite(team_id integer)
RETURNS app_public.team_members
AS $$
  UPDATE app_public.team_members
    SET accepted_at = now()
    WHERE accepted_at IS NULL
    AND team_members.team_id = accept_team_invite.team_id
    AND member_id = app_public.current_user_id()
    RETURNING *;
$$ LANGUAGE sql VOLATILE STRICT SECURITY DEFINER;
```

Notes on the above function:

- `STRICT` is optional, it means that if any of the arguments are null then the
  mutation will not be called (and will thus return null with no error) - this
  allows us to mark `teamId` as a required argument.
- `SECURITY INVOKER` is the default, it means the function will run with the
  _security_ of the person who _invoked_ the function
- `SECURITY DEFINER` means that the function will run with the _security_ of
  the person who _defined_ the function, typically the database owner - this
  means that the function may bypass RLS, RBAC and other security concerns. Be
  careful when using `SECURITY DEFINER` - think of it like `sudo`!
- we use `LANGUAGE sql` here, but you can use `LANGUAGE plpgsql` if you need
  variables or looping or if blocks or similar concerns; or if you want to
  write in a more familiar language you can use `LANGUAGE plv8` (JavaScript,
  requires extension), or one of the built in `LANGUAGE` options such as
  Python, Perl or Tcl

A note on **named types**: if you have a function that `RETURNS SETOF table(a int, b text)` then PostGraphile will not _currently_ pick it up due to the
[common PostGraphile function
restrictions](/postgraphile/function-restrictions/). Work is underway to lift
these restrictions, but it's easy to work around for now - just define a
named type:

```sql
CREATE TYPE my_function_return_type AS (
  a int,
  b text
);
```

and then change your function to `RETURNS SETOF my_function_return_type`.

<!--
### Graphile Plugins

If you prefer adding mutations on the JavaScript side, you can use
`ExtendSchemaPlugin` from `graphile-utils`; see [Schema
Plugins](/postgraphile/extending/) for more information.

### GraphQL Schema Stitching

You can also stitch multiple GraphQL schemas together, you can read more about
doing this with PostGraphile here: [Authenticated and Stitched Schemas with
PostGraphile, Passport and
Stripe](https://medium.com/@sastraxi/authenticated-and-stitched-schemas-with-postgraphile-passport-and-stripe-a51490a858a2).

-->
