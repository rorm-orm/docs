# Linter

The linter validates the [internal model representation](internal_model_representation.md)
before any migrations are generated from it.
It runs automatically as part of [`rorm-cli make-migrations`](makemigrations.md) —
if any check fails, no migration files are written.

## Checks

**Model level:**

- Model names must be unique.
- A model must contain at least one field.
- Every model needs a field with the `PrimaryKey` annotation.

**Naming** (applies to model and field names):

- Names must not be empty and may only consist of `[a-zA-Z0-9_]`.
- Names must not start or end with `_` (reserved for internal use).
- Names must not consist of numerics only.
- Model names must not start with `sqlite_` (reserved by SQLite).

**Field level:**

- Field names must be unique within their model.
- The `AutoIncrement` annotation may only be used once per model
  and requires the `PrimaryKey` annotation on the same field.
- Annotations must not conflict with each other
  (e.g. `AutoCreateTime` combined with `AutoIncrement`).

!!! note
    Most of these rules are already enforced at compile time by the
    `derive(Model)` macro. The linter catches the rest as well as
    handwritten or manipulated `.models.json` files.
