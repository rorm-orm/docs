## `derive(Model)` macro

At the heart of the orm is the derive macro turning a rust struct into a db model.

It uses `#[rorm(..)]` attributes on fields to provide additional information.

For example a database needs to know how much space a string is expected to occupy:
```rust
use rorm::prelude::*;

#[derive(Model)]
struct User {
	.. // fields missing to be functional

	#[rorm(max_length = 255)]
	username: String,
}
```

These attributes can be stacked on a field or multiple annotations can be set in a single attribute:
```rust
use rorm::prelude::*;

#[derive(Model)]
struct User {
	.. // fields missing to be functional

	#[rorm(unique)]
	#[rorm(max_length = 255)]
	username: String,

	#[rorm(max_length = 255, unique)]
	email: String,
}
```

## Annotations
Annotations are the extra information defined in the `#[rorm(..)]` attributes.
Some of them map directly to SQL annotations while other are purely for orm purposes.

### `autoincrement`
The `autoincrement` annotation instructs the database to populate the
field using a running counter when creating the rows of this model.

```rust
use rorm::prelude::*;

#[derive(Model)]
struct Order {
	.. // fields missing to be functional

	#[rorm(autoincrement)]
	order_number: i64,
}
```

### `auto_create_time` and `auto_update_time`
You can utilize the annotations `auto_create_time` and `auto_update_time` to
automatically set the current time on creation or on update of the model
to the annotated field.

```rust
use chrono::{DateTime, Utc};
use rorm::prelude::*;

#[derive(Model)]
struct File {
	.. // fields missing to be functional

	#[rorm(auto_create_time)]
	created: DateTime<Utc>,

	#[rorm(auto_update_time)]
	modified: DateTime<Utc>,
}
```

### `default`
A default value to populate this field with, if a new model instance
is created without mentioning this field. Note that you need a patch
struct to utilize its advantage in the Rust code.

```rust
use rorm::prelude::*;

#[derive(Model)]
struct User {
	.. // fields missing to be functional

	#[rorm(default = false)]
	is_admin: bool,
}
```

### `id`
Shorthand for both `primary_key` and `autoincrement`

```rust
use rorm::prelude::*;

#[derive(Model)]
struct User {
	#[rorm(id)]
	id: i64,
}
```

### `index`

rorm currently supports standard indexes and composite indexes.

To create a standard index:
```rust
use rorm::prelude::*;

#[derive(Model)]
struct User {
    .. // fields missing to be functional

    #[rorm(index, max_length = 255)]
    name: String,
}
```

---

To create a composite index, include the `name` property:
```rust
use rorm::prelude::*;

#[derive(Model)]
struct User {
    .. // fields missing to be functional
    
    #[rorm(index("name"), max_length = 255)]
    first_name: String,


    #[rorm(index("name"), max_length = 255)]
    last_name: String,
}
```

!!!info
    This will create a composite index in the order of occurrences of the 
    index, so for the example above: `first_name, last_name`.

    To change the order in the index creation, use the `priority` property:

    ```rust
    use rorm::prelude::*;

    #[derive(Model)]
    struct User {
    .. // fields missing to be functional
    
        #[rorm(index("name", priority = 2), max_length = 255)]
        first_name: String,
    
    
        #[rorm(index("name", priority = 1), max_length = 255)]
        last_name: String,
    }
    ```

### `max_length`
Specify the maximum length a String can have. This is required for every string.

```rust
use rorm::prelude::*;

#[derive(Model)]
struct User {
	.. // fields missing to be functional

	#[rorm(max_length = 255)]
	username: String,
}
```

### `on_delete`, `on_update`
These annotations specify the policy for the relations that are applied on
either delete or update.

Possible values are:
- `Restrict`
- `Cascade`
- `SetNull`
- `SetDefault`

```rust
use rorm::prelude::*;

#[derive(Model)]
struct Post {
    #[rorm(id)]
    id: i64,
    
    #[rorm(on_delete = "Cascade", on_update = "SetNull")]
    user: Option<ForeignModel<User>>,
}

#[derive(Model)]
struct User {
	#[rorm(primary_key)]
	username: String,
}
```

### `primary_key`
Marks a field as primary key in the database. 
As primary keys are by default unique, the `unique` annotation is not available
for fields that have the `primary_key` annotation set.

```rust
use rorm::prelude::*;

#[derive(Model)]
struct User {
	#[rorm(primary_key)]
	username: String,
}
```

### `rename`
With this annotation, rorm is instructed to use the provided field name
instead of real name.

```rust
use rorm::prelude::*;

#[derive(Model)]
struct User {
    .. // fields missing to be functional
    
	#[rorm(rename = "user_name")]
	username: String,
}
```

### `unique`

With this annotation, rorm is instructed to use the provided field name
instead of real name.

```rust
use rorm::prelude::*;

#[derive(Model)]
struct User {
    .. // fields missing to be functional
    
	#[rorm(unique)]
	chosen_number: i16,
}
```

!!!info
    This annotation is not required on fields with the `primary_key` 
    annotation, as `primary_key` enforces uniqueness on database level.
