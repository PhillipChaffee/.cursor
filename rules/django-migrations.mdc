---
description: Never hand-write migrations; always use the ORM generator.
alwaysApply: true
---

# Database Migrations

Never write or edit migration files by hand. Always use the ORM's migration generator so the framework tracks state correctly and produces deterministic output.

## Django

```bash
python manage.py makemigrations
```

For custom data migrations, generate an empty migration first, then fill in the operation logic:

```bash
python manage.py makemigrations <app_name> --empty -n <descriptive_name>
```
