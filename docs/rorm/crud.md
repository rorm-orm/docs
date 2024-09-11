# CRUD

To perform database operations, one of the four CRUD macros is invoked.

They all take an executor as their first argument (i.e. `&Database` or `&mut Transaction`)
and the model whose table to interact with as its second argument.

The macro evaluates into a builder whose methods can be chained to configure and execute the operation.

The following examples are written using these models:

```rust
use rorm::prelude::*;

#[derive(Model)]
struct User {
    #[rorm(id)]
    id: i64,

    #[rorm(max_length = 255, unique)]
    username: String,

    #[rorm(max_length = 255)]
    password: String,

    posts: BackRef<field!(Post::F.user)>,
}

#[derive(Model)]
struct Post {
    #[rorm(id)]
    id: i64,
    
    #[rorm(max_length = 255)]
    message: String,
    
    user: ForeignKey<User>,
}
```

## Query

In order to retrieve data from the database the `query!` macro is used:
```rust
async fn query_example(db: &Database) -> Result<(), rorm::Error> {
    let all_users: Vec<User> = query!(db, User)
        .all()
        .await?;
    
    let bob: Option<User> = query!(db, User)
        .condition(User::F.username.equals("bob"))
        .optional()
        .await;
    if bob.is_none() {
        println!("No user named bob was found");
    }
    
    Ok(())
}
```

The last method called on the macro specifies how the query is resolved. The following four are available:

### `all()`
Executes the query in a future which resolves to all rows matching the query.
```rust
let all_users: Vec<User> = query!(&db, User).all().await.unwrap();
```

### `one()`
Executes the query in a future which resolves to exactly one row.
(If no matching row is found, this will be treated as an error)
```rust
let user: User = query!(&db, User).one().await.unwrap();
```

### `optional()`
Executes the query in a future which resolves to at most one row.
```rust
let user: Option<User> = query!(&db, User).optional().await.unwrap();
```

### `stream()`
Behaves like `all()` but instead of returning a future which collects all resulting rows in a `Vec` before resolving,
it produces a stream which has to be polled per row.

If you're not comfortable with rust's async streams, you can always start using `all()` until you notice performance issues.

### Add conditions
The optional `.condition(...)` method can be invoked to add a condition the returned rows must satisfy.

!!! note
    This directly corresponds to adding a `WHERE` clause in sql

To construct conditions use a comparison method on the field syntax
```
User::F.username.equals("bob")
^^^^^^^^^^^^^^^^        ^^^^^ - Value to compare against
|                ^^^^^^
|                |
|                Comparison operator
|
Field to compare
```

The concrete comparisons available depend on the field's type.

Non-exhaustive list of commonly used ones:
`equals`, `not_equals`, `less_than`, `less_equals`, `greater_than`, `greater_equals`

Conditions can then be combined using the `or!` and `and!` macros:
```rust
query!(db, User)
    .condition(and!(
        User::F.username.equals("alice"),
        User::F.password.equals(leaked_pw)
    ))
    .optional()
```

### Customize what to select

In its most basic usage (`query!(db, User)`) the query will select every column and return them in the model's struct.

This can be customized by changing the macro's second argument.

!!! note
    The `query!` macro is somewhat unique in regard to its second argument.

    The other macros won't behave comparibly.

As simplest alternative a `Patch` can be specified instead of the whole `Model` to only select some columns:

```rust
#[derive(Patch)]
#[rorm(model = "User")]
struct UserWithoutPassword {
    id: i64,
    username: String,
    posts: BackRef<field!(Post::F.user)>,
}

// Query every field from the struct UserWithoutPassword
let users: Vec<UserWithoutPassword> = query!(db, UserWithoutPassword).all().await?;
```

This can get quite annoying when you have to specify a struct for every combination of fields you might want to query together.

Therefore, you can use the field syntax to query tuples of fields:
```rust
// Only query the user's id and username
let users: Vec<(i64, String)> =
    query!(db, (User::F.id, User::F.username)).all().await?;
```

This syntax also work with relations:
```rust
let posts: Vec<(String, String)> =
    query!(db, (Post::F.message, Post::F.user.username)).all().await?;
```

When you want to select your relations' fields and there are a lot of them, specifying them all like this can get quite verbose.
On top of that, due to rust limitations regarding tuples, the maximum number of fields you can query in one go is 32.
To mitigate this, there is a syntax combining individual fields with patches:

```rust
let posts: Vec<(String, UserWithoutPassword)> =
    query!(db, (Post::F.message, Post::F.user as UserWithoutPassword)).all().await?;
```

#### Limit & offset

Just as you would expect them, the `.limit(u64)` and `.offset(u64)` functions can be used to add limits and offsets
to the SQL query. Note that `.limit()` can not be combined with `.one()`, since the latter already adds a limit of 1.
Also, the `.offset()` can not be used without a limit. The following examples illustrate possible uses:

```rust
query!(db, Post).limit(10).all().await?;
query!(db, Post).offset(13).one().await?;
query!(db, Post).limit(10).offset(42).all().await?;
query!(db, Post).limit(100).offset(1337).stream();
```

There is also the `.range()` function which provides a convenient way to add both the limit and offset.
Mixing a range with the previous functions for limit and offset is not allowed.
Thus, the following example will return at most 10 elements since it corresponds to limit 10 and offset 30:

```rust
query!(db, Post).range(30..40).all().await?;
```

#### Ordering

TODO: `order_...`

## Insert

In order to create new rows in the database, the `insert!` macro is used:

