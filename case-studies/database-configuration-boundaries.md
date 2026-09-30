# Removing database-path authority from a browser settings API

A browser settings API accepted a database pathname even though its ordinary task was to update application preferences. Separately, demo startup could place its database at a predictable name in a shared temporary directory. Two small source changes removed unnecessary path-selection authority and isolated the demo fallback from other local users.

The reviewed code was the application's historical Python/FastAPI implementation. The current backend uses Rust.

## Boundary 1: preferences versus filesystem destinations

The configuration update model originally included `db_path` as a caller-settable string. Validated fields were written to configuration, and database-path changes were among the settings requiring restart. A caller able to reach that management API could therefore choose the database location for a subsequent startup.

SQLite connection setup can open or create a database at the supplied filename. That gives the setting filesystem significance, subject to the application's operating-system permissions; it is not equivalent to changing a display preference. [Python `sqlite3.connect` documentation](https://docs.python.org/3/library/sqlite3.html#sqlite3.connect).

The correction removed `db_path` from the request model and from the restart-needed field set. The model already prohibited extra fields, so an attempted path update became a validation error instead of being silently ignored or persisted. Pydantic documents the difference between allowing, ignoring and forbidding extra input. [Pydantic extra-field configuration](https://pydantic.dev/docs/validation/latest/api/pydantic/config/#pydantic.config.ConfigDict.extra).

| Request or action | Before | After |
|---|---|---|
| `PUT /api/config` containing `db_path` | Database location was part of the accepted settings model and required restart to take effect. | HTTP 422; the field is outside the browser update contract. |
| A permitted mode preference update | Accepted as an ordinary setting with restart reporting. | Still accepted; the existing positive regression checks this path. |
| Change database location as an operator | Possible through application configuration. | Remains an operator-controlled configuration decision outside this browser endpoint. |

The original patch pattern is a capability reduction, not a pathname blacklist:

```text
browser_update_schema:
    include only browser-editable preferences
    exclude database destination
    reject all unknown fields

on settings update:
    validate request against that schema
    persist only the validated, explicitly supplied fields
    report restart needs for permitted startup settings
```

An absolute-path restriction or a check for `..` would still leave the browser choosing among filesystem destinations. Removing the field closes that authority at the input boundary. The evidence establishes arbitrary database-path selection being removed; it does not demonstrate unrestricted filesystem writes, code execution or access beyond the process's permissions.

## Boundary 2: private demo storage inside a shared temporary area

Demo mode first chose a database beside the configured application database. If that parent directory was unwritable, the old fallback used one fixed filename in the system temporary directory. Another local user could potentially occupy that predictable name before startup, creating an unintended collision or symlink target.

The second correction retained the ordinary writable-directory path and changed only the fallback. It created a new private directory with `tempfile.mkdtemp`, then placed the demo database inside it.

```text
choose demo database:
    if the ordinary data directory is writable:
        use the ordinary demo path
    otherwise:
        private_directory = securely create a new temporary directory
        use a fixed database basename inside that private directory
```

The fixed basename is safe against pre-placement by other user IDs because its containing directory is newly created and private. The security property comes from secure directory creation and access permissions, not just randomizing a filename. Python documents `mkdtemp` as creating a directory without a creation race, accessible only to the creating user ID. [Python `tempfile.mkdtemp` documentation](https://docs.python.org/3/library/tempfile.html#tempfile.mkdtemp).

That isolation does not defend against a compromised process with the same user ID or privileged host access. It also does not provide automatic cleanup: `mkdtemp` leaves directory deletion to its caller. The inspected patch addressed creation safety; it did not establish a cleanup lifecycle. [Python temporary-directory lifecycle documentation](https://docs.python.org/3/library/tempfile.html#tempfile.mkdtemp).

## Existing verification and boundary separation

The configuration correction added an existing synthetic regression: a config update containing `db_path` must return 422. The surrounding suite also checks that a permitted mode update returns 200 and reports a restart requirement. These establish the API contract under the fake application's local test client.

The temporary-directory correction changed six lines of startup source and added no test in that commit. Its evidence is the exact change from a predictable shared fallback to `mkdtemp`, interpreted against the standard-library contract. The temporary-directory behavior was reviewed against the library contract; no symlink experiment was run.

A browser-origin guard is a separate boundary: comparing Origin host:port with Host and allowing requests without Origin does not authenticate a caller or establish full scheme-sensitive origin equality. The two June 14 fixes protect input and local-storage boundaries without relying on that guard to provide identity.
