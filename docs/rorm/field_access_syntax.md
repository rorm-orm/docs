# Field access syntax

Throughout rorm's API you refer to a model's fields
by writing them like a normal field access on the model's name:

```rust
User.username
```

This expression doesn't access any value —
there is no `User` instance in scope —
instead the `derive(Model)` macro makes the model's name
usable as an expression whose "fields" are so-called field proxies.

A field proxy represents the field's declaration (i.e. the column),
not its value. It is a zero-sized value you can pass around
and call methods on, and it is what the CRUD builders expect
in all places where a column is required:

```rust
// as condition
rorm::query(db, User)
    .condition(User.username.equals("bob"))
    .optional()
    .await?;

// as ordering
rorm::query(db, User).order_asc(User.username).all().await?;

// as selection
let names: Vec<(i64, String)> = rorm::query(db, (User.id, User.username)).all().await?;

// as update target
rorm::update(db, User).set(User.username, new_name).condition(...).await?;
```

## Traversing relations

Field access can walk through `ForeignModel` fields
to reach the related model's fields:

```rust
// The username of the user a post belongs to
Post.user.username
```

rorm resolves such paths automatically by adding the necessary JOINs
to the generated query.
This works in every position shown above — conditions, ordering and selection —
and paths may span multiple hops:

```rust
rorm::query(db, Comment)
    .condition(Comment.post.user.username.equals("alice"))
    .all()
    .await?;
```

## The `field!` macro

Some types need to refer to a field in *type* position,
for example `BackRef` and `ForeignModelByField`.
Since `Post.user` is an expression, it can't be used there directly —
the `field!` macro converts the familiar syntax into the field's type:

```rust
#[derive(Model)]
struct User {
    #[rorm(id)]
    id: i64,

    posts: BackRef<field!(Post.user)>,
}
```
