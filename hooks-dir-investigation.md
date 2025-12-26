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

## Related Code Locations

- Flag definition: `cmd/podman/root.go:595-597`
- Flag processing: `cmd/podman/root.go:263-265`
- Runtime option: `pkg/domain/infra/runtime_libpod.go:192-194`
- Option validation: `libpod/options.go:259-273`
- Hook loading: `libpod/container_internal.go:2427-2458`
- Hook manager: `vendor/go.podman.io/common/pkg/hooks/hooks.go:51-66`
- Directory reading: `vendor/go.podman.io/common/pkg/hooks/read.go:64-94`

## Test Coverage

Existing tests:
- `test/system/030-run.bats:1546` - Tests hooks are preserved on restart
- `test/e2e/run_test.go:932` - Tests hooks with comma in directory name

**Missing test:** No test verifies behavior with invalid/non-existent hooks directories.
