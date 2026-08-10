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

To declare a standard index:
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

To declare a composite index, add a `name` shared by its fields.
The index will contain the fields in their order of occurrence,
which can be overwritten with the optional `priority` property:

```rust
use rorm::prelude::*;

#[derive(Model)]
struct User {
    .. // fields missing to be functional
    
    #[rorm(index(name = "name", priority = 2), max_length = 255)]
    first_name: String,

    #[rorm(index(name = "name", priority = 1), max_length = 255)]
    last_name: String,
}
```

Fields without a `priority` are treated as if they had `priority = 0`.

---

Unlike columns, indexes don't live in their table's namespace.
Their names have to be unique across the whole database,
which is why the migrator prefixes them with their table's name:

| Declaration                | Index in the database |
|----------------------------|-----------------------|
| `#[rorm(index)]`           | `<table>_<column>_idx` |
| `#[rorm(index(name = ".."))]` | `<table>_<name>_idx`  |

`make-migrations` reports an error if two indexes end up with the same name.

!!!note
    Adding or removing an `index` does not touch the column itself,
    so the migration keeps the data it holds.

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

With this annotation, rorm is instructed to enforce on database level
that the field's values are unique across all rows.

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

## Noteworthy field types

Besides the primitive types (`bool`, `i16`, `i32`, `i64`, `f32`, `f64`, `String`, `Vec<u8>`),
their `Option`s, `chrono`/`time` types, `Uuid` and `Url`,
rorm ships some field types worth pointing out.
(Relations' types are covered in [relations](./relations.md).)

### `MaxStr`

`MaxStr<N>` is a `String` whose maximum length `N` is part of the type:

```rust
use rorm::fields::types::MaxStr;
use rorm::prelude::*;

#[derive(Model)]
struct User {
    #[rorm(id)]
    id: i64,

    /// No `#[rorm(max_length = ...)]` required,
    /// the type already carries it.
    username: MaxStr<255>,
}
```

While a plain `String` field relies on the database to reject too long strings —
resulting in a rather opaque `rorm::Error` at some later point —
`MaxStr` enforces the length upon construction.
This moves the check to the place where the string enters your program,
for example during deserialization of an api request,
where the error can be handled much more gracefully.

### `Json`

`Json<T>` stores any `T: Serialize + DeserializeOwned` by serializing it to json.
This is a convenient escape hatch for structured data
which doesn't need to be queried by its fields:

```rust
use rorm::fields::types::Json;
use rorm::prelude::*;

#[derive(Model)]
struct Session {
    #[rorm(id)]
    id: i64,

    data: Json<HashMap<String, String>>,
}
```

### Postgres-only types

When [committing to postgres](./getting_started.md#cargo-features)
with the `postgres-only` feature, some types from the ecosystem
become usable as fields directly:

- `ipnetwork::IpNetwork` — an IP address with subnet mask, stored as postgres' `inet`
- `mac_address::MacAddress` — stored as postgres' `macaddr`
- `bit_vec::BitVec` — stored as postgres' `varbit`

```rust
use ipnetwork::IpNetwork;
use rorm::prelude::*;

#[derive(Model)]
struct Device {
    #[rorm(primary_key)]
    uuid: Uuid,

    ip: Option<IpNetwork>,
}
```

### `PhantomData`

`PhantomData<T>` (for any `T: 'static`) implements `FieldType` with zero columns.
It can be used to attach compile-time only information to a model
without storing anything in the database,
which comes in handy when working with [generic models](#generic-models).

## Experimental features

The following features are opt-in on a per-model basis and considered experimental:
their API may change in future releases without the usual deprecation cycle.

### Unregistered models

Normally, every `#[derive(Model)]` registers the model in a global list
which `rorm::write_models` uses to generate the [internal model representation](../migrations/internal_model_representation.md)
for migrations.

The `experimental_unregistered` attribute disables this auto-registration:

```rust
#[derive(Model)]
#[rorm(experimental_unregistered)]
struct Draft {
    #[rorm(id)]
    id: i64,
}
```

An unregistered model is invisible to `make-migrations` until it is
registered explicitly using the `register_model!` macro:

```rust
rorm::register_model!(Draft);
```

This split is useful when a model is defined generically (e.g. by a library)
and only concrete usages should produce database tables.

### Generic models

The `experimental_generics` attribute allows a model to take generic type parameters.
Every generic parameter used as a field has to implement `rorm::fields::traits::FieldType`:

```rust
#[derive(Model)]
#[rorm(experimental_generics)]
struct Container<T: rorm::fields::traits::FieldType> {
    #[rorm(id)]
    id: i64,

    value: T,
}
```

Since a generic model can't know its concrete instantiations,
`experimental_generics` implies `experimental_unregistered`:
the concrete instantiation you're using
has to be registered explicitly:

```rust
rorm::register_model!(Container<String>);
```

!!!warning
    The table name is derived from the struct's name (or its `rename` annotation),
    so all instantiations of a generic model share it.
    Register only one instantiation per generic model.

The typical use case is a library providing a model
which is generic over some application-defined type:
the library declares the generic model,
the application picks the concrete type and registers it.