```rust
#[derive(Patch)]
#[rorm(model = "User")]
struct NewUser {
    username: String,
    password: String,
}

#[derive(Patch)]
#[rorm(model = "Post")]
struct NewPost {
    message: String,
    user: ForeignKey<User>,
}

async fn insert_example(db: &Database) -> Result<(), rorm::Error> {
    // insert a single user
    insert!(db, NewUser).single(&NewUser {
        username: "alice".to_string(),
        password: "Secure-123".to_string(),
    }).await?;

    // insert a collection of user posts
    let posts: Vec<> = vec![...];
    insert!(db, NewPost).bulk(&posts).await?;
    
    Ok(())
}
```

!!! note
    Since the `id` field is annotated with `#[rorm(id)]` it is set by the database.

    Therefore, we mustn't set it ourselves. 
    So we declare a patch (`NewUser` or `NewPost`) which doesn't contain it and insert that
    instead of the model structs themselves.

But what if you need the values of fields which are set by the database?

### Returning

The `insert!` macro returns the whole model it just inserted:

```rust
#[derive(Patch)]
#[rorm(model = "User")]
struct NewUser {
    username: String,
    password: String,
}

// Note that the type `User` is returned instead of just a `NewUser`
let new_user: User = insert!(db, NewUser).single(&NewUser {..}).await?;
```

To customize the behaviour, a family of `.return_...()` methods are provided:
```rust
pub async fn show_various_returns(db: &Database, user: &NewUser) -> Result<(), Error> {
    // Return model instance by default
    let _: User = insert!(db, NewUser)
        .single(user)
        .await?;

    // Return a patch's instance instead of whole model
    // (including the one used to insert and the model itself)
    let _: AnotherUserPatch = insert!(db, NewUser)
        .return_patch::<UserPatch>()
        .single(user)
        .await?;

    // Return a tuple of fields
    let _: (i64, String) = insert!(db, NewUser)
        .return_tuple((User::F.id, User::F.name))
        .single(user)
        .await?;

    // Return the model's primary key
    let _: i64 = insert!(db, NewUser)
        .return_primary_key()
        .single(user)
        .await?;

    // Return nothing
    let _: () = insert!(db, NewUser)
        .return_nothing()
        .single(user)
        .await?;

    Ok(())
}
```

## Update

In order to change models' fields the `update!` macro is used:

```rust
pub async fn set_good_password(db: &Database) -> Result<(), rorm::Error> {
    update!(db, User)
        .set(User::F.password, "I am way more secure™".to_string())
        .condition(User::F.password.equals("password"))
        .await?;
    Ok(())
}
```

### Dynamic mode and `set_if`

Before executing the query, `set` has to be called at least once
to set a value for a column (the first call changes the builder's type).
Otherwise the query wouldn't do anything.

This can be limiting when your calls are made conditionally.

To support this, the builder can be put into a "dynamic" mode by calling `begin_dyn_set`.
Then calls to `set` won't change the type.
When you're done, use `finish_dyn_set` to go back to "normal" mode.
It will check the number of "sets" and return `Result` which is `Ok` for at least one and an `Err` for zero.
Both variants contain the builder in "normal" mode to continue.

A common pattern for dynamic mode is to check a bunch of `Option`s and inserting the `Some`s:
```rust
async fn update_user(
    db: &Database,
    user_id: i64,
    optional_new_username: Option<String>,
    optional_new_password: Option<String>
) -> Result<(), rorm::Error> {
    let mut updater = update!(db, User)
        .condition(User::F.id.equals(user_id))
        .begin_dyn_set();

    if let Some(new_username) = optional_new_username {
        updater = updater.set(User::F.username, new_username);
    }
    if let Some(new_password) = optional_new_password {
        updater = updater.set(User::F.password, new_password);
    }

    match updater.finish_dyn_set() {
        Ok(updater) => updater.await?,
        Err(_) => println!("Nothing to update"),
    }
    
    Ok(())
}
```

This can be simplified using the `set_if` method which takes an `Option` and calls `set` internally if the option is `Some`:
```rust
async fn update_user(
    db: &Database,
    user_id: i64,
    optional_new_username: Option<String>,
    optional_new_password: Option<String>
) -> Result<(), rorm::Error> {
    let updater = update!(db, User)
        .condition(User::F.id.equals(user_id))
        .set_if(User::F.username, optional_new_username)
        .set_if(User::F.password, optional_new_password);
    
    match updater.finish_dyn_set() {
        Ok(updater) => updater.await?,
        Err(_) => println!("Nothing to update"),
    }

    Ok(())
}
```

## Delete

In order to delete rows from a table, the `delete!` macro is used:

```rust
pub async fn delete_single_user(db: &Database, user: &UserPatch) -> Result<(), rorm::Error> {
    delete!(db, User)
        .single(user)
        .await?;
    Ok(())
}
pub async fn delete_many_users(db: &Database, users: &[UserPatch]) -> Result<(), rorm::Error> {
    delete!(db, User)
        .bulk(users)
        .await?;
    Ok(())
}
pub async fn delete_underage(db: &Database) -> Result<(), rorm::Error> {
    let num_deleted: u64 = delete!(db, User)
        .condition(User::F.age.less_equals(18))
        .await?;
    Ok(())
}
```

A `delete!` is quite simple. It expects only one of four methods which specify what to delete:

### `.single(...)`
To delete a single row, use the `single` method and pass it the model to delete.

!!! note
    `single` doesn't require the actual model. It accepts any patch which contains the primary key.

### `.bulk(...)`
`bulk` is used like `single` but takes an iterator of instances and deletes all at once.

### `.condition(...)`
When you need more complex deleting logic than just some concrete instances,
you can use the `conditon` method.
Any row which matches the provided condition will be deleted.

See [query](crud.md#add-conditions) for how conditions look.

### `all()`
For the case where you'd want to wipe the whole table, you can use the `all` method.
