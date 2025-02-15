# Data Policies

ChiselStrike lets you express data policies through a policy file that
succinctly expresses your rules for how the data is served from
storage.  Although the policy language is currently limited, it is set
to rapidly expand in the immediate future.

Examples used in this chapter build on the `my-backend` files from the
[Introduction](Intro/first.md).

## Data Transformation

The most basic policy you could enact is saying that the data read
from ChiselStrike shall be systematically transformed before it's sent
to the frontend.  You basically say "this kind of data must be
transformed like this whenever it is accessed".

Let's first examine more precisely how you define which data we're
talking about.  This is done using _labels_: TypeScript decorators on
the fields of your models.

For instance, suppose we have a BlogComment object defined and add the "labels"
decorator as shown below:

```typescript title="my-backend/models/models.ts"
import { ChiselEntity, labels } from "@chiselstrike/api"

export class BlogComment extends ChiselEntity {
    content: string;
    @labels("pii") by: string;
}
```

Labels are specified by using the `@labels` decorator with a list of strings. Each
individual string denotes a label.

We add the `pii` label to the `by` field, because we intend to
refer to it when dictating how `by` should be treated.

:::note
You can pick any name for a label.  We don't have any restrictions or
conventions at this time.
:::

Now let's enforce a transformation on `pii` fields.  Please create
the file `my-backend/policies/pol.yml` like this:

```yaml title="my-backend/policies/pol.yml"
labels:
  - name: pii
    transform: anonymize
```

When you save this file, you should see this in the `chisel dev`
output:

```
Policy defined for label pii
```

And now notice how the output of the `comments` endpoint changes.

If you invoke the `/dev/comments` endpoint with:

```bash
curl -s localhost:8080/dev/comments
```

The `curl` command reports:

```json
{
    "results": [
        {
            "id": "a4ca3ab3-2e26-4da6-a5de-418c1e6b9b83",
            "content": "First comment",
            "by": "xxxxx"
        },
        {
            "id": "fed312d7-b36b-4f34-bb04-fba327a3f440",
            "content": "Second comment",
            "by": "xxxxx"
        },
        {
            "id": "adc89862-dfaa-43ab-a639-477111afc55e",
            "content": "Third comment",
            "by": "xxxxx"
        },
        {
            "id": "5bfef47e-371b-44e8-a2dd-88260b5c3f2c",
            "content": "Fourth comment",
            "by": "xxxxx"
        }
    ]
}
```

The `pii` fields were anonymized!  It is not possible for any
endpoint code to accidentally read `pii` data, eliminating human
errors from the process.

## Policy Exceptions

Here is how you can except the `comments` endpoint from automatic
data anonymization.  Please edit the file
`my-backend/policies/pol.yml` like this:

```yaml title="my-backend/policies/pol.yml"
labels:
  - name: pii
    transform: anonymize
    except_uri: /comments
```

The `except_uri` key lets you specify a path that's exempt from the
transformation policy being defined.  In this case, it matches exactly
the `comments` endpoint.  But in general, the value can be a path
prefix and even a regular expression; any matching endpoints will be
exempt from the policy.

If you now query the `/dev/comments` endpoint:

```bash
curl -s localhost:8080/dev/comments
```

The `curl` command reports:

```json
{
    "results": [
        {
            "id": "a4ca3ab3-2e26-4da6-a5de-418c1e6b9b83",
            "content": "First comment",
            "by": "Jill"
        },
        {
            "id": "fed312d7-b36b-4f34-bb04-fba327a3f440",
            "content": "Second comment",
            "by": "Jack"
        },
        {
            "id": "adc89862-dfaa-43ab-a639-477111afc55e",
            "content": "Third comment",
            "by": "Jim"
        },
        {
            "id": "5bfef47e-371b-44e8-a2dd-88260b5c3f2c",
            "content": "Fourth comment",
            "by": "Jack"
        }
    ]
}
```

As you can see, this endpoint now operates with the raw, untransformed
data.

## Policies for Logged-in Users

ChiselStrike supports [having users log into your dynamic
website](./login.md).  It even lets you restrict endpoint access by
user.

To restrict who can access the `comments` endpoint, please edit the
file `my-backend/policies/pol.yml` like this:

```yaml title="my-backend/policies/pol.yml"
endpoints:
  - path: /comments
    users: ^admin@example.com$
```

This says that only the user `admin@example.com` can access the `/comments`
endpoint.

:::note
We currently match `users` against the user's email -- the only field
NextAuth guarantees to be unique.  Your feedback is welcome as we
evolve this aspect of our product.
:::

The `endpoints` section can have any number of `path` items, each
affecting a different path prefix.  The `users` attribute is a regular
expression that the logged-in user's email must match in order to access
endpoints under the given path.

`path` values may overlap, in which case longer overrides shorter.
When you attempt to access an endpoint, the longest specified prefix
of its path dictates which users may access it.  Although multiple
`path` entries may overlap, they must not be identical.

When `users` is specified, anonymous access to the path will be
prohibited.  For example, if you want to force the user to log in to
access `comments` but don't care which specific user is accessing it,
you can set `users` to `.*`.

### Restricting Data Access to Matching User

As explained in ["Accessing User Info in the
Backend"](login#accessing-user-info-in-the-backend), you can store the
logged-in user as a field in your entities.  Let's continue the
example from that link here.  Please edit the file `models/models.ts`
like this:

```typescript title="my-backend/models/models.ts"
import { ChiselEntity } from "@chiselstrike/api"

export class BlogComment extends ChiselEntity {
    content: string = "";
    @labels("protect") author: AuthUser;
}
```

Then please add the following policy:

```yaml title="my-backend/policies/pol.yml"
labels:
  - name: protect
    transform: match_login
```

The `match_login` transformation compares fields labeled with
`protect` (if they are of AuthUser type) to the value of
`loggedInUser()`.  When the field value doesn't match, the row is
ignored.  So in this case, when endpoints read BlogComment entities,
they will see only the rows whose `author` matches the currently
logged-in user.

:::tip
You can use `except_uri` here, and it works the same as described
above.
:::
