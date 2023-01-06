# Modsurfer `validate` GitHub Action

## Overview

Modsurfer by [Dylibso](https://dylib.so) is an application to help teams debug and observe their 
WebAssembly modules and components. It scans WASM binaries and extracts critical information about
the code contained within, as well as makes WASM code searchable and visible to the teams creating
and executing it. See a demo of the Modsurfer application here: https://modsurfer.app

This GitHub Action runs Modsurfer's CLI, [found here](https://github.com/dylibso/modsurfer), using
the `validate` command. This command expects two inputs: 
- **path** (`-p` from the CLI): pointing to the .wasm file to validate
- **check** (`-c` from the CLI): pointing to the YAML file (default: `mod.yaml`) to validate against

## Usage

To use this action, add the following step in your workflow:

```yaml
    # ...
    steps:
      - uses: actions/checkout@v3
      - name: modsurfer validate
        uses: dylibso/modsurfer-validate-action@main
        with:
            path: path/to/your.wasm
            check: path/to/mod.yaml
```

An example `mod.yaml` (a "check file") could be: 

```yaml
validate:
  # url: https://github.com/extism/extism/blob/main/mod.yaml
  allow_wasi: false
  
  imports:
    include:
      - http_get
      - log_message
      - proc_exit
    exclude: 
      - fd_write
    namespace:
      include:
        - env
      exclude:
        - some_future_deprecated_module_name
        - wasi_snapshot_preview1

  exports: 
    max: 100
    include:
      - _start
      - bar
    exclude:
      - main
      - foo

  size:
    max: 4MB

  complexity:
    max_risk: low

```

When a module fails to validate against the provided check file, a report is printed (seen below), 
and will issue a non-zero exit code, failing the CI workflow. 

```
┌────────┬──────────────────────────────────────────────────┬──────────┬──────────┬───────────────────┬────────────┐
│ Status │ Property                                         │ Expected │ Actual   │ Classification    │ Severity   │
╞════════╪══════════════════════════════════════════════════╪══════════╪══════════╪═══════════════════╪════════════╡
│ FAIL   │ allow_wasi                                       │ false    │ true     │ ABI Compatibility │ |||||||||| │
├────────┼──────────────────────────────────────────────────┼──────────┼──────────┼───────────────────┼────────────┤
│ FAIL   │ complexity.max_risk                              │ <= low   │ medium   │ Resource Limit    │ |          │
├────────┼──────────────────────────────────────────────────┼──────────┼──────────┼───────────────────┼────────────┤
│ FAIL   │ exports.exclude.main                             │ excluded │ included │ Security          │ |||||      │
├────────┼──────────────────────────────────────────────────┼──────────┼──────────┼───────────────────┼────────────┤
│ FAIL   │ exports.include.bar                              │ included │ excluded │ ABI Compatibility │ |||||||||| │
├────────┼──────────────────────────────────────────────────┼──────────┼──────────┼───────────────────┼────────────┤
│ FAIL   │ exports.max                                      │ <= 100   │ 151      │ Security          │ ||||||     │
├────────┼──────────────────────────────────────────────────┼──────────┼──────────┼───────────────────┼────────────┤
│ FAIL   │ imports.include.http_get                         │ included │ excluded │ ABI Compatibility │ ||||||||   │
├────────┼──────────────────────────────────────────────────┼──────────┼──────────┼───────────────────┼────────────┤
│ FAIL   │ imports.include.log_message                      │ included │ excluded │ ABI Compatibility │ ||||||||   │
├────────┼──────────────────────────────────────────────────┼──────────┼──────────┼───────────────────┼────────────┤
│ FAIL   │ imports.namespace.exclude.wasi_snapshot_preview1 │ excluded │ included │ ABI Compatibility │ |||||||||| │
├────────┼──────────────────────────────────────────────────┼──────────┼──────────┼───────────────────┼────────────┤
│ FAIL   │ imports.namespace.include.env                    │ included │ excluded │ ABI Compatibility │ ||||||||   │
├────────┼──────────────────────────────────────────────────┼──────────┼──────────┼───────────────────┼────────────┤
│ FAIL   │ size.max                                         │ <= 4MB   │ 4.4 MiB  │ Resource Limit    │ |          │
└────────┴──────────────────────────────────────────────────┴──────────┴──────────┴───────────────────┴────────────┘
```