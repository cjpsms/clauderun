# clauderun

Type a shell command badly — typos, wrong flags, or just say what you want in plain
language (Thai, English, whatever) — and Claude Haiku turns it into the real command.
You see the command, confirm it, and only then does it run.

```
$ clauderun gti stauts
  command  git status
  does     Show the working tree status
  danger   safe  (typo, cli)

Run? [y/N/e] y

$ clauderun "หาว่า port 4002 ใครใช้อยู่"
  command  sudo -A ss -tulnp | grep :4002
  does     แสดงโปรแกรมที่ใช้ port 4002
  danger   modifies  (translate, cli)

Run? [y/N/e]

$ clauderun "ลบ folder __pycache__ ทั้งหมดในนี้"
  command  find . -type d -name __pycache__ -delete
  does     ลบ folder __pycache__ ทั้งหมดที่อยู่ในนี้และ subdirectories
  danger   destructive  (translate, cli)
           matches destructive pattern `\bfind\b.*\s(-delete\b|-exec\s+rm\b)`

DESTRUCTIVE — type yes to run, e to edit:
```

Sibling of [claudeinstall](https://github.com/cjpsms/claudeinstall): same shape, but
here the model *has* to write shell, so the safety work moves into the wrapper.

## How it works

1. **Context** — OS, shell, cwd, the top-level entries in cwd, git branch, and which common
   tools actually exist on this machine (so it says `eza` not `exa`, and knows what "the
   video" refers to).
2. **Ask Claude Haiku** — JSON only: `{command, kind, explain, danger, needs_sudo, notes}`.
   The prompt forbids `sudo` (the wrapper adds `sudo -A` itself), `curl | sh`, and install
   steps.
3. **Check the programs** — every program the command calls is looked up in `PATH`. A missing
   one is flagged with a ready-made `claudeinstall <name>` hint.
4. **Our own danger classifier** — regexes for `rm -r`, `dd`, `mkfs`, `git push --force`,
   `git reset --hard`, `find -delete`, `kill -9`, `curl | sh`, `> existing-file`, and so on.
   It runs on the final text, independent of the model; the higher tier wins.
5. **Confirm** — `safe`/`modifies`: `y/N/e` (`e` edits the line in place, then re-checks).
   `destructive`: you must type `yes` in full, and `-y` does not skip it.
6. **Run** via `bash -c` in the current directory, with the real terminal (interactive
   programs work). If it exits non-zero, the last 20 stderr lines can be sent back once
   for a corrected command — which you confirm again. Max 2 rounds.

Only `kind == "typo"` answers are cached (`~/.cache/clauderun/`): a typo fix doesn't depend
on which directory you're in, a translated request does.

## Install

```
git clone https://github.com/cjpsms/clauderun ~/program/clauderun
~/program/clauderun/install.sh        # symlinks into ~/.local/bin
```

Needs Python ≥ 3.10 (stdlib only) and one of:

- the `claude` CLI ([Claude Code](https://claude.com/claude-code)) logged in — default, uses
  your subscription, no API key;
- or `ANTHROPIC_API_KEY` + `pip install anthropic` (`--backend api`, uses structured output).

`sudo -A` is used when `SUDO_ASKPASS` is set, plain `sudo` otherwise.

## Flags

```
-y, --yes        skip y/N for safe/modifies commands (never for destructive)
-n, --dry-run    show the command, don't run it
--no-retry       don't offer a fix when the command fails
--no-cache       ignore and don't write the typo-fix cache
--backend        auto | cli | api
--model          model override (default: haiku / claude-haiku-4-5)
-v               show context, token usage, classifier disagreements
```

## What it will not do

- run anything you haven't seen — every command is printed before the prompt;
- take `sudo` from the model — it is stripped and re-added by the wrapper only when needed;
- run `curl … | sh` or similar (blocked in the prompt *and* by the classifier);
- auto-run a destructive command, even with `-y`.

It is still an LLM writing shell: read the line before you press `y`.

## License

0BSD
