# Shell Autocompletion

`v2hub` supports tab-completion of commands, subcommands, options, and — where it can — real values pulled live from the API: subscription tokens, config hashes, provider hashes, and banned/whitelisted IPs. It works the same way `git` does: type a partial command, press `Tab`, and either complete it or cycle through matching suggestions.

## Automatic Setup

Completion is installed automatically the first time you run `v2hub` in an interactive shell (bash, zsh, or fish) — nothing to install or type. Restart your shell (or open a new terminal) after the first run, then try:

```bash
v2hub get<TAB>                                # -> suggests real subscription tokens
v2hub update-config <token> --config-id <TAB> # -> suggests config hashes for that subscription
v2hub provider <TAB>
```

The first time setup succeeds, a one-line notice is printed to stderr:

```text
[v2hub] Enabled zsh tab-completion (restart your shell, or run: source /path/to/script)
```

### How Automatic Setup Decides Whether to Run

The installer is deliberately conservative — every one of these conditions must hold, or it silently does nothing:

- **Not already answering a `<TAB>` press.** Click/Typer invoke the CLI with a `*_COMPLETE` environment variable set to serve a completion request; the installer never runs on that hot path.
- **Not opted out** — see [Opting Out](#opting-out) below.
- **Running in a real interactive terminal** — both stdin and stdout must be a TTY. Scripts, CI, cron jobs, and `subprocess` calls never trigger it.
- **Shell is detected and supported.** Detection uses `shellingham`, which inspects the parent process tree rather than trusting a possibly-stale `$SHELL`. Only bash, zsh, and fish are wired up automatically; PowerShell is skipped because it needs a profile-script decision the installer can't make silently.
- **Not already installed for this shell and CLI revision.** A marker file under `~/.cache/v2hub-cli/` (or `$XDG_STATE_HOME`/`$XDG_CACHE_HOME`) records what's already been set up, so the installer runs at most once per shell — until that revision changes, e.g. if the set of dynamic completions changes in a future release, at which point it re-installs once to pick up the update.

If installation fails for any reason (no permission to write the rc file, an exotic environment, a sandboxed container, etc.), it fails **completely silently** — no error, no traceback — and your actual command still runs normally.

## Manual Setup

If you'd rather manage completion yourself, opt out of the automatic setup and install (or print) the completion script manually:

```bash
export V2HUB_NO_AUTOCOMPLETE=1

v2hub --install-completion         # install for your current shell
v2hub --show-completion            # print the script for your shell
v2hub --show-completion bash       # or force a specific shell
```

## Opting Out

Set `V2HUB_NO_AUTOCOMPLETE=1` (or `true`/`yes`/`on`) to skip automatic setup entirely — useful if you manage your dotfiles some other way (chezmoi, nix, a custom completion setup, etc.) and don't want the CLI editing them:

```bash
export V2HUB_NO_AUTOCOMPLETE=1
```

## Dynamic Value Completion

In shells that support it (zsh, fish — not bash, which has no equivalent display), each dynamic suggestion is shown together with a short hint so you can tell entries apart:

```text
abc123de…  My Personal VPN
f91a02bb…  My Work VPN
```

Only the value itself (the token or hash) is ever inserted — the hint is just there to help you pick the right one.

| Completing | Hint shown | Requires |
| --- | --- | --- |
| Subscription token | Subscription name | `--base-url`/`V2HUB_API_URL` and `--api-token`/`V2HUB_API_TOKEN` already resolvable |
| `--config-id` on `update-config` | Config comment (for `config` sources) or a truncated URL/token (for `external_url`/`internal_token` sources) | The `token` argument already typed on the command line |
| Provider hash (admin) | — | `--base-url` and `--secret-key`/`V2HUB_ADMIN_SECRET` |
| Banned/whitelisted IP (admin) | — | `--base-url` and `--secret-key`/`V2HUB_ADMIN_SECRET` |

Dynamic completion resolves credentials the same way the real commands do: already-typed `--base-url`/`--api-token`/`--secret-key` on the command line first, then the matching environment variable.

Completion lookups use a short, fixed 2-second timeout with no retries — deliberately more aggressive than a real command's network settings, so a slow or unreachable API never makes a `<TAB>` press feel like the shell has hung. If a lookup fails or times out for any reason, it returns no suggestions rather than showing an error.
