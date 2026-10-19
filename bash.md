# Bash Tips

## Subshell
Commands inside `( ... )` run in a child shell. Changes to variables, `cd`, `set` options, etc. do **not** affect the parent shell.

**Command:**
```bash
# cd only inside the subshell; the current directory is unchanged afterwards
(cd /tmp && ls)
pwd

# Variables set in a subshell are lost
x=1; (x=2; echo "inside: $x"); echo "outside: $x"   # inside: 2, outside: 1

# Command substitution also runs in a subshell
today=$(date +%F)
```

## Command group `{ ...; }`
`{ ...; }` groups commands in the **current** shell (no subshell). Syntax rules:
- A space is required after `{`, and the last command must end with `;` or a newline before `}`.
- Side effects persist: `cd`, variable assignments, `exit` (exits your script/shell, not just the group).

**Command:**
```bash
{ cd /tmp; ls; }   # the cd persists
```

### Combine stdout of several commands
Redirect the whole group once instead of each command separately.

```bash
# Write a header plus command output into one file
{ echo "== $(date) =="; uname -a; df -h; } > report.txt

# Append (>>) works the same way
{ echo "--- run ---"; ./build.sh; } >> build.log

# Pipe the combined output into one command
{ echo "name size"; ls -l | awk '{print $9, $5}'; } | column -t

# Combine stdout AND stderr of the whole group
{ make; make test; } > build.log 2>&1
{ make; make test; } 2>&1 | tee build.log
```

Without the group, only the last command is redirected:
```bash
echo header; ls > out.txt     # "header" goes to the terminal
{ echo header; ls; } > out.txt # both go to out.txt
```

### Read several lines from one input
Commands in the group share the same stdin, so each `read` consumes the next line:
```bash
{ read -r first; read -r second; } < file.txt
echo "$first / $second"
```

### Error-handling idiom
```bash
cd "$dir" || { echo "cannot cd to $dir" >&2; exit 1; }
```

### `{ }` vs `( )`
| | `{ ...; }` | `( ... )` |
|---|---|---|
| Runs in | current shell | subshell (child process) |
| `cd` / variables persist | yes | no |
| `exit` | exits the script | exits only the subshell |
| Syntax | needs spaces and trailing `;` | no special spacing |

Note: a group on either side of a pipe (`{ ...; } | cmd`) still runs in a subshell in bash, so variables set there are lost.

## case / esac
Match a value against glob patterns. Each branch ends with `;;`, and `*)` is the default.

**Command:**
```bash
case "$1" in
    start|run)
        echo "starting" ;;
    stop)
        echo "stopping" ;;
    *.tar.gz|*.tgz)
        tar xzf "$1" ;;
    *)
        echo "usage: $0 {start|stop|file.tar.gz}"
        exit 1 ;;
esac
```

## Verify a checksum
`md5sum -c` / `sha256sum -c` read lines of the form `<hash>  <file>` (two spaces) and check them. `-` means read that list from stdin.

**Command:**
```bash
# Check a single file against a known hash
echo "$hash  $file" | md5sum -c -
echo "$hash  $file" | sha256sum -c -

# Check against a downloaded checksum file
sha256sum -c SHA256SUMS
sha256sum -c --ignore-missing SHA256SUMS   # skip entries for files you don't have
```

On macOS (no GNU coreutils), use `shasum`, or `brew install coreutils` to get `md5sum`/`sha256sum`:
```bash
echo "$hash  $file" | shasum -a 256 -c -
```

## Install Docker on Linux
Use the official Docker install script, then add your user to the `docker` group so Docker commands can run without `sudo`.

**Command:**
```bash
# 1. Download and run the official install script
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# 2. Add the current user to the docker group
sudo usermod -aG docker "$USER"

# Log out and back in, or run the following in the current session:
newgrp docker
```

After logging back in, verify the setup with:
```bash
docker --version
docker ps
```

## cd relative to the script's location
Scripts often live in a subdirectory (e.g. `scripts/`) but need to run from the project root. `$0` is the script's path, `dirname` strips the filename, and `/..` goes one level up. This works no matter which directory you run the script from.

**Command:**
```bash
#!/usr/bin/env bash
cd "$(dirname "$0")/.." || exit 1   # now at the project root
```

Keep both pairs of quotes: the inner ones protect `$0`, the outer ones protect the result, so paths with spaces still work.

Save the absolute path for later use:
```bash
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
ROOT_DIR="$(cd "$SCRIPT_DIR/.." && pwd)"
```

If the script may be `source`d, use `${BASH_SOURCE[0]}` instead of `$0`. When a file is sourced, `$0` is the calling shell (e.g. `bash`), not the file:
```bash
cd "$(dirname "${BASH_SOURCE[0]}")/.." || return
```

## Sort by version number
`sort -V` (`--version-sort`) compares the numbers inside each string as numbers, so `1.10` sorts after `1.9`. A plain `sort` compares character by character and puts `1.10` first.

**Command:**
```bash
printf 'v1.10.0\nv1.2.0\nv1.9.3\nv2.0.0\n' | sort      # v1.10.0 v1.2.0 v1.9.3 v2.0.0  (wrong)
printf 'v1.10.0\nv1.2.0\nv1.9.3\nv2.0.0\n' | sort -V   # v1.2.0 v1.9.3 v1.10.0 v2.0.0

# Latest git tag
git tag | sort -V | tail -1

# Newest first
ls linux-*.tar.gz | sort -rV
```
