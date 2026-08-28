---
paths:
  - "backend/app/core/db/*_orm.py"
  - "backend/app/core/db/*_db.py"
description: ORM serialization rules — prevent JSON "null" strings in SQLite
---

# ORM Serialization — JSON column NULL handling

SQLAlchemy's JSON column stores Python `None` as the JSON text `"null"` in SQLite by default, **not** SQL NULL. Two rules prevent this:

- **Always use `JSON(none_as_null=True)`** on nullable JSON column definitions. This makes Python `None` map to SQL NULL at the type level.
  - BAD: `mapped_column(JSON, nullable=True)`
  - GOOD: `mapped_column(JSON(none_as_null=True), nullable=True)`
- **Always use `exclude_none=True`** in `model_dump()` calls targeting JSON columns. Without it, `None` fields inside the serialized dict become JSON `"null"` values.
  - BAD: `orm.field = model.model_dump(mode="json")`
  - GOOD: `orm.field = model.model_dump(mode="json", exclude_none=True)`

These apply to every model persisted through a JSON column.
