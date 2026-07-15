# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`ntlm_theft` is an offensive-security tool: it generates decoy files (docx, xlsx, url, lnk, pdf, etc.) that force a Windows host to make an outbound SMB/HTTP request to an attacker-controlled server, leaking NTLMv2 hashes for capture by tools like Responder or ntlmrelayx. All logic lives in `ntlm_theft/cli.py`; there is no test suite or CI. It's packaged (`pyproject.toml`, setuptools) purely so it can be installed as a pipx/pip CLI — there's still just the one module.

## Commands

Install as a CLI with pipx (pulls in the `xlsxwriter` dependency automatically):
```
pipx install .                 # normal install
pipx install --editable .      # editable, for active development on this repo
```

Run the generator (all three flags are required):
```
ntlm_theft -g <type> -s <server_ip> -f <base_filename>
```
- `-g/--generate`: `all`, `modern` (skips attack types broken on current Windows), or a specific type (see `choices` in the `argparse` setup inside `main()` in `ntlm_theft/cli.py`, e.g. `docx`, `xlsx`, `lnk`, `url`, `theme`).
- `-s/--server`: IP of the SMB capture listener.
- `-f/--filename`: base filename (no extension) — also becomes the output directory name.

Output is written to a new directory named `<filename>/` in the current working directory. If that directory already exists, the script prompts to delete it before regenerating.

There are no lint/test commands — nothing is configured for this repo. To sanity-check a change without reinstalling, run `python3 -m ntlm_theft -g <type> -s <ip> -f <name>` directly from the repo root.

## Architecture

`ntlm_theft/cli.py` is a flat dispatch table, not a class hierarchy. `script_directory = os.path.dirname(os.path.abspath(__file__))` at module level resolves to wherever `cli.py` ends up installed, which is why `templates/` had to move *inside* the `ntlm_theft/` package (`ntlm_theft/templates/`) — it's shipped as package data (`[tool.setuptools.package-data]` in `pyproject.toml`) so it's present next to `cli.py` in an installed wheel, not just in a source checkout.

1. **All argument parsing and dispatch live inside `main()`** (the `[project.scripts]` entry point in `pyproject.toml` points to `ntlm_theft.cli:main`). The `argparse` `-g` choices are the authoritative list of supported attack types (more reliable than the README's prose list — the README lags behind the code; e.g. `libraryms` and the `.htm` handler variant exist in the script but aren't documented in the README's attack list).
2. **One `create_*` function per attack type**, defined at module scope (outside `main()`), each with the signature `(generate, server, filename)`. Every function writes exactly one output file (or, for docx variants, one directory that gets zipped into a `.docx`) and prints a `Created: ...` line. Functions that are broken on modern Windows check `if generate == "modern": print("Skipping ..."); return` at the top instead of being excluded from `all`.
3. **Two families of generators**:
   - *Inline string templates*: most functions (`.url`, `.rtf`, `.xml`, `.htm`, `.wax`, `.m3u`, `.asx`, `.jnlp`, `.application`, `.pdf`, `.theme`, `.scf`, etc.) build the file content as an f-string/concatenation directly in the function, substituting `server` into a UNC or `file://` path.
   - *Template-based*: the three `.docx` variants (`includepicture`, `remotetemplate`, `frameset`) each copy a prebuilt OOXML directory tree from `ntlm_theft/templates/docx-*-template/`, string-replace the placeholder `127.0.0.1` inside a specific `*.xml.rels` file, then `shutil.make_archive` + rename the zip to produce the final `.docx`. `create_lnk` instead binary-patches a fixed byte offset (`0x136`) in `ntlm_theft/templates/shortcut-template.lnk` with the UTF-16LE-encoded UNC path. `create_xlsx_externalcell` uses the `xlsxwriter` library directly instead of a template.
4. **Dispatch block at the bottom of `main()`** (`if args.generate == "all" ...` / `elif ...`) calls the relevant `create_*` function(s) with an output path built from `args.filename`. When `all`/`modern` runs, it calls every generator in sequence; each single-type branch (`elif args.generate == "xlsx"`) calls just that one.

`ntlm_theft/__main__.py` just calls `main()`, enabling `python3 -m ntlm_theft` as a no-install alternative.

### Adding a new attack type

Follow the existing pattern: write a `create_X(generate, server, filename)` module-level function that prints `Created: ...`, add its name to the `choices` set in `main()`'s `argparse` block, add it to the `all`/`modern` dispatch block, and add its own `elif args.generate == "X":` branch. If it doesn't work on current Windows, guard it with the `if generate == "modern": ... return` early-skip convention rather than omitting it from `all`.
