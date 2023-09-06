
# CRUD

To perform database operations, the handle `db` is given to a macro,
which expands to a builder statement. In order to add information like
conditions to an operation, the methods of that builder can be used.

Consider this preamble for the following snippets:

```rust
use rorm::{delete, insert, query, update, Database};

#[derive(Clone, Model, Debug)]
pub struct Car {
    #[rorm(max_length = 255)]
    pub(crate) brand: String,

    #[rorm(max_length = 255)]
    pub(crate) color: String,

    #[rorm(primary_key)]
    pub(crate) serial_no: i64,
}
```

## Query

Use the `query!` macro to start a `SELECT` operation. It can be chained with an
optional `.condition()` and collected with `.all()`, `.one()` or `.optional()`.

```rust
async fn query(db: &Database) {
    // SELECT  id, username, age FROM user ;
    let all_users = query!(db, User).all().await.unwrap();

    // SELECT  id, username, age FROM user ;
    let first_user = query!(db, User).one().await.unwrap();

    // SELECT  id, username, age FROM user WHERE (age = 0);
    let one_user_with_age_zero = query!(db, User)
        .condition(User::FIELDS.age.equals(0))
        .one()
        .await
        .unwrap();

    // SELECT  id, username, age FROM user WHERE (age > 100);
    let users_over_100 = query!(db, User)
        .condition(User::FIELDS.age.greater(100))
        .all()
        .await
        .unwrap();
}
```

## Insert

Use the `insert!` macro to start an `INSERT` operation. It can be chained with
`.single()` to add one instance or `.bulk()` to add a slice of instances.

```rust
async fn insert(db: &Database) {
    // INSERT OR ABORT INTO car (brand, color, serial_no) VALUES ('VW', 'black', 0);
    insert!(db, Car)
        .single(&Car {
            brand: "VW".to_string(),
            color: "black".to_string(),
            serial_no: 0,
        })
        .await
        .unwrap();

    // INSERT OR ROLLBACK INTO car (brand, color, serial_no) VALUES (?, ?, ?), (?, ?, ?), (?, ?, ?), ...;
    let mut cars = vec![];
    for i in 1..65536 {
        cars.push(Car {
            brand: "VW".to_string(),
            color: "red".to_string(),
            serial_no: i,
        })
    }
    insert!(db, Car).bulk(&cars).await.unwrap();
}
```

## Update

Use the `update!` macro to start an `UPDATE` operation. Chain it with
one or more `.set()` calls to update the column values as well as an
optional filter with `.condition()`.

```rust
async fn update(db: &Database) {
    // UPDATE OR ABORT user SET username = 'user';
    update!(db, User).set(User::FIELDS.username, "user").await;

    // UPDATE OR ABORT user SET username = 'boss', age = 42 WHERE (id = 1);
    update!(db, User)
        .set(User::FIELDS.username, "boss")
        .set(User::FIELDS.age, 42)
        .condition(User::FIELDS.id.equals(1))
        .await;
}
```

## Delete

Use the `delete!` macro to start a `DELETE` operation. Either chain it with
`.all()` to clear the whole table, with `.single()` to delete a model
instance or with `.condition()` to set an explicit filter for the deletion.

```rust
async fn delete(db: &Database) {
    // DELETE FROM car ;
    delete!(db, Car).all().await.expect("failed to delete all");

    // SELECT  brand, color, serial_no FROM car ;
    // DELETE FROM car WHERE (serial_no = ?) ;
    if let Some(car) = query!(db, Car).optional().await.unwrap() {
        delete!(db, Car)
            .single(&car)
            .await
            .expect("failed to delete one");
    }

    // DELETE FROM car WHERE (serial_no > 1337) ;
    delete!(db, Car)
        .condition(Car::FIELDS.serial_no.greater(1337))
        .await
        .expect("failed to delete some");
}
```
