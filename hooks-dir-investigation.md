# Investigation: --hooks-dir Option Implementation and Validation

## Summary
The `--hooks-dir` option in Podman does not validate whether the provided paths exist or are valid directories. Invalid paths are silently ignored during container creation.

## How --hooks-dir is Implemented

### 1. Flag Definition (cmd/podman/root.go:595-597)
```go
hooksDirFlagName := "hooks-dir"
pFlags.StringArrayVar(&podmanConfig.HooksDir, hooksDirFlagName, podmanConfig.ContainersConfDefaultsRO.Engine.HooksDir.Get(), "Set the OCI hooks directory path (may be set multiple times)")
_ = cmd.RegisterFlagCompletionFunc(hooksDirFlagName, completion.AutocompleteDefault)
```

The flag accepts multiple string values (directories) via `StringArrayVar`.

### 2. Flag Processing (cmd/podman/root.go:263-265)
```go
if cmd.Flag("hooks-dir").Changed {
    podmanConfig.ContainersConf.Engine.HooksDir.Set(podmanConfig.HooksDir)
}
```

When the flag is changed, the values are stored in the configuration **without any validation**.

### 3. Runtime Option (libpod/options.go:259-273)
```go
// WithHooksDir sets the directories to look for OCI runtime hook configuration.
func WithHooksDir(hooksDirs ...string) RuntimeOption {
    return func(rt *Runtime) error {
        if rt.valid {
            return define.ErrRuntimeFinalized
        }

        if slices.Contains(hooksDirs, "") {
            return fmt.Errorf("empty-string hook directories are not supported: %w", define.ErrInvalidArg)
        }

        rt.config.Engine.HooksDir.Set(hooksDirs)
        return nil
    }
}
```

**Only validation:** Checks for empty strings in the array. No check for path existence or validity.

### 4. Hook Loading (libpod/container_internal.go:2449-2458)
```go
manager, err := hooks.New(ctx, c.runtime.config.Engine.HooksDir.Get(), []string{"precreate", "poststop"})
if err != nil {
    return nil, err
}

allHooks, err = manager.Hooks(config, c.config.Spec.Annotations, len(c.config.UserVolumes) > 0)
if err != nil {
    return nil, err
}
```

Hooks are loaded when a container is created using `hooks.New()`.

### 5. Hook Manager (vendor/go.podman.io/common/pkg/hooks/hooks.go:51-66)
```go
func New(_ context.Context, directories []string, extensionStages []string) (manager *Manager, err error) {
    manager = &Manager{
        hooks:           map[string]*current.Hook{},
        directories:     directories,
        extensionStages: extensionStages,
    }

    for _, dir := range directories {
        err = ReadDir(dir, manager.extensionStages, manager.hooks)
        if err != nil && !errors.Is(err, os.ErrNotExist) {
            return nil, err
        }
    }

    return manager, nil
}
```

**Key issue:** Line 60 shows that `os.ErrNotExist` errors are **silently ignored**. This means non-existent directories don't cause failures.

### 6. Directory Reading (vendor/go.podman.io/common/pkg/hooks/read.go:64-69)
```go
func ReadDir(path string, extensionStages []string, hooks map[string]*current.Hook) error {
    logrus.Debugf("reading hooks from %s", path)
    files, err := os.ReadDir(path)
    if err != nil {
        return err
    }
    // ...
}
```

`os.ReadDir()` returns `os.ErrNotExist` if the directory doesn't exist, which is then ignored by the caller.

## The Problem

When a user provides an invalid path via `--hooks-dir`:

1. ✅ The flag is parsed successfully
2. ✅ The value is stored in the configuration
3. ✅ The runtime option is set (only checks for empty strings)
4. ❌ During container creation, `hooks.New()` tries to read from the invalid path
5. ❌ The error `os.ErrNotExist` is caught and **silently ignored**
6. ❌ The container runs successfully without any warning to the user

## Why This Design Exists

The current behavior appears intentional to allow flexibility:
- Users can specify hooks directories that may not exist yet
- Multiple directories can be specified, and some may be optional
- Default directories (`/usr/share/containers/oci/hooks.d`, `/etc/containers/oci/hooks.d`) might not exist on all systems

## Potential Solutions

### Option 1: Validate at Parse Time (Early Validation)
Add validation in `cmd/podman/root.go` when the flag is processed:
```go
if cmd.Flag("hooks-dir").Changed {
    // Validate each directory exists
    for _, dir := range podmanConfig.HooksDir {
        if _, err := os.Stat(dir); err != nil {
            return fmt.Errorf("hooks directory %q: %w", dir, err)
        }
    }
    podmanConfig.ContainersConf.Engine.HooksDir.Set(podmanConfig.HooksDir)
}
```

