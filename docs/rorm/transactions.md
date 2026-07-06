# Transactions

A transaction is a sequence of one or more database operations that 
are treated as a single unit of work. Transactions in a database are used to
ensure data integrity, consistency, and reliability when multiple operations 
need to be executed as a single, atomic unit.

In rorm, a transaction can be retrieved from a `Database` instance:

```rust
use std::error::Error;

use rorm::prelude::*;

async fn transaction(db: &Database) -> Result<Transaction, Box<dyn Error>> {
    Ok(db.start_transaction().await?)
}
```

The resulting `Transaction` can be used in place of the `Database` instance,
with the difference that the instance must be provided as mutable reference.


There are two possible options to end a transaction:


* `commit()` This will end the transaction and apply all modifications to the database
* `rollback()` This will end the transaction and rollback all modifications

!!! tip

    Rollback is default that gets executed if a `Transaction`
    is dropped.


## Example usage

```rust
use std::fs::File;
use std::io::Write;

use rorm::prelude::*;

#[derive(Model)]
pub struct User {
    #[rorm(id)]
    pub id: i64,
    
    #[rorm(unique, max_length = 255)]
    pub username: String,
}

#[derive(Patch)]
#[rorm(model = "User")]
pub struct UserInsert {
    pub username: String,
}

pub async fn create_user(
    db: &Database,
    username: String,
    profile_picture: &[u8]
) -> Result<(), Box<dyn std::error::Error>> {
    // Start a transaction
    let mut tx = db.start_transaction().await?;

    // Check if the username is already taken
    let is_taken = rorm::query(&mut tx, User.id)
        .condition(User.username.equals(&username))
        .optional()
        .await?
        .is_some();
    
    if is_taken {
        return Err("Username is already taken".to_string());
    }
    
    // Insert the new user
    let id = rorm::insert(&mut tx, UserInsert)
        .return_primary_key()
        .single(&UserInsert { username })
        .await
        .unwrap();

    // If any of the io tasks throw an error, the transaction is dropped when the 
    // function exits, and a rollback is performed.
    // So the user wouldn't be created in the database in this case
    let mut file = File::create(format!("profile_pictures/{id}.png"))?;
    file.write_all(profile_picture)?;
    
    // Commit the transaction
    tx.commit().await?;
    
    Ok(())
}
```

## Transaction hooks

Sometimes a transaction is accompanied by side effects outside the database —
an in-memory cache is updated, a file is written, another service is notified.
To keep those side effects consistent with the transaction's outcome,
closures can be registered on the transaction through `Transaction::hooks()`:

```rust
let mut tx = db.start_transaction().await?;

tx.hooks()
    .pre_commit(|| async {
        // Runs before the COMMIT is sent to the database.
        // Returning an error aborts the commit.
        Ok(())
    })
    .post_commit(|| {
        // Runs after the transaction has been committed successfully.
    })
    .on_rollback(|| {
        // Runs when the transaction is rolled back —
        // explicitly or by being dropped without a commit.
    });
```

There are three hook points:

- `pre_commit` takes an async closure which is run before the actual `COMMIT` is sent to the database.
  If it returns an error, the transaction is not committed.
  (Note: the commit can still fail afterward due to a database error.)
- `post_commit` takes a sync closure which is only run after the database confirmed the commit.
- `on_rollback` takes a sync closure which is run when the transaction is rolled back,
  no matter whether through an explicit call to `rollback()` or by dropping the transaction.
  It may be called before, during or after the actual database operation —
  don't rely on any specific ordering relative to the database's rollback.

A common pattern is keeping caches in sync with the database:
write to the cache optimistically and register a hook to undo the write
in case the transaction doesn't go through.

```rust
async fn create_device(
    db: &Database,
    cache: &'static DeviceCache,
    new_device: NewDevice,
) -> Result<Device, rorm::Error> {
    let mut tx = db.start_transaction().await?;

    let device = rorm::insert(&mut tx, NewDevice)
        .single(&new_device)
        .await?;

    // Update the cache immediately, but undo it if the transaction fails
    cache.insert(device.uuid);
    let uuid = device.uuid;
    tx.hooks().on_rollback(move || cache.remove(uuid));

    // ... further operations which might fail and drop the transaction ...

    tx.commit().await?;
    Ok(device)
}
```

For hooks which need to carry state or should be extended from several places
during a transaction's lifetime, there is a second, more powerful API available
through `Transaction::adv_hooks()`. It operates on types implementing the
`TransactionHook` trait (found in `rorm::db::transaction`), which bundles
all three hook points into a single type:

