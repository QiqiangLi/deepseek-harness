# Worker roles and warm pools

This reference defines the host-level worker placement seam for integrations that run more than one DSH session in parallel. A worker role names the minimum capabilities a session needs. A warm pool keeps idle workers ready without granting a project directory or a user directory permanent access.

## Roles

The initial role registry contains three names.

| Role | Capabilities | Typical use |
| --- | --- | --- |
| `local-readonly` | Read workspace files and run read-only commands | Inspection, discovery, and static checks |
| `local-edit` | Read and write workspace files and run commands | Local code changes and builds |
| `remote-runner` | Read workspace files, run commands, use SSH, and use network access | Remote checks and remote execution |

Role names are identifiers. A deployment may add a role, but it must declare its capability set and lifecycle limits. A caller that names a role receives that role or an explicit capability error. Placement must not silently widen the role.

## Pool contract

The host owns one pool per role. Pool configuration declares the warm size, maximum leased workers, idle timeout, and teardown policy. A pool exposes three operations.

```text
acquire(role, request) -> lease
lease.release(outcome)
pool.close()
```

`request` carries resource intent such as `read_paths`, `write_paths`, `ssh_hosts`, `network`, and the commands that the worker must run. The pool checks that intent against the role before starting work. It performs one lightweight capability probe after a worker starts. A denied path, command, SSH host, or network request is a permission error and does not become a stalled session.

The lease owns the worker until `release`. Release always clears temporary workspace mounts, child processes, and credential references. A failed probe destroys the worker instead of returning it to the warm pool. A worker that exceeds the idle timeout is also destroyed and replaced only when the warm-size policy requires it.

## Session placement

Session creation accepts an optional `workerRole`. When it is absent, the host derives the smallest role from the declared resource intent. The MCP or other transport adapter forwards the role and intent. It does not create or manage pools.

Session teardown releases the lease after the session's child processes and streams settle. A failed session still releases its lease. The session result records the selected role and the probe outcome so callers can distinguish permission failure from execution failure.

## Configuration example

```yaml
workerPools:
  local-readonly:
    warmSize: 2
    maxLeased: 8
    idleTimeoutMs: 300000
  local-edit:
    warmSize: 1
    maxLeased: 4
    idleTimeoutMs: 300000
  remote-runner:
    warmSize: 1
    maxLeased: 2
    idleTimeoutMs: 600000
```

The configuration is process-local. It does not encode a permanent project path, home directory, secret, or operating-system ACL. Resource intent is supplied per session and is removed when the lease is released.

## Failure and ownership

The caller owns role selection. The host owns capability checks, pool admission, and cleanup. A role mismatch, unavailable capability, or failed probe returns a structured permission error for the caller to handle. Quota and rate-limit failures may follow the deployment's normal retry policy. Permission errors must not be retried as model or worker failures.
