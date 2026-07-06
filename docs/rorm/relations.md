# Relations

## Foreign Keys

You can use the type `ForeignModel<T>` where `T` is any type that derives 
also `Model`. It will create a reference to the primary key of the
model `T`.


```rust
use rorm::prelude::*;

#[derive(Model)]
struct Post {
    #[rorm(id)]
    id: i64,
    
    creator: ForeignModel<User>,
}

#[derive(Model)]
struct User {
    #[rorm(id)]
    id: i64,
}
```

!!! tip

    You can point a `ForeignModel<T>` to the `Model` its contained in to 
    build self-refenencing structs.

    ```rust
    use rorm::prelude::*;
    
    #[derive(Model)]
    struct Post {
        #[rorm(id)]
        id: i64,
    
        creator: ForeignModel<Post>,
    }
    ```

### Cascading foreign keys

By default the database will refuse to delete or update a row
while another row's foreign key is still pointing to it (`Restrict`).
This behavior can be changed with the `on_delete` and `on_update`
[annotations](./model_declaration.md#on_delete-on_update).

Since `Cascade` for both is such a common combination —
"when the parent is deleted, delete its children as well" —
there is a dedicated type for it: `CascadingForeignModel<T>`.
It behaves exactly like a `ForeignModel<T>` with
`#[rorm(on_delete = "Cascade", on_update = "Cascade")]`,
but expresses the cascading behavior directly in the type.

```rust
use rorm::fields::types::CascadingForeignModel;
use rorm::prelude::*;

#[derive(Model)]
struct Post {
    #[rorm(id)]
    id: i64,

    /// Deleting the thread deletes its posts as well
    thread: CascadingForeignModel<Thread>,

    /// Deleting the user keeps the post but clears the field
    #[rorm(on_delete = "SetNull")]
    user: Option<ForeignModel<User>>,
}
```

Like `ForeignModel` has `ForeignModelByField`, there is a
`CascadingForeignModelByField<T>` to reference non-primary keys (see below).

### Foreign keys on non-primary keys

With `ForeignModel` it is not possible to reference a non-primary key.
In order to support this use case, the slightly more verbose 
`ForeignModelByField<T>` type was created.

For the generic parameter `T`, you have to use the provided `field!()` macro,
with the [field access syntax](./field_access_syntax.md).

```rust
use rorm::prelude::*;

#[derive(Model)]
struct Post {
    #[rorm(id)]
    id: i64,
    
    creator_aid: ForeignModelByField<field!(User.another_id)>,
}

#[derive(Model)]
struct User {
    #[rorm(id)]
    id: i64,
    
    #[rorm(unique)]
    another_id: i64,
}

```

## Backrefs

A backref is the (virtual) reference to be able to query 
"all items of a specific model with a `ForeignModel` pointing to the model on 
which the backref is defined on."

```rust
use rorm::prelude::*;

#[derive(Model)]
struct Post {
    #[rorm(id)]
    id: i64,
    
    creator: ForeignModel<User>,
}

#[derive(Model)]
struct User {
    #[rorm(id)]
    id: i64,
    
    posts: BackRef<field!(Post.creator)>
}
```

!!! note
    
    As `BackRef`s are not a concept of databases, but rather a convenience
    feature provided by rorm to simplify querying, there's no need to
    make migrations after the field was added, as it handled internally by
    rorm.

### Using a backref

A `BackRef` field is not filled automatically when its model is queried —
it acts as a cache which starts out empty.
It is filled explicitly through the [field access syntax](./field_access_syntax.md):

```rust
let mut user: User = rorm::query(db, User)
    .condition(User.id.equals(user_id))
    .one()
    .await?;

// Query the user's posts into the backref's cache ...
User.posts.populate(db, &mut user).await?;

// ... and access them through `get`
let posts: &Vec<Post> = user.posts.get().unwrap();
```

`populate` always queries, overwriting a previously filled cache.
If you just want the referenced rows and don't care about the cache,
there are two convenience methods combining both steps:

```rust
// Populates only if the cache is still empty, then borrows it
let posts: &mut [Post] = User.posts.get_or_query(db, &mut user).await?;

// Like `get_or_query` but takes ownership, leaving the cache empty again
let posts: Vec<Post> = User.posts.take_or_query(db, &mut user).await?;
```

For filling the backrefs of several instances at once,
there is `populate_bulk` which takes a mutable slice instead of a single instance.
