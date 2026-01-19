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

    guard.commit()?;
    Ok(())
}

async fn test_double_creation(db: &Database) -> Result<(), rorm:Error> {
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
