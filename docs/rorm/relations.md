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
