# ffidemo

A notebook kept in SQLite, where SQLite is the C library already installed
on the machine and Zuri reaches it through `ffi`. There is no binding code
in C or Rust: the package reads SQLite's own `sqlite3.h`, opens
`libsqlite3`, and everything else is Zuri.

It is a working application, notes with tags, attachments and ranked
full-text search, and a tour of what calling a real C library involves.

## Running it

It needs SQLite 3.24 or later, built with FTS5, which is how Debian,
Ubuntu, Fedora, Homebrew and most other distributions ship it.

```sh
zuri run                        # a tour, in a notebook that lives in memory
zuri run . add 'Trip' --body 'Pack boots and a map' --tag travel
zuri run . list
zuri run . search boots
zuri run . show 1
zuri run . attach 1 --file map.png
zuri run . stats
zuri run . --help
```

The notebook is `notes.db` in the working directory. `--db` names
another, and so does the `NOTES_DB` environment variable.

## Testing it

```sh
zuri test
```

## Layout

| Path | What it holds |
| --- | --- |
| `index.zu` | The entry point: re-exports `app`, and starts it when run. |
| `app/index.zu` | The command line, and the tour. |
| `app/notes.zu` | `Notebook`: notes, tags, attachments, search and statistics. |
| `app/sqlite/` | The SQLite driver, built on `ffi`. |
| `app/sqlite/sqlite3.h` | SQLite's header, exactly as SQLite ships it. |
| `tests/` | The driver, the notebook and the command line, under `zuri test`. |
| `project.toml` | The name, the version, and everything else about the project. |

## What it takes from ffi

Each part of the driver leans on a different part of the module.

| SQLite needs | How the driver does it |
| --- | --- |
| Its whole API declared | `ffi.declare()` reads `sqlite3.h` as shipped: every macro, typedef and prototype. `allow_missing` binds a header newer than the library. |
| A connection handed back through `sqlite3 **` | A pointer-sized slot from `ffi.alloc()`, read back with `get()`. |
| `SQLITE_TRANSIENT`, a function pointer cast from -1 | The header's own constant, which `ffi` reads as a typed pointer. |
| Handles released exactly once | `own()` with `sqlite3_close_v2`, `sqlite3_finalize` and `sqlite3_blob_close` as destructors, so the collector cleans up anything left open. |
| Text SQLite allocates, such as `sqlite3_expanded_sql()` | Owned with `sqlite3_free` as its destructor. |
| `sqlite3_mprintf('%Q', ...)`, a variadic function | Called directly; the extra argument is converted and promoted as C would. |
| Row callbacks for `sqlite3_exec()` | A Zuri function passed straight in, reading `char **` arrays. |
| SQL functions, aggregates and collations in Zuri | Lasting callbacks from `ffi.callback()`, released when the connection closes. |
| Per-group aggregate state | Eight bytes from `sqlite3_aggregate_context()`, holding a key into a Zuri dictionary. |
| Update, progress and trace hooks | Lasting callbacks; the trace hook reads a `sqlite3_int64` through a pointer. |
| A progress handler during a long query | `ffi.threaded()` runs the step on a helper thread, and SQLite's calls from that thread are answered by the isolate. |
| Incremental blob I/O | `sqlite3_blob_read()` into memory from `ffi.alloc_bytes()`. |
| Online backup | `sqlite3_backup_*()` between two connections. |

## License

MIT