```rust
#[derive(Default)]
struct ChangedUsers {
    uuids: Vec<Uuid>,
}

impl TransactionHook for ChangedUsers {
    fn post_commit(&mut self) {
        for uuid in &self.uuids {
            notify_user_changed(*uuid);
        }
    }
}

// Anywhere with access to the transaction:
tx.adv_hooks()
    .get_or_insert_default::<ChangedUsers>()
    .uuids
    .push(user.uuid);
```

`adv_hooks()` provides `push` to add another hook instance,
`get_or_insert_default` / `get_or_insert_with` to work on a single shared instance of a hook type
and `get_all` for raw access to all registered instances of one type.

!!! note
    Hooks which are added while `pre_commit` hooks are already running are ignored.

## Using a Guard

When either a transaction or the database itself may be passed
to a function, a `TransactionGuard` can be used. It can be
retrieved by an `Executor`, which is implemented by both the
`Transaction` and `Database`. The guard is responsible for
committing the transaction if necessary; since mid-transaction
commits are not supported, this is a no-op when the guard was
created from a transaction and will perform the commit if the
guard was created from a database.

```rust
async fn create_user(username: String, exe: impl Executor<'_>) -> Result<(), rorm::Error> {
    // The guard will either be a new transaction or an
    // existing one, depending on the Executor
    let mut guard = exe.ensure_transaction().await?;

    // It can be used very similar to a normal transaction
    // through `get_transaction`
    rorm::insert(guard.get_transaction(), UserInsert)
        .return_primary_key()
        .single(&UserInsert { username })
        .await?;

    guard.commit().await?;
    Ok(())
}

async fn test_double_creation(db: &Database) -> Result<(), rorm::Error> {
    // Start a transaction
    let mut tx = db.start_transaction().await?;

    // The first call will work but not commit the transaction
    create_user("username".to_string(), &mut tx).await?;

    // The second call will also work but not commit the transaction
    create_user("another username".to_string(), &mut tx).await?;

    // Commit the transaction, since the guard's commit
    // in `create_user` is a no-op there
    tx.commit().await?;

    Ok(())
}
```

### Accessing the underlying transaction

`get_transaction()` returns a `&mut Transaction` for both kinds of guards,
which is all you need to run queries or to register [transaction hooks](#transaction-hooks):

```rust
guard.get_transaction().hooks().post_commit(|| ...);
```

Note that the guard itself doesn't implement `Executor` —
queries always go through `get_transaction()`.

Since `TransactionGuard`'s variants are public, you can also destructure it
in case you need the transaction itself
(e.g. with its full lifetime instead of a reborrow):

```rust
match guard {
    // You took over the transaction — committing it is your job now
    TransactionGuard::Owned(tx) => { ... }
    // The `&mut Transaction` with its original lifetime
    TransactionGuard::Borrowed(tx) => { ... }
}
```

### Forgetting to commit and dropping the guard

Under the hood, a `TransactionGuard` is either an *owned* `Transaction`
(when it was created from a `Database`)
or a *borrowed* `&mut Transaction` (when it was created from an existing transaction).
What happens when the guard is dropped without a `commit` —
be it by mistake, through an early `return`/`?` or an explicit `drop(guard)` —
depends on which of the two it is:

- **Owned**: the guard owns a real transaction, so dropping it behaves like
  dropping a `Transaction`: everything done through the guard is rolled back
  (and `on_rollback` hooks fire).
- **Borrowed**: dropping the guard does **nothing**.
  The operations made through it remain part of the outer transaction
  and it is the caller who decides whether they are committed or rolled back.

For the error path this asymmetry works out nicely:
when your function bails with `?`, its operations are either rolled back (owned)
or the error propagates to the caller who won't commit its transaction (borrowed).

But it also has two consequences to be aware of:

1. Dropping a guard is **not a rollback mechanism**.
   In the borrowed case your operations simply stay pending in the outer transaction —
   if the caller commits it, they will be applied.
   So don't swallow errors after writing through a guard;
   propagate them so the caller knows not to commit.
2. Forgetting `commit` is only a silent no-op in the borrowed case.
   In the owned case your function will "work" but persist nothing —
   which is easy to miss when the function happens to always be tested
   with a transaction passed in.
   `TransactionGuard` is marked `#[must_use]`,
   so the compiler will at least warn you when a guard is not used at all.
