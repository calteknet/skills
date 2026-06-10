# Odoo Version-Specific Migration Notes

Consult this file when the user's source or target version involves any of the scenarios below.

---

## account.move (Invoices) — v12 → v13+

This is the most breaking change in Odoo history for data migration.

| v12 and earlier | v13+ |
|---|---|
| `account.invoice` model | `account.move` model |
| `account.invoice.line` | `account.move.line` |
| `state: open` | `state: posted` |
| `type: out_invoice` | `move_type: out_invoice` |
| `date_invoice` | `invoice_date` |
| `date_due` | `invoice_date_due` |
| `payment_term_id` on invoice | same, but reconciliation changed |

**Action**: If migrating from v12 or earlier to v13+, always generate a dedicated invoice migration script. The model rename means a plain field map is not enough — the target model itself changes.

---

## product.template vs product.product — all versions

- `product.template` is the "template" (shared attributes across variants).
- `product.product` is a specific variant.
- In v13+, the split is more enforced. Migrating `product.template` records is usually correct for most use cases.
- If the user has product variants, warn them that `product.product` IDs will not match after migration and any `product_id` many2one on other models (e.g., `sale.order.line`) must be re-resolved.

---

## sale.order — v13 → v14+

- `amount_by_group` field removed in v14 (computed, no migration needed)
- `picking_policy` values unchanged

---

## sale.order — v16 → v17

- No breaking field changes, but `website_id` relation may differ if e-commerce is installed
- `commitment_date` renamed to `expected_date` in some builds — inspect with `fields_get` first

---

## res.partner — all versions

- `property_*` fields (e.g., `property_account_receivable_id`) are **company-dependent** fields. They are stored in `ir.property`, not directly on `res.partner`. They cannot be exported/imported as regular fields.
  - **Action**: Warn the user. These must be set via dedicated `ir.property` migration or reconfigured manually.
- `child_ids` (contacts) should be migrated after the parent partner to preserve hierarchy.

---

## res.users — all versions

- Do **not** attempt to migrate `res.users` via API — password hashes, OAuth tokens, and session data are instance-specific.
- **Action**: Tell the user to recreate users manually or use the Odoo import UI with a CSV, then reset passwords.

---

## Migrating from Odoo 8–11 (very old)

- The XML-RPC endpoint path changed. v8–10 used `/xmlrpc/` (no `/2/`). Check with the user.
- The `fields_get` call still works but the `ir.model.fields` approach is more reliable for very old versions.
- Many models were significantly restructured between v10 and v13. Recommend a field-by-field review rather than bulk mapping.

---

## Odoo Online / Odoo.sh

- The DB name is the subdomain: `https://mycompany.odoo.com` → db = `mycompany`
- API keys are required (passwords are disabled for external API access on Odoo Online)
- `ir.attachment` binary migration is blocked on Odoo Online — warn the user

---

## General Many2one Resolution Strategy

When a source record has `partner_id = [42, "Acme Corp"]`, the ID `42` is meaningless on target.

Resolution options (in order of preference):
1. **External ID** (`ir.model.data`): most reliable if records were imported with XML IDs
2. **`ref` field** (internal reference): works for products, partners with refs set
3. **`name` match**: fragile but usable for unique names (partners, countries, etc.)
4. **Pre-migration map**: export a source→target ID mapping file before the main migration

Always surface this to the user and let them choose.