**Pros:** Fail fast, clear error message
**Cons:** Breaks backward compatibility if users rely on the current behavior

### Option 2: Validate in WithHooksDir (Runtime Validation)
Add validation in `libpod/options.go`:
```go
func WithHooksDir(hooksDirs ...string) RuntimeOption {
    return func(rt *Runtime) error {
        // ... existing checks ...

        for _, dir := range hooksDirs {
            if _, err := os.Stat(dir); err != nil {
                return fmt.Errorf("hooks directory %q: %w", dir, err)
            }
        }

        rt.config.Engine.HooksDir.Set(hooksDirs)
        return nil
    }
}
```

**Pros:** Validation happens before runtime is created
**Cons:** Still breaks backward compatibility

### Option 3: Warning Instead of Error (Soft Validation)
Log a warning when invalid directories are encountered:
```go
// In hooks.go New() function
for _, dir := range directories {
    err = ReadDir(dir, manager.extensionStages, manager.hooks)
    if err != nil {
        if errors.Is(err, os.ErrNotExist) {
            logrus.Warnf("Hooks directory %q does not exist, skipping", dir)
            continue
        }
        return nil, err
    }
}
```

**Pros:** Maintains backward compatibility, alerts users to potential issues
**Cons:** Users might not notice warnings

### Option 4: Strict Mode Flag
Add a global flag like `--strict-validation` that enables strict validation:
```go
if strictMode && errors.Is(err, os.ErrNotExist) {
    return nil, fmt.Errorf("hooks directory does not exist: %s", dir)
}
```

**Pros:** Backward compatible, opt-in validation
**Cons:** Adds complexity, users need to know about the flag

## Recommendation

**Option 3 (Warning)** is the best approach because:
1. It maintains backward compatibility
2. It alerts users to potential typos or configuration errors
3. It's consistent with Podman's design philosophy of being permissive but informative
4. Similar behavior exists in line 2444 where deprecated implicit hook directories show warnings

## Relative Path Support

### Question: Does --hooks-dir support relative paths?

**Short answer:** Yes, but with caveats that may cause unexpected behavior.

### How Relative Paths Work

1. **No Path Normalization:** The code does not convert relative paths to absolute paths anywhere in the flow:
   - `cmd/podman/root.go:595-597` - Flag accepts raw string values
   - `libpod/options.go:266-268` - Only checks for empty strings
   - `vendor/.../hooks/read.go:66` - Passes path directly to `os.ReadDir(path)`

2. **Resolution Timing:** Relative paths are resolved by `os.ReadDir()` relative to the **current working directory** at the time hooks are loaded (during container creation).

3. **Code Evidence:**
```go
// vendor/go.podman.io/common/pkg/hooks/read.go:64-69
func ReadDir(path string, extensionStages []string, hooks map[string]*current.Hook) error {
    logrus.Debugf("reading hooks from %s", path)
    files, err := os.ReadDir(path)  // Uses path as-is
    if err != nil {
        return err
    }
```

### Potential Issues with Relative Paths

1. **Working Directory Dependent:** The hooks directory resolution depends on where `podman` is invoked from:
   ```bash
   cd /home/user
   podman run --hooks-dir=./hooks alpine echo test  # Looks for /home/user/hooks

   cd /tmp
   podman run --hooks-dir=./hooks alpine echo test  # Looks for /tmp/hooks
   ```

2. **No Validation:** Combined with the lack of path existence validation, users might not realize their hooks aren't being loaded:
   ```bash
   podman run --hooks-dir=./hooks alpine echo test
   # Silently succeeds even if ./hooks doesn't exist
   ```

3. **Container Restart:** If containers are restarted from a different working directory, the relative path may resolve differently.

### Documentation Status

The documentation (`docs/source/markdown/podman.1.md:62`) describes `--hooks-dir=*path*` but does not specify whether paths should be absolute or relative.

### Recommendation

For production use, **always use absolute paths** with `--hooks-dir` to avoid working-directory-dependent behavior. The lack of path validation makes debugging relative path issues difficult.

---

## Shell Expansion in Hook Path Property

### Question: Does the hook.path property support shell notation like tilde (~) or environment variables ($HOME)?

**Short answer:** No. The hook path does not support any shell expansion.

### How Hook Paths Are Processed

The hook path is treated as a **literal string** throughout the entire processing pipeline:

1. **JSON Parsing** (`vendor/.../hooks/1.0.0/hook.go:26-30`):
```go
func Read(content []byte) (hook *Hook, err error) {
    if err = json.Unmarshal(content, &hook); err != nil {
        return nil, err
    }
    return hook, nil
}
```
The path is extracted from JSON as a plain string with no expansion.

