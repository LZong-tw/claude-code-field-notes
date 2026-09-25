## Terminal color depth

Claude Code renders its UI (mascot, theme colors, diffs, spinner) with 24-bit RGB colors. When it decides the terminal can't do 24-bit, it rounds every RGB value to the nearest entry of the xterm 256-color palette. The palette's color cube only has six steps per channel (0, 95, 135, 175, 215, 255), so the result is usually paler or grayer. The UI still works, but it looks noticeably **washed out**.

Checked against **2.1.282** (Linux build, running under WSL2).

### How the level is decided

1. **Base level** comes from a bundled copy of `supports-color`. The two lines that matter:
   - `COLORTERM=truecolor` gives level 3 (24-bit).
   - a `TERM` ending in `-256color` gives level 2 (256 colors).
2. **Upgrades to level 3** after that:
   - `TERM_PROGRAM=vscode` when the base level is 2.
   - `TERM` is one of `alacritty`, `contour`, `foot`, `ghostty`, `rio`, `wezterm`, `xterm-ghostty`, `xterm-kitty`. This is skipped when `FORCE_COLOR` or `NO_COLOR` is set, or stdout is not a TTY.
3. **Downgrade to level 2:** inside tmux (`TMUX` set) unless `CLAUDE_CODE_TMUX_TRUECOLOR` is set.

Nothing in this chain recognizes **Windows Terminal**.

### The WSL + Windows Terminal case

Windows Terminal supports 24-bit color but does not set `COLORTERM`. Inside WSL, `TERM` is usually `xterm-256color`. Windows Terminal forwards `WT_SESSION` into WSL through `WSLENV`, but the color logic above never looks at it. The result is **level 2**: Claude Code in WSL looks duller than the same version running natively on Windows in the same terminal. The native Windows build gets full color.

Concrete example: the mascot orange `#D77757` (215,119,87) is quantized to palette index 174 (`#D78787`, i.e. 215,135,135), which reads as pink. The screenshot color matched index 174 exactly.

Measurement: we ran `claude` in a pty for 10 s under each setting and counted the SGR foreground sequences it emitted (Kali on WSL2, Windows Terminal, `TERM=xterm-256color`, no tmux):

| Environment | 24-bit (`38;2;…`) | 256-color (`38;5;…`) |
|---|---|---|
| as-is (`COLORTERM` unset) | 0 | 5 |
| `COLORTERM=truecolor` | 10 | 0 |
| `FORCE_COLOR=3` | 0 | 10 |

`FORCE_COLOR=3` does **not** give Claude Code's own UI 24-bit color. Per the settings docs, `FORCE_COLOR`/`NO_COLOR` are meant for subprocesses, not Claude Code's UI (anthropics/claude-code#59585). Setting it also disables the `TERM`-based upgrade in step 2.

### Fix

Set `COLORTERM` only when the session is actually inside Windows Terminal, e.g. in `~/.zshrc` / `~/.bashrc` on the WSL side:

```sh
[ -n "$WT_SESSION" ] && export COLORTERM=truecolor
```

The condition matters. If you export it unconditionally, logging into the same machine over SSH from a terminal without 24-bit support (or from a Linux VT) makes every program that trusts `COLORTERM` emit sequences that terminal can't render. Open a new tab afterward; existing shells keep the old environment. The same variable also fixes other tools that trust `COLORTERM`. For example, Powerlevel10k hex colors stop being rounded to 256.

Claude Code itself hints at this: a startup tip `Try setting environment variable COLORTERM=truecolor for richer colors` is eligible when `COLORTERM` is unset and the level is below 3 (30-session cooldown).

### Evidence (2.1.282)

Base level (bundled `supports-color`):

```js
if(H.COLORTERM==="truecolor")return 3;...if(/-256(color)?$/i.test(H.TERM))return 2;
```

Upgrades and the tmux downgrade:

```js
function le(){if(a.TERM_PROGRAM==="vscode"&&u.level===2)return u.level=3,!0;return!1}
var se=new Set(["alacritty","contour","foot","ghostty","rio","wezterm","xterm-ghostty","xterm-kitty"]);
function ie(){if(!process.stdout.isTTY||a.NO_COLOR||a.FORCE_COLOR!==void 0||te())return!1;let e=a.TERM;if(e&&se.has(e)&&u.level<3)return u.level=3,!0;return!1}
function ae(){if(a.CLAUDE_CODE_TMUX_TRUECOLOR)return!1;if(a.TMUX&&u.level>2)return u.level=2,!0;return!1}
```

The tip:

```js
id:"colorterm-truecolor",...content:async()=>"Try setting environment variable COLORTERM=truecolor for richer colors",cooldownSessions:30,isRelevant:async()=>!a.COLORTERM&&ue.level<3
```

`WT_SESSION` is read elsewhere, including on Linux (it isn't used for color). Only the `isMicrosoftWindowsTerminal()` helper requires `win32`:

```js
function rFo(){if(H()==="windows"||a.WT_SESSION)process.env.CLAUDE_CODE_ALT_SCREEN_FULL_REPAINT??="1"}
isMicrosoftWindowsTerminal(){return this.proc.platform==="win32"&&!!this.proc.env.WT_SESSION}
```
