Thanks for reporting this issue! I've identified and fixed the problem.

## The Problem

When using `all_databases: true`, the code was attempting to execute the `pg_dumpall` command via `sh -c`, but the `helper.Exec()` function splits command strings by whitespace. This caused the shell command structure to break, and `pg_dumpall` wasn't receiving the `--host` and `--port` parameters correctly, causing it to fall back to socket-based connection attempts.

The original code was building a command like:
```
sh -c pg_dumpall --host=postgresql --port=5432 --username=postgres > /tmp/backupall.sql
```

But when `helper.Exec()` split this by whitespace, `sh -c` only received `pg_dumpall` as the command argument, losing all the connection parameters and redirection.

## The Fix

I've updated the code in `database/postgresql.go` to:

1. **For `all_databases` case**: The `build()` method now returns a complete shell command string with output redirection:
   ```go
   pgDumpallCmd := "pg_dumpall " + strings.Join(pgDumpallArgs, " ") + " > " + db._dumpFilePath
   ```

2. **Execution method**: The `perform()` method now uses `helper.ExecScript()` for the `all_databases` case instead of `helper.Exec()`. The `ExecScript()` function properly handles shell scripts by writing them to a temporary file and executing them via `sh`, which allows shell features like output redirection (`>`) to work correctly.

This ensures that when `all_databases: true` is set, the command is executed as:
```bash
pg_dumpall --host=postgresql --port=5432 --username=postgres > /path/to/backup.sql
```

And all connection parameters are properly passed to `pg_dumpall`, preventing it from falling back to socket connections.

## Testing

You can test this fix with your configuration:
```yaml
databases:
  my_postgresql:
    type: postgresql
    host: postgresql
    port: 5432
    username: postgres
    password: xxxxxxxxxxxxxxxxx
    all_databases: true
```

The fix should now properly connect to your PostgreSQL server using the specified host and port, rather than attempting to use a Unix socket.