2. **Path Validation** (`vendor/.../hooks/1.0.0/hook.go:47-49`):
```go
if err := fileutils.Exists(hook.Hook.Path); err != nil {
    return err
}
```
The path is checked for existence **as-is** using `unix.Faccessat()` or `os.Stat()`, which do not perform shell expansion.

3. **Hook Execution** (`vendor/.../hooks/exec/exec.go:55-63`):
```go
cmd := osexec.Cmd{
    Path:   hook.Path,  // Used directly
    Args:   hook.Args,
    Env:    hook.Env,
    // ...
}
```
The path is passed directly to Go's `os/exec.Cmd.Path`, which expects an **absolute path** or a path relative to the current working directory. Go's `os/exec` does **not** perform shell expansion.

### What This Means

**These will NOT work:**
```json
{
  "version": "1.0.0",
  "hook": {
    "path": "~/hooks/my-hook.sh"        // ❌ Tilde not expanded
  }
}
```

```json
{
  "version": "1.0.0",
  "hook": {
    "path": "$HOME/hooks/my-hook.sh"    // ❌ Variable not expanded
  }
}
```

```json
{
  "version": "1.0.0",
  "hook": {
    "path": "${HOME}/hooks/my-hook.sh"  // ❌ Variable not expanded
  }
}
```

**These WILL work:**
```json
{
  "version": "1.0.0",
  "hook": {
    "path": "/home/user/hooks/my-hook.sh"  // ✅ Absolute path
  }
}
```

```json
{
  "version": "1.0.0",
  "hook": {
    "path": "/usr/local/bin/my-hook"       // ✅ Absolute path
  }
}
```

### Why No Shell Expansion?

1. **OCI Runtime Spec**: The `Hook` struct in the OCI runtime specification defines `Path` as a simple string field (`vendor/.../runtime-spec/specs-go/config.go:189`):
```go
type Hook struct {
    Path    string   `json:"path"`
    Args    []string `json:"args,omitempty"`
    Env     []string `json:"env,omitempty"`
    Timeout *int     `json:"timeout,omitempty"`
}
```

2. **Direct Execution**: Go's `os/exec` package executes binaries directly without invoking a shell, so shell features like tilde expansion or variable substitution are not available.

3. **Security**: Avoiding shell expansion reduces attack surface and prevents unintended variable expansion.

### Test Evidence

All existing tests use absolute paths:
- `test/system/030-run.bats:1559`: Uses `"path": "$hooksdir/hook.sh"` - but the `$hooksdir` is expanded **by the shell when creating the JSON file**, not by Podman
- `test/e2e/run_test.go:943`: Uses `fmt.Sprintf` to inject absolute paths into the JSON before writing it

### Workarounds

If you need to use environment variables, expand them **before** creating the hook JSON:

**Option 1: Shell substitution when creating the file**
```bash
cat > /etc/containers/oci/hooks.d/my-hook.json <<EOF
{
  "version": "1.0.0",
  "hook": {
    "path": "${HOME}/hooks/my-hook.sh"
  },
  "stages": ["prestart"]
}
EOF
```

**Option 2: Template processing**
```bash
envsubst < my-hook.json.template > /etc/containers/oci/hooks.d/my-hook.json
```

**Option 3: Always use absolute paths**
```json
{
  "version": "1.0.0",
  "hook": {
    "path": "/usr/local/bin/my-hook"
  },
  "stages": ["prestart"]
}
```

### Recommendation

**Always use absolute paths** in hook JSON files. This avoids any ambiguity and ensures the hook binary can be found regardless of the current working directory.

## Related Code Locations

- Flag definition: `cmd/podman/root.go:595-597`
- Flag processing: `cmd/podman/root.go:263-265`
- Runtime option: `pkg/domain/infra/runtime_libpod.go:192-194`
- Option validation: `libpod/options.go:259-273`
- Hook loading: `libpod/container_internal.go:2427-2458`
- Hook manager: `vendor/go.podman.io/common/pkg/hooks/hooks.go:51-66`
- Directory reading: `vendor/go.podman.io/common/pkg/hooks/read.go:64-94`
- Path type storage: `vendor/go.podman.io/common/internal/attributedstring/slice.go:32-42`

## Test Coverage

Existing tests:
- `test/system/030-run.bats:1546` - Tests hooks are preserved on restart
- `test/e2e/run_test.go:932` - Tests hooks with comma in directory name (uses absolute paths)

**Missing tests:**
- No test verifies behavior with invalid/non-existent hooks directories
- No test validates relative path behavior
