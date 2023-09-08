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
    let is_taken = query!(&mut tx, (User::F.id,))
        .condition(User::F.username.equals(&username))
        .optional()
        .await?
        .is_some();
    
    if is_taken {
        return Err("Username is already taken".to_string());
    }
    
    // Insert the new user
    let id = insert!(&mut tx, UserInsert)
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
