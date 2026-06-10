# Odoo Connection Boilerplate

Use this as the base connection class in all generated scripts.

```python
import xmlrpc.client
import os

class OdooClient:
    """
    Thin wrapper around Odoo's XML-RPC API.
    Works with Odoo 12+ (common endpoint + object endpoint).
    """

    def __init__(self, url: str, db: str, username: str, password: str):
        self.url = url.rstrip("/")
        self.db = db
        self.username = username
        self.password = password
        self.uid = None

        self._common = xmlrpc.client.ServerProxy(f"{self.url}/xmlrpc/2/common")
        self._models = xmlrpc.client.ServerProxy(f"{self.url}/xmlrpc/2/object")

    def authenticate(self) -> int:
        """Authenticate and cache UID. Raises on failure."""
        version = self._common.version()
        print(f"  Connected to {self.url} — Odoo {version.get('server_version', '?')}")
        uid = self._common.authenticate(self.db, self.username, self.password, {})
        if not uid:
            raise PermissionError(
                f"Authentication failed for user '{self.username}' on db '{self.db}' at {self.url}"
            )
        self.uid = uid
        print(f"  Authenticated as UID {uid}")
        return uid

    def execute(self, model: str, method: str, *args, **kwargs):
        """Raw execute_kw wrapper."""
        return self._models.execute_kw(
            self.db, self.uid, self.password,
            model, method, list(args), kwargs
        )

    def fields_get(self, model: str, attributes=None) -> dict:
        """Return field definitions for a model."""
        attrs = attributes or ["string", "type", "required", "relation", "readonly"]
        return self.execute(model, "fields_get", [], {"attributes": attrs})

    def search_read(self, model: str, domain: list, fields: list, batch_size=100):
        """
        Generator that yields records in batches.
        Usage:
            for batch in client.search_read('sale.order', [], ['name', 'partner_id']):
                process(batch)
        """
        offset = 0
        while True:
            batch = self.execute(
                model, "search_read",
                [domain],
                {"fields": fields, "limit": batch_size, "offset": offset}
            )
            if not batch:
                break
            yield batch
            if len(batch) < batch_size:
                break
            offset += batch_size

    def create(self, model: str, vals_list: list) -> list:
        """Create records. Returns list of new IDs."""
        return self.execute(model, "create", [vals_list])

    def write(self, model: str, ids: list, vals: dict) -> bool:
        return self.execute(model, "write", [ids, vals])

    def search(self, model: str, domain: list, fields=None) -> list:
        if fields:
            return self.execute(model, "search_read", [domain], {"fields": fields})
        return self.execute(model, "search", [domain])


def connect_from_env(prefix: str) -> OdooClient:
    """
    Build an OdooClient from environment variables.
    prefix = 'SRC' → reads SRC_URL, SRC_DB, SRC_USER, SRC_PASS
    prefix = 'DST' → reads DST_URL, DST_DB, DST_USER, DST_PASS
    """
    url  = os.environ[f"{prefix}_URL"]
    db   = os.environ[f"{prefix}_DB"]
    user = os.environ[f"{prefix}_USER"]
    pw   = os.environ[f"{prefix}_PASS"]
    client = OdooClient(url, db, user, pw)
    client.authenticate()
    return client
```

## Environment variable setup (recommended)

```bash
# .env  — add to .gitignore!
export SRC_URL="https://old.mycompany.com"
export SRC_DB="mydb_v14"
export SRC_USER="admin"
export SRC_PASS="supersecret"

export DST_URL="https://new.mycompany.com"
export DST_DB="mydb_v17"
export DST_USER="admin"
export DST_PASS="anothersecret"

source .env
```

## Notes

- The same class works for Odoo 12 through 17+. For Odoo 8–11, the endpoint is `/xmlrpc/2` but the auth call differs slightly — flag this to the user if they mention very old versions.
- For Odoo.sh or Odoo Online, the URL is typically `https://<instance>.odoo.com` and the DB name is the same as the subdomain.
- API keys (Settings → Technical → API Keys) are preferred over passwords for production use. They go in the `password` field of the auth call.
