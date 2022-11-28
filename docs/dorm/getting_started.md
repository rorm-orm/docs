# Getting started

DORM is an SQL ORM using the [data mapper pattern](https://en.wikipedia.org/wiki/Data_mapper_pattern) for the [D programming language](https://dlang.org) built on top of [RORM](https://github.com/rorm-orm/rorm).

## Setting up a Project

DORM uses the [DUB](https://code.dlang.org) package manager that usually comes bundled with the D compilers or can be installed through most package managers. To use DORM in your existing project, simply add `dorm` to your dub.json/dub.sdl recipe. This can be done on the command line using

```sh
dub add dorm
# to be able to work with the auto-migrator:
dub add dorm:build-models
```

Then either start by writing code immediately or setup the common file structure like what the project templates do. (See below)

See [Defining Models](#defining-models) to continue.

The `dorm` dependency automatically downloads the [`rorm-lib`](https://github.com/rorm-orm/rorm-lib) library as pre-compiled binary for the target platform as well as a pre-compiled [`rorm-cli`](https://github.com/rorm-orm/rorm-cli) CLI application for the host platform. Due to possible FFI changes for rorm-lib, no automatic updates to the underlying libraries are done. DORM always downloads a pinned version of rorm-lib, previously verified to be working by the maintainers of the DORM project. The library releases are GPG-signed, but the signature is not checked by DORM. File integrity for the correct version is however checked by DORM. (the checksum is stored with the pinned dependency version and link)

!!!info
	The automatic `rorm-lib` download also downloads the `rorm-cli` CLI tool. Both these binaries are stored inside the dependency package directory (e.g. `~/.dub/packages/dorm-v0.1.0/dorm/`) `rorm-cli` is simply wrapped by `dub run dorm`, so it may also be installed globally and used as standalone application. (Using it directly has much faster startup time than using `dub run dorm` and will not trigger dub to do any internet lookup to see if the package is up-to-date)

### Starting a new Project

If you want to start a new project, dorm comes with project templates you can use:

```sh
# DORM with use with vibe-d (web server)
dub init <project-name> -t dorm

# or standalone
dub init <project-name> -t dorm -- base
```

What this does is adding the `dorm` and `dorm:build-models` as dependencies to the project, adds a `models.d` file that contains all the models in a declarative format (see section below) and puts some basic connection code inside of app.d. (See [DORM APIs](#intro-to-dorm-apis))

### Defining Models

A model is the code representation for data stored in an SQL database. In DORM they are defined using a `class` that inherits from `Model`:

```d
/**
	Contains declarative definitions of database models. (maps to SQL tables)
*/
module models;

// imports common things needed in this modelling module. Should not be used
// outside of `module models;` because it adds quite a lot of stuff to the
// global namespace, which might not be useful elsewhere.
import dorm.design;

// Makes it so the models defined in this module can be exported to the
// internal JSON representation that is used by `rorm-cli` / `dub run dorm` to
// automatically create migration files that can be used to initialize the DB.
mixin RegisterModels;

// example model
class User : Model
{
	@Id long id;

	@maxLength(255)
	string username;

	short age;
}

class Car : Model
{
	@maxLength(255)
	string brand;

	@maxLength(255)
	string color;

	@primaryKey
	long serialNo;
}
```

This will then later automatically be used by the auto-migrator to create actual SQL tables and handle application upgrades for users automatically. Some types, for example strings, have mandatory annotations like `@maxLength`. See [Model Declaration](model_declaration.md) for further details.

All Models in DORM should be put into a single file, commonly called `models.d`. This file should be put directly inside inside the source root folder of the project for applications. Libraries defining models for applications should make their models `abstract` classes when intended to make them extensible or define `mixin templates` to use them as-is when requested by the application developer.

### Exporting Models

The `dorm:build-models` DUB dependency, which is added by default using the dorm `dub init` template, will automatically generate a `.models.json` file whenever the project is built from the models definition D file. For this to work, `mixin RegisterModels;` must be included inside the models definition file.

Then, after having built the project at least once, using `dub build` or `dub run`, the _current models_ are automatically contained inside the `.models.json` file. This file can then further be used by the CLI tool (`dub run dorm` or `rorm-cli` when installed as standalone) 

Internally this works adding a `postBuildCommand` when building as an executable and automatically running the app that was built with the argument `--DORM-dump-models-json`. This argument is handled by the `RegisterModels` mixin using a `shared static this()` module constructor. This will be run before any other code that doesn't import the models definition module and exit the app before reaching user-code.

!!!info
	The key takeaway here is that you just need to add `dorm:build-models` as a dependency, run `dub build` or `dub run` once and then you can use the auto-migrator to generate migrations and create the database.

### Generating Migrations

Migrations can be created by running this CLI command:

```sh
dub run dorm -- make-migrations
# or with rorm-cli installed as standalone:
rorm-cli make-migrations
```

This command will read the automatically generated models definition file `.models.json` to compute the required database migrations as TOML files in the directory `migrations/`.

!!!note
	The actual database table creation (SQL) is done in a later step using `rorm-cli migrate`. First we need to setup the database connection information.

Migrations can also be created manually, without the need for the auto-migrator or the `dorm:build-models` sub-package. Head over to the [migration files documentation](../migrations/migration_files.md) for details about the file format.

For more information see [rorm-cli make-migrations](../migrations/makemigrations.md)

### Configuring the Database

DORM and the CLI tool (`rorm-cli` or `dub run dorm`) need to know how to connect to the database for migrations and for use in code. By convention this is stored inside a `database.toml` TOML configuration file inside the app/project folder, e.g. where the `migrations/` folder is present. The TOML file format is very similar to INI.

The database connection configuration consists of a `[Database]` section which contains a `Driver` key and some driver-specific options. A simple example using an SQLite database looks like this:

```toml
# database.toml
[Database]
# Valid driver types are: "MySQL", "Postgres" and "SQLite"
Driver = "SQLite"

# Filename of the database
Filename = "sqlite.db"
```

!!!tip
	If you don't have any `database.toml` file yet, you can create a template one using

	```sh
	dub run dorm -- migrate
	# or with rorm-cli installed:
	rorm-cli migrate
	```

### Creating Tables / Running Migrations

When both a `migrations` folder and the `database.toml` exist, as described in the previous two sections, running

```sh
rorm-cli migrate
```

will create missing tables, alter fields, etc.

For more information see [rorm-cli migrate](../migrations/migrate.md)

## Intro to DORM APIs

The entry-point to all DB-related DORM APIs is the DormDB struct. To use it, you need to first setup the DORM runtime using `mixin SetupDormRuntime;` somewhere in global scope. Then you can create a `DormDB` instance simply by calling its constructor with connection options as parameter:

```d hl_lines="7 16"
// app.d
import dorm.api.db;

import models;

// required to use DORM (use in global scope)
mixin SetupDormRuntime;

void main()
{
	// TODO: parse this from database.toml
	DBConnectOptions options = {
		backend: DBBackend.SQLite,
		name: "database.sqlite3"
	};
	auto db = DormDB(options);
}
```

The `db` variable here can be used to do all kinds of DB-related operations. Most APIs use the [Builder pattern](https://en.wikipedia.org/wiki/Builder_pattern).

The following examples assume the Models definition example from [Defining Models](#defining-models) is used.

### Select / Find / Query

Use the `select` method on the `DormDB` object to start a `SELECT` operation. You can query for Models or [Patches](#patches). This returns a builder and can further use the methods `condition`, `limit`/`offset`/`range` and `orderBy` to manipulate what this select does.

The data can be retrieved using:
- `stream` as a regular D range that can be iterated over, be filtered, etc., which lazily fetches the data from the DB,
- `array` to eagerly fetch all data into a single array,
- `findOne` to fetch one row or error if there is none,
- `findOptional` to fetch one row or return null if there is none.

```d
// SELECT id, username, age FROM user
User[] allUsers = db.select!User.all;

// SELECT id, username, age FROM user LIMIT 1
User firstUser = db.select!User.findOne;

// SELECT id, username, age FROM user WHERE age = 0
auto usersWithZeroAge = db.select!User
	.condition(u => u.age.equals(0))
	.stream;
```

### Insert

Use the `insert` method on the `DormDB` object to perform an `INSERT` operation. You can pass a regular Model, a list of Models, a [Patch](#patches) or a list of Patches into this.

```d
Car car = new Car();
car.brand = "Toyota";
car.color = "pearl white";
car.serialNo = 0;
// INSERT OR ABORT INTO car (brand, color, serial_no) VALUES ('Toyota', 'pearl white', 0);
db.insert(car);
```

### Update

Use the `update` method on the `DormDB` object to start an `UPDATE` operation. You pass a Model class into the template parameter. This returns a builder struct on which you can call the methods `condition` and `set!"fieldName"(value)` or `set(Patch)` to manipulate what will be done. When ready to send the update, call `.await` to perform the update.

```d
// UPDATE OR ABORT user SET username = 'newUsername'
db.update!User
	.set!"username"("newUsername")
	.await;

// UPDATE OR ABORT user SET username = 'boss', age = 42 WHERE id = 1
db.update!User
	.set!"username"("boss")
	.set!"age"(42)
	.condition(u => u.id.equals(1))
	.await;
```

### Remove / Delete

Use the `remove` method on the `DormDB` object to either delete an instance from the DB or start a `DELETE` operation. When passing in a Model instance or a [Patch](#patches) with the primary key field, it will be deleted immediately. When passing in a Model type as template parameter, a builder is returned. On this builder all methods immediately finish right now, so it's not strictly a builder, but rather groups together the different delete methods. You can call `byCondition`, `single` or `all` to actually perform the delete.

```d
// SELECT brand, color, serial_no FROM car
auto car = db.select!Car.findOptional;
if (!car.isNull)
{
	// DELETE FROM car WHERE serial_no = ?
	db.remove(car.get);
}

// DELETE FROM car WHERE serial_no > 1337
db.remove!Car
	.byCondition(c => c.serialNo.greaterThan(1337));

// DELETE FROM car
auto affectedRows = db.remove!Car.all();
```

## Patches

Often you don't want to get every available column when querying data and often you don't want to provide every defined column when inserting data. To avoid clashing with the default values or potentially using wrong values, DORM decides to always use all provided columns inside the passed in object. You can however pass in partial objects, so called "Patches", which can take a few different forms:
- usually they are `structs`, annotated with `@DormPatch!MyModel`, containing just the fields that you want to use (without annotations)
- they can also simply be a `struct` immediately inside a `Model` class (called implicit patches) - these are not annotated with `DormPatch` and contain the definition of all fields. To avoid code duplication, you use them inside the class as members and use `@embedded` to embed all members as fields.
- where applicable, tuples defining the requested field names along with the Model type as template parameter can be used as patches as well.

Patch structs may be defined anywhere (can be put inside the `models` module or just before using them inside a method)

```d
@DormPatch!User
struct UserNew
{
	string username;
	short age;
}
```

This UserNew is now a patch for the User model and can be used in a variety of places to only specify a limit set of columns.

```d
// INSERT OR ROLLBACK INTO user (username, age) VALUES ("foo", 42), ("bar", 0), ("baz", 1337)
db.insert([
	UserNew("foo", 42),
	UserNew("bar", 0),
	UserNew("baz", 1337),
]);
```

When using `insert`, only the fields that are defined in the patch as well as `@constructValue` annotated fields will be included in the actual SQL insert statement. Everything else is checked by DORM to be done by the database / table, which would previously be created using the rorm-cli migrator. Forgetting fields in an insert will result in a compile time error if these fields don't have any attributes that make them auto-generated.

When using `select` to query rows, only the specified columns will be included in the result, which can speed up query performance.

Patches can be used inside `update.set` to set all fields defined in the patch.

Patches containing the primary key can be used as parameter to the `remove` method.
