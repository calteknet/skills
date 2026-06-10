---
name: odoo-migration
description: >
  Use this skill whenever the user wants to migrate data between Odoo instances or versions using JSON-RPC.
  Triggers include: migrating Odoo data, syncing records between Odoo environments, exporting/importing
  Odoo models, upgrading Odoo data from one version to another, or any task involving xmlrpc/jsonrpc
  connections to Odoo. Also use when the user asks to inspect Odoo model fields, compare schemas between
  two Odoo instances, or generate a field mapping between versions. If the user mentions Odoo and data
  in the same breath, consult this skill.
---

# Odoo Data Migration via JSON-RPC

Helps Claude guide users through migrating data between any two Odoo instances (any version combination)
using Odoo's JSON-RPC / XML-RPC API. The workflow is:

**Connect → Inspect fields → Export → Transform (auto-map fields) → Dry-run → Import**

---

## Step 0 — Gather Inputs

Before writing any code, collect these from the user if not already provided:

| Input | Variable name | Example |
|---|---|---|
| Source Odoo URL | `src_url` | `https://old.mycompany.com` |
| Source version | `src_version` | `14` |
| Source DB name | `src_db` | `mydb_v14` |
| Source username | `src_user` | `admin` |
| Source password/API key | `src_pass` | *(ask user to supply at runtime)* |
| Target Odoo URL | `dst_url` | `https://new.mycompany.com` |
| Target version | `dst_version` | `17` |
| Target DB name | `dst_db` | `mydb_v17` |
| Target username | `dst_user` | `admin` |
| Target password/API key | `dst_pass` | *(ask user to supply at runtime)* |
| Model(s) to migrate | `models` | `['res.partner', 'sale.order']` |
| Domain filter (optional) | `domain` | `[('active','=',True)]` — default `[]` |
| Batch size (optional) | `batch_size` | `100` (default) |

**Never hard-code passwords.** Always prompt the user to set them as environment variables
(`SRC_PASS`, `DST_PASS`) or pass them via a config file excluded from version control.

---

## Step 1 — Explain the Approach

Before generating code, give the user a plain-English summary of what will happen:

1. Connect to both instances and authenticate via XML-RPC (`/xmlrpc/2/common` + `/xmlrpc/2/object`).
2. Introspect the model's fields on both sides using `fields_get()`.
3. Auto-detect field differences (renamed, removed, type-changed) and produce a suggested mapping dict.
4. User reviews and edits the mapping.
5. Export records from source using `search_read()` with the domain filter, in batches.
6. Transform each batch using the mapping.
7. **Dry-run**: validate transformed records against the target schema without writing.
8. On user approval, import via `create()` or `write()` (upsert logic if an external ID or matching key is provided).
9. Log every success, skip, and failure to a JSON log file.

---

## Step 2 — Generate the Connection + Inspection Script

See `references/connection.md` for the boilerplate connection class.

Produce `inspect_fields.py` first. This script:
- Connects to both instances
- Calls `fields_get()` on the chosen model on each side
- Prints a side-by-side diff of field names, types, and `required` flags
- Outputs `field_diff.json` with three sections: `only_in_source`, `only_in_target`, `type_changed`

Show the user how to run it:
```bash
python inspect_fields.py \
  --src-url $SRC_URL --src-db $SRC_DB --src-user $SRC_USER \
  --dst-url $DST_URL --dst-db $DST_DB --dst-user $DST_USER \
  --model sale.order
```

---

## Step 3 — Auto-generate Field Mapping

After the user runs `inspect_fields.py` and shares `field_diff.json`, generate a `mapping.py` file
containing a `FIELD_MAP` dict. Use these heuristics for auto-suggestions:

| Situation | Suggested action |
|---|---|
| Field exists on both sides, same type | Map 1:1 |
| Field name changed (fuzzy match ≥ 0.8) | Map with a comment `# renamed from X` |
| Field type changed (e.g. char → many2one) | Flag as `"NEEDS_TRANSFORM"` with a comment |
| Field only in source | Map to `None` (drop it) with a comment |
| Field only in target + required | Flag as `"NEEDS_DEFAULT"` |
| Relational field (many2one, one2many) | Add a note about resolving external IDs |

Present the mapping to the user and ask them to confirm or edit before proceeding.

---

## Step 4 — Generate the Migration Script

See `references/migration_script.md` for the full template.

The script (`migrate.py`) must support these CLI flags:

```
--dry-run          Validate and log without writing to target
--apply            Actually write to target (requires --dry-run to have passed)
--model            Odoo model technical name (e.g. sale.order)
--domain           Optional filter, e.g. "[('state','=','sale')]"
--batch-size       Records per API call (default 100)
--mapping-file     Path to mapping.py or mapping.json
--log-file         Output path for migration log (default: migration_YYYYMMDD.json)
--skip-existing    Skip records that already exist on target (matched by `external_id` or key field)
--upsert-key       Field name to use for upsert matching (e.g. `ref` or `email`)
```

---

## Step 5 — Dry-Run Protocol

Explain to the user what the dry run checks:
- All required target fields are populated after transform
- Many2one values resolve to existing IDs on the target
- Field types match (no string going into an integer field, etc.)
- No duplicate records if `--skip-existing` is set

Dry-run output is a JSON report:
```json
{
  "model": "sale.order",
  "total_records": 312,
  "would_create": 290,
  "would_skip": 20,
  "errors": [
    {"record_id": 45, "field": "partner_id", "issue": "ID 45 not found on target"}
  ]
}
```

Tell the user: **only proceed to `--apply` when `errors` is empty or all errors are acceptable skips.**

---

## Step 6 — Import + Logging

On `--apply`:
- Write in batches using `execute_kw('model', 'create', [batch])`
- For upserts: `search` for the upsert key first, then `write` if found, else `create`
- Append to the log file after each batch (never overwrite — append so partial runs are recoverable)

Log format per record:
```json
{"source_id": 123, "target_id": 456, "status": "created", "model": "sale.order"}
{"source_id": 78,  "target_id": null, "status": "skipped", "reason": "already exists"}
{"source_id": 99,  "target_id": null, "status": "error",   "reason": "partner_id not found"}
```

---

## Version-Specific Notes

Read `references/version_notes.md` before generating code if the user mentions any of these:
- Migrating invoices (`account.move`) — significant changes v12→v13+
- Migrating products — `product.template` vs `product.product` split varies by version
- Any migration touching v13 or earlier (old API style differs)
- Migrating users or access rights (security model changes)

---

## Common Pitfalls — Always Mention These

1. **Relational IDs don't transfer.** A `partner_id = 42` on source is meaningless on target. Always resolve by name/ref/external ID.
2. **`id` field is read-only.** Never include `id` in the create payload; use `external_id` or a natural key instead.
3. **Attachments and binary fields** are expensive — warn the user and handle separately.
4. **`active = False` records** are excluded by default from `search_read`. Add `('active', 'in', [True, False])` to the domain if needed.
5. **Required computed fields** may need to be set in a specific order (e.g. `sale.order` lines before header confirmation).
6. **API rate limits / timeouts** — use `batch_size=50` for complex models, `100–500` for simple ones.
