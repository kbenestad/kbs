# kbs

A simple filename and content search tool for the command line. Built as a
faster-to-use alternative to typing raw `grep`/`find` invocations for
everyday searches — supports OR terms, wildcards, file-type filters, tree
output, and surfacing surrounding context around a content match.

## Install

One-line install:

```
curl -fsSL https://raw.githubusercontent.com/kbenestad/kbs/refs/heads/main/kbs -o ~/.local/bin/kbs
chmod +x ~/.local/bin/kbs

```

Or a safer version:

```sh
mkdir -p ~/.local/bin
curl -fsSL https://raw.githubusercontent.com/kbenestad/kbs/refs/heads/main/kbs -o ~/.local/bin/kbs
chmod +x ~/.local/bin/kbs
```

Make sure `~/.local/bin` is on your `PATH`:

```sh
echo $PATH | grep -q "$HOME/.local/bin" || echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

(Use `~/.zshrc` instead if your shell is zsh.)

Requires Python 3. No other dependencies.

## Usage

```
kbs SEARCHTERM [flags]
```

By default, `kbs` searches **filenames** (not contents), starting in the
current directory and recursing into all subfolders.

### Basic search

```sh
kbs report
```

Finds any filename containing "report", anywhere below the current directory.

### Multiple terms (OR)

Separate terms with `|`. **Quote the whole argument** — an unquoted `|` is a
shell pipe operator, not a search separator, and bash will misinterpret it.

```sh
kbs "invoice | receipt"
```

### Wildcards

`*` matches any sequence of characters. **Quote the argument** — an unquoted
`*` gets expanded by bash itself before `kbs` ever sees it.

```sh
kbs "draft*"
```

### Search file contents instead of filenames

```sh
kbs -c "budget"
```

Prints each matching file, followed by which of your search term(s) were
found in it:

```
./finance/q1.md
budget

./notes/meeting.txt
budget | forecast
```

### Restrict to the current folder only (no recursion)

```sh
kbs report -r
```

### Filter by file type

```sh
kbs report -p office   # office/doc formats: .doc .docx .odt .rtf .md .txt .csv .pdf
kbs report -p html     # .html .htm
kbs report -p dev      # common code files: .py .js .ts .go .rs .java .php .rb ...
```

Only one `-p` value at a time. New types can be added easily — see
"Adding a new file type" below.

### Tree view

```sh
kbs report -t
```

Prints matches as an indented tree instead of a flat list.

### Show context around a content match

Requires `-c`. Shows N words before and after the match (default 8):

```sh
kbs "budget" -c -s
kbs "budget" -c -s 15
```

```
./finance/q1.md
we came in under the projected budget for the quarter overall and
```

### Show every match, not just the first

Same as `-s`, but every occurrence gets its own snippet instead of just the
first one:

```sh
kbs "budget" -c -sa
kbs "budget" -c -sa -s 15
```

### Colors

Filenames print in yellow, the folder path in light grey. When `-s`/`-sa`
shows context, the matched term itself is also highlighted yellow —
if it's hard to tell which yellow is which at a glance, that's a real
trade-off of picking one highlight color for two different things, not a
bug; ask if you'd rather they were different colors.

Colors switch off automatically when output isn't going to a terminal
(piped into another command, redirected to a file).

### Interactive mode — jump to a result's folder

```sh
kbs report -c -i
```

Lists matches grouped by folder, numbered. Enter a number and (if you've
installed the shell function below) your terminal `cd`s straight into that
folder.

```
kbs -- results:
Searchterm report was found in the following files:
1. finance/q1.md
2. notes/meeting.txt
   notes/followup.txt

Open folder? Enter number:
```

**Important:** a running program cannot change its parent shell's working
directory — this is true of every command-line tool, not specific to `kbs`.
To make `-i` actually `cd` for you, add this function to `~/.bashrc` (or
`~/.zshrc`):

```sh
kbs() {
    local cdfile
    cdfile=$(mktemp)
    KBS_CD_TARGET_FILE="$cdfile" command kbs "$@"
    if [ -s "$cdfile" ]; then
        cd "$(cat "$cdfile")"
    fi
    rm -f "$cdfile"
}
```

Then `source ~/.bashrc`. This defines a shell function named `kbs` that
wraps the real `kbs` script — `command kbs` inside it deliberately bypasses
the function and calls the actual program in your `PATH`, avoiding infinite
recursion. For every normal (non-`-i`) call, the temp file stays empty and
nothing extra happens. Without this function, `-i` still works — it just
prints the target folder instead of jumping there.

### Find and replace in file contents

```sh
kbs "old text" -rp "new text"
```

Scans matching files, shows a preview of what would change and how many
occurrences per file, and asks for confirmation before writing anything:

```
kbs -- replace preview: "old text" -> "new text"
  finance/q1.md  (2 occurrences)
  notes/meeting.txt  (1 occurrence)

Replace in 2 file(s)? [y/N]:
```

Nothing is written unless you answer `y`. `-rp` can't be combined with
`-c`, `-s`, `-sa`, `-t`, or `-i` — it's a standalone operation. `-r`
(non-recursive) and `-p TYPE` (file-type filter) still apply, to scope
which files get scanned.

**Note:** `-r` was already taken by `--root` (non-recursive search), so
replace uses `-rp`/`--replace` instead — not `-r`.

### Combining flags

Flags combine freely:

```sh
kbs "invoice*" -c -p office -s 10
```

## Flag reference

| Flag | Long form | Meaning |
|---|---|---|
| `-c` | `--content` | Search file contents instead of filenames |
| `-r` | `--root` | Current folder only, don't recurse |
| `-p TYPE` | `--type TYPE` | Restrict to a file type: `office`, `html`, `dev` |
| `-t` | `--tree` | Print results as a tree |
| `-s [N]` | `--surface [N]` | Show N words of context around a content match (default 8); requires `-c` |
| `-sa` | `--surface-all` | Like `-s`, but every match instead of just the first; requires `-c` |
| `-i` | `--interactive` | Numbered results grouped by folder; pick one to `cd` there (needs shell function) |
| `-rp NEWTERM` | `--replace NEWTERM` | Find/replace in file contents, with a confirmation prompt |

## Adding a new file type

Open `kbs` and find the `TYPES` dictionary near the top of the file:

```python
TYPES = {
    'office': {'.doc', '.docx', '.odt', '.rtf', '.md', '.markdown',
               '.pdf', '.txt', '.csv'},
    'html': {'.html', '.htm'},
    'dev': {'.py', '.js', '.jsx', '.ts', '.tsx', '.sh', '.bash', '.c', '.h',
            '.cpp', '.hpp', '.java', '.go', '.rs', '.php', '.rb', '.json',
            '.yaml', '.yml', '.css', '.sql'},
}
```

Add a new entry — for example, images:

```python
    'images': {'.jpg', '.jpeg', '.png', '.gif', '.svg'},
```

That's the only change needed. `kbs report -p images` works immediately,
and an invalid `-p` value (e.g. a typo) now gives a clear error rather than
silently matching nothing.

## Notes and caveats

- Matching is case-insensitive throughout.
- Content search skips files it can't decode as text (binaries, etc.) rather
  than erroring.
- `-s` (surface) shows only the **first** match per file; use `-sa` to see
  every occurrence instead.
- Always quote search terms containing `|` or `*`, or your shell will
  interpret them before `kbs` gets a chance to.

## License

Apache License, Version 2.0. See [LICENSE](LICENSE).
