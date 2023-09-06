# A full example

The following snippet illustrates how to read the previously mentioned config
file to connect to the database, query all users and update, insert and
delete some. Note that this is just the most basic functionality, but
`rorm` will provide a lot more functionality in the future:

```rust
use std::fs::read_to_string;

use rorm::{delete, insert, update, query, Database};
use rorm::config::DatabaseConfig;
use serde::Deserialize;

#[derive(Deserialize, Debug)]
#[serde(rename_all = "PascalCase")]
pub struct ConfigFile {
    pub database: DatabaseConfig,
    // more configuration parameters depending on your application ...
}

#[rorm::rorm_main]
#[tokio::main]
async fn main() {
    // Read the config from a TOML file
    let path = "config.toml";
    let db_conf_file = toml::from_str::<ConfigFile>(
        read_to_string(&path)
            .expect("File read error")
            .as_str(),
    )
    .expect("Couldn't deserialize configuration file");

    // Connect to the database to get the database handle using the TOML configuration
    let db = Database::connect(DatabaseConfiguration {
        driver: db_conf_file.database.driver,
        min_connections: 1,
        max_connections: 1,
    })
    .await
    .expect("error connecting to the database");

    // Query all users from the database
    for user in query!(&db, User)
        .all()
        .await
        .expect("querying failed")
    {
        println!(
            "User {} '{}' is {} years old",
            user.id, user.username, user.age
        )
    }

    // Add three new users to the database
    insert!(&db, UserNew)
        .bulk(&[
            UserNew {
                username: "foo".to_string(),
                age: 42,
            },
            UserNew {
                username: "bar".to_string(),
                age: 0,
            },
            UserNew {
                username: "baz".to_string(),
                age: 1337,
            },
        ])
        .await;

    // Update the second user by increasing its age
    let all_users = query!(&db, User).all().await.expect("error");
    update!(&db, User)
        .set(User::FIELDS.age, all_users[2].age + 1)
        .condition(User::FIELDS.id.equals(all_users[2].id))
        .await
        .expect("error");

    // Delete some user with age 69 or older than 100 years
    let zero_aged_user = query!(&db, User)
        .condition(or!(
            User::FIELDS.age.greater(100),
            User::F.age.equals(69)
        ))
        .one()
        .await
        .expect("error");
    delete!(&db, User).single(&zero_aged_user).await;
}
```
