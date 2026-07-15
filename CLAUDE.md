# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`ntlm_theft` is a single-file offensive-security tool: it generates decoy files (docx, xlsx, url, lnk, pdf, etc.) that force a Windows host to make an outbound SMB/HTTP request to an attacker-controlled server, leaking NTLMv2 hashes for capture by tools like Responder or ntlmrelayx. All logic lives in `ntlm_theft.py`; there is no package, test suite, or CI.

## Commands

Install the one dependency:
```
pip3 install xlsxwriter
```

Run the generator (all three flags are required):
```
python3 ntlm_theft.py -g <type> -s <server_ip> -f <base_filename>
```
- `-g/--generate`: `all`, `modern` (skips attack types broken on current Windows), or a specific type (see `choices` in the `argparse` setup near the top of `ntlm_theft.py`, e.g. `docx`, `xlsx`, `lnk`, `url`, `theme`).
- `-s/--server`: IP of the SMB capture listener.
- `-f/--filename`: base filename (no extension) — also becomes the output directory name.

Output is written to a new directory named `<filename>/` in the current working directory. If that directory already exists, the script prompts to delete it before regenerating.

There are no lint/test/build commands — nothing is configured for this repo.

## Architecture

The script is a flat dispatch table, not a class hierarchy:

1. **`argparse` setup** at the top defines the `-g` choices, which is the authoritative list of supported attack types (more reliable than the README's prose list — the README lags behind the code; e.g. `libraryms`, `theme`, and the `.htm` handler variant exist in the script but aren't documented in the README's attack list).
2. **One `create_*` function per attack type**, each with the signature `(generate, server, filename)`. Every function writes exactly one output file (or, for docx variants, one directory that gets zipped into a `.docx`) and prints a `Created: ...` line. Functions that are broken on modern Windows check `if generate == "modern": print("Skipping ..."); return` at the top instead of being excluded from `all`.
3. **Two families of generators**:
   - *Inline string templates*: most functions (`.url`, `.rtf`, `.xml`, `.htm`, `.wax`, `.m3u`, `.asx`, `.jnlp`, `.application`, `.pdf`, `.theme`, `.scf`, etc.) build the file content as an f-string/concatenation directly in the function, substituting `server` into a UNC or `file://` path.
   - *Template-based*: the three `.docx` variants (`includepicture`, `remotetemplate`, `frameset`) each copy a prebuilt OOXML directory tree from `templates/docx-*-template/`, string-replace the placeholder `127.0.0.1` inside a specific `*.xml.rels` file, then `shutil.make_archive` + rename the zip to produce the final `.docx`. `create_lnk` instead binary-patches a fixed byte offset (`0x136`) in `templates/shortcut-template.lnk` with the UTF-16LE-encoded UNC path. `create_xlsx_externalcell` uses the `xlsxwriter` library directly instead of a template.
4. **Dispatch block at the bottom** (`if args.generate == "all" ...` / `elif ...`) calls the relevant `create_*` function(s) with an output path built from `args.filename`. When `all`/`modern` runs, it calls every generator in sequence; each single-type branch (`elif args.generate == "xlsx"`) calls just that one.

### Adding a new attack type

Follow the existing pattern: write a `create_X(generate, server, filename)` function that prints `Created: ...`, add its name to the `choices` set in the `argparse` block, add it to the `all`/`modern` dispatch block, and add its own `elif args.generate == "X":` branch. If it doesn't work on current Windows, guard it with the `if generate == "modern": ... return` early-skip convention rather than omitting it from `all`.
