# action-guard: a Claude Code hook that keeps an agent from changing a web application

*Template package, 30 September 2026. Everything you need is in this one document: three Python files (the hook, an installer, the tests), a policy file, and the steps to install and prove them. No dependencies beyond Python 3.8 or newer.*

## What it does

Claude Code runs a **hook** before every tool call. This hook looks at the call, works out what it would do to your application, and either lets it through, refuses it with a reason the model can read, or turns it into a permission prompt for you. The policy file says which application to protect and what counts as "changing" it. Out of the box:

| The call would... | Decision |
|---|---|
| Read from the application (GET, HEAD, OPTIONS) | passes, no prompt |
| Write to it (POST, PUT, PATCH, DELETE) | **refused**, unless the path is in `write_ok_paths` |
| Touch a path in `denied_paths` (e.g. `/admin`) with any method | **refused** |
| Touch a path in `ask_paths` | permission prompt |
| Send a request from inline code or a script (the method cannot be read) | **refused** (configurable) |
| Send a request whose target cannot be read (`curl "$URL"`, `xargs curl`) | permission prompt (configurable) |
| Send an unreadable number of requests to the application (a shell loop) | **refused** |
| Fill a form, run JavaScript or upload a file in a browser tab that is on the application | **refused** |
| Click or type in such a tab | permission prompt |
| Take a screenshot, read the page, find an element | passes |
| Call a tool of the application's own MCP server that is in `deny_tools` / `ask_tools` | refused / prompt |
| Anything else (other hosts, other tools) | no opinion — the normal permission flow applies |

A refusal cancels the call and hands the model a sentence saying what was refused and what to do instead (tell the user what change it wanted). It does not end the session.

## How Claude Code hooks work (the contract this relies on)

Checked against the official hooks reference (`code.claude.com/docs/en/hooks`) on 29 September 2026; re-check it if your Claude Code is much newer.

- A `PreToolUse` entry in `settings.json` names a **matcher** (a regular expression over tool names, e.g. `Bash|WebFetch|mcp__.*`) and a shell **command**. Before any matching tool call, Claude Code runs the command with one JSON object on **stdin**: `tool_name`, `tool_input` (for Bash: `{"command": "..."}`; for WebFetch: `{"url": ...}`; for MCP tools: their arguments), `cwd`, `session_id`, `tool_use_id`, `permission_mode`, `hook_event_name`, and, inside a subagent, `agent_id` and `agent_type`.
- **Print nothing and exit 0** = no opinion; the normal permission flow follows.
- **Print** `{"hookSpecificOutput": {"hookEventName": "PreToolUse", "permissionDecision": "deny", "permissionDecisionReason": "..."}}` **and exit 0** = the call is cancelled and the reason is shown to the model. `"ask"` in place of `"deny"` shows you the permission prompt with that reason. `"allow"` skips the prompt — never print it from a guard.
- An optional top-level `"systemMessage"` puts a line in the UI for the human.
- Exit code 2 also blocks, but the installed command is wrapped as `... || exit 1`, which would turn a 2 into a non-blocking 1; so decisions always travel as JSON.
- On a **timeout** (default 600 s; the installer sets 20 s) or a **non-zero exit other than 2**, the call proceeds. The guard is written to **fail open**: any internal error is logged and the call goes through. A guard that can wedge a session is worse than none; the tests are what make this trade acceptable.
- Hooks fire for **subagents** too (their calls carry `agent_type`).
- When several hooks match one call they run in parallel and the **most restrictive decision wins** (deny > ask > allow). So this hook can sit next to other hooks without coordination.
- Settings scopes: `~/.claude/settings.json` (you, every project), `.claude/settings.json` in a project (committed, shared with the team), `.claude/settings.local.json` (project, not committed). Claude Code re-reads settings files live for directories that had a settings file when the session started; otherwise open `/hooks` once or restart.
- Windows: hooks run under Git Bash if it is installed, otherwise PowerShell; when Claude Code runs PowerShell as a tool the tool is named `PowerShell`, so the matcher includes it.

## The three layers inside the script

1. **Reader** — turns a tool call into a list of `Request(url, method, via)` objects plus notes about what it could not read. For Bash/PowerShell it splits the command at unquoted `; | & newline ( )`, strips `sudo`/`env`/`time`/assignments, follows `bash -c`, `ssh host '...'`, `$(...)` and heredocs, substitutes variables assigned in the same command, and reads the options of curl, wget, httpie, `Invoke-WebRequest`/`Invoke-RestMethod`. For `python -c`, `node -e` and script files it can find the URLs but not the method, so those come back as `UNKNOWN`. For `WebFetch` it reads the URL. For the Claude in Chrome tools it reads `navigate` targets and remembers which host each tab is on, so later `form_input`/`computer` calls on that tab can be judged. For any other MCP tool it records `(server, tool)`.
2. **Findings** — the requests, plus `uncounted` (loops, `xargs`, `wget -r`), `unknown_targets` (a URL held in a variable the command did not set), `browser` events and the `mcp` tool.
3. **Policy** — `policy()` in the script. This is the part you change. It is about seventy lines, reads top to bottom, and is the only place that knows what "changing the application" means; deny beats ask beats silence.

Two design choices worth keeping when you adapt it: **the hook never sleeps or waits for anything** (it is on the critical path of every matching call; the whole run is about 0.03 s on a laptop), and **every refusal says what to do instead**, because the model reads it and a bare "denied" produces retries and workarounds.

## Files

Save each block below as the named file, all in one folder (for example `~/claude-hooks/action-guard/`; on Windows `%USERPROFILE%\claude-hooks\action-guard\`). Plain UTF-8; either line ending works.

### `action_guard.py`

```python
"""action-guard: a Claude Code PreToolUse hook that refuses, or asks about, actions against a web
application the agent works with. Template: keep the skeleton, change the policy.

How Claude Code talks to it (hooks reference, code.claude.com/docs/en/hooks):
  - It runs the command in settings.json before every tool call whose name matches the matcher,
    with one JSON object on stdin: tool_name, tool_input, cwd, session_id, tool_use_id, and
    agent_type when a subagent made the call.
  - Print nothing and exit 0: no objection; the normal permission flow still applies.
    (Never print "allow": that skips the permission prompt.)
  - Print {"hookSpecificOutput": {"hookEventName": "PreToolUse", "permissionDecision": "deny",
    "permissionDecisionReason": "..."}} and exit 0: the call is cancelled and the reason goes to
    the model. "ask" instead of "deny" shows the user the permission prompt.
  - Exit code 2 also blocks, but the settings entry wraps the command in `|| exit 1`, which would
    turn a 2 into a non-blocking 1; so decisions always travel as JSON, and any internal error
    ends in "no output, exit 0" (fail open). A guard that can wedge a session is worse than none.

Sub-commands:
  hook          the hook itself (JSON on stdin)
  check "<cmd>" what the hook would decide for a Bash command, without a session
  status        config in force, tab table, log tail

Config: action_guard.json next to this file, overridden by ~/.claude/action-guard/config.json
(hosts, paths and MCP rules add up across files; every other value replaces).
State and log: ~/.claude/action-guard/.
Stdlib only, Python 3.8+.
"""
import json
import os
import re
import sys
import time
from pathlib import Path

HERE = Path(__file__).resolve().parent
LOG_MAX = 512 * 1024
SUBST = "__AG_SUBST__"
HEREDOC_RE = re.compile(r"^__AG_HEREDOC_(\d+)__$")

BUILTIN = {
    "protected_hosts": [],
    "allowed_methods": ["GET", "HEAD", "OPTIONS"],
    "denied_paths": [],
    "ask_paths": [],
    "write_ok_paths": [],
    "deny_browser_tools_on_protected_hosts": ["form_input", "javascript_tool", "file_upload", "shortcuts_execute"],
    "ask_browser_actions_on_protected_hosts": ["left_click", "right_click", "double_click", "triple_click", "type", "key", "scroll"],
    "mcp_rules": [],
    "on_unreadable_method": "deny",
    "on_unknown_target": "ask",
    "log": True,
}


# ---------------------------------------------------------------- state, log, config

def disabled():
    return os.environ.get("ACTION_GUARD", "").strip().lower() in ("off", "0", "false", "no")


def state_dir():
    d = Path(os.environ.get("ACTION_GUARD_STATE") or Path.home() / ".claude" / "action-guard")
    d.mkdir(parents=True, exist_ok=True)
    return d


def log(line):
    try:
        f = state_dir() / "guard.log"
        if f.exists() and f.stat().st_size > LOG_MAX:
            data = f.read_bytes()[-LOG_MAX // 4:]
            f.write_bytes(data[data.find(b"\n") + 1:])
        with open(f, "a", encoding="utf-8") as fh:
            fh.write(time.strftime("%Y-%m-%d %H:%M:%S ") + line + "\n")
    except Exception:
        pass


ADDITIVE = ("protected_hosts", "denied_paths", "ask_paths", "write_ok_paths", "mcp_rules")


def _merge(base, over):
    """Later config over earlier: objects merge; the ADDITIVE lists add; every other value replaces
    (so a local file can narrow allowed_methods or the browser lists, not only widen them)."""
    for k, v in over.items():
        if isinstance(v, dict) and isinstance(base.get(k), dict):
            _merge(base[k], v)
        elif isinstance(v, list) and isinstance(base.get(k), list) and k in ADDITIVE:
            base[k] = base[k] + [x for x in v if x not in base[k]]
        else:
            base[k] = v
    return base


def load_config():
    cfg = json.loads(json.dumps(BUILTIN))
    for path in (HERE / "action_guard.json",
                 Path(os.environ.get("ACTION_GUARD_CONFIG") or state_dir() / "config.json")):
        if path.exists():
            try:
                _merge(cfg, json.loads(path.read_text(encoding="utf-8")))
            except Exception as ex:           # a broken file must not switch the guard off
                log("config: %s unreadable (%r)" % (path, ex))
    return cfg


class Lock(object):
    """mkdir is atomic on macOS, Linux and Windows; a stale lock is taken over after 30 s."""

    def __init__(self):
        self.path = str(state_dir() / "state.lock")

    def __enter__(self):
        deadline = time.time() + 5
        while True:
            try:
                os.mkdir(self.path)
                return self
            except OSError:
                try:
                    if time.time() - os.stat(self.path).st_mtime > 30:
                        os.rmdir(self.path)
                        continue
                except OSError:
                    pass
                if time.time() > deadline:
                    raise RuntimeError("state lock busy")
                time.sleep(0.05)

    def __exit__(self, *exc):
        try:
            os.rmdir(self.path)
        except OSError:
            pass
        return False


def load_tabs():
    """tab id -> host of the page it was last navigated to (browser tools carry a tabId)."""
    try:
        d = json.loads((state_dir() / "tabs.json").read_text(encoding="utf-8"))
        return d if isinstance(d, dict) else {}
    except Exception:
        return {}


def save_tabs(tabs):
    f = state_dir() / "tabs.json"
    tmp = f.with_suffix(".tmp")
    tmp.write_text(json.dumps(tabs), encoding="utf-8")
    os.replace(str(tmp), str(f))


# ---------------------------------------------------------------- hosts and paths

def host_of(url):
    from urllib.parse import urlsplit
    u = str(url).strip()
    if not re.match(r"^[A-Za-z][A-Za-z0-9+.-]*://", u):
        u = "http://" + u
    try:
        h = urlsplit(u).hostname
    except ValueError:
        return None
    return (h or "").rstrip(".").lower() or None


def path_of(url):
    from urllib.parse import urlsplit
    u = str(url).strip()
    if not re.match(r"^[A-Za-z][A-Za-z0-9+.-]*://", u):
        u = "http://" + u
    try:
        return urlsplit(u).path or "/"
    except ValueError:
        return "/"


def is_protected(host, cfg):
    host = (host or "").lower()
    for key in cfg.get("protected_hosts") or []:
        key = str(key).lower().lstrip(".")
        if key and (host == key or host.endswith("." + key)):
            return True
    return False


def path_matches(path, patterns):
    """'/admin' matches /admin and everything under /admin/; '/adm*' matches any path starting /adm."""
    for pat in patterns or []:
        pat = str(pat)
        if pat.endswith("*"):
            if path.startswith(pat[:-1]):
                return pat
        elif path == pat or path.startswith(pat.rstrip("/") + "/"):
            return pat
    return None


# ---------------------------------------------------------------- what a call would do

class Request(object):
    def __init__(self, url, method, via, unknown=False):
        self.url = url
        self.host = host_of(url) if url else None
        self.path = path_of(url) if url else "/"
        self.method = (method or "GET").upper()
        self.via = via              # "curl", "WebFetch", "navigate" ...
        self.unknown = unknown      # the target or the count could not be read


class Findings(object):
    def __init__(self):
        self.requests = []          # Request objects
        self.uncounted = []         # reasons a request count cannot be read (loop, xargs ...)
        self.unknown_targets = []   # descriptions of requests whose target cannot be read
        self.code_calls = []        # inline code or scripts that talk HTTP (method unreadable)
        self.browser = []           # (short tool name, action, tabId) for Claude in Chrome calls
        self.mcp = None             # (server, tool) for an MCP tool of another server


# --- a small shell reader: enough for curl, wget, httpie and the PowerShell web cmdlets ---

def extract_heredocs(text):
    bodies = []
    pat = re.compile(r"<<-?[ \t]*(?:(['\"])(\w+)\1|\\?(\w+))([^\n]*)\n(.*?)\n[ \t]*(?:\2|\3)[ \t]*(?=\n|$)", re.S)

    def repl(m):
        bodies.append(m.group(5))
        return " __AG_HEREDOC_%d__ %s" % (len(bodies) - 1, m.group(4) or "")
    return pat.sub(repl, text), bodies


def extract_substitutions(text, inner):
    for _ in range(40):
        m = re.search(r"\$\(([^()]*)\)", text)
        if not m:
            break
        inner.append(m.group(1))
        text = text[:m.start()] + SUBST + text[m.end():]
    return re.sub(r"`([^`\n]*)`", lambda m: (inner.append(m.group(1)), SUBST)[1], text)


def split_commands(text, ps=False):
    """Simple commands, split at unquoted ; | & newline ( ) -- and { } in PowerShell."""
    seps = ";|&\n(){}" if ps else ";|&\n()"
    out, buf, q = [], [], None
    i, n = 0, len(text)
    while i < n:
        c = text[i]
        if q:
            buf.append(c)
            if c == "\\" and q == '"' and not ps and i + 1 < n:
                buf.append(text[i + 1])
                i += 2
                continue
            if c == q:
                q = None
        elif c in "'\"":
            q = c
            buf.append(c)
        elif c == "\\" and not ps and i + 1 < n:
            if text[i + 1] != "\n":
                buf.append(c)
                buf.append(text[i + 1])
            i += 2
            continue
        elif c in seps:
            if c == "&" and ((buf and buf[-1] == ">") or (i + 1 < n and text[i + 1] == ">")):
                buf.append(c)
            else:
                seg = "".join(buf).strip()
                if seg:
                    out.append(seg)
                buf = []
                if ps and c in "{}":
                    out.append(c)
        elif c == "#" and (not buf or buf[-1].isspace()):
            j = text.find("\n", i)
            i = n if j < 0 else j
            continue
        else:
            buf.append(c)
        i += 1
    seg = "".join(buf).strip()
    if seg:
        out.append(seg)
    return out


def tokenize(seg, ps=False):
    toks, buf, q, have = [], [], None, False
    i, n = 0, len(seg)
    while i < n:
        c = seg[i]
        if q:
            if c == q:
                q = None
            elif not ps and q == '"' and c == "\\" and i + 1 < n and seg[i + 1] in '"\\$`':
                buf.append(seg[i + 1])
                i += 1
            else:
                buf.append(c)
        elif c in "'\"":
            q = c
            have = True
        elif not ps and c == "\\" and i + 1 < n:
            buf.append(seg[i + 1])
            i += 1
            have = True
        elif c.isspace():
            if buf or have:
                toks.append("".join(buf))
            buf, have = [], False
        else:
            buf.append(c)
        i += 1
    if buf or have:
        toks.append("".join(buf))
    return toks


def substitute(tok, variables):
    if "$" not in tok or not variables:
        return tok

    def repl(m):
        return variables.get((m.group(1) or m.group(2)).lower(), m.group(0))
    for _ in range(2):
        tok = re.sub(r"\$\{(\w+)\}|\$(?:env:)?(\w+)", repl, tok)
    return tok


PREFIXES = frozenset(("sudo", "doas", "time", "nohup", "exec", "command", "env", "nice", "timeout",
                      "then", "do", "else", "elif", "if", "!", "{"))
MULTIPLIERS = frozenset(("xargs", "parallel", "watch"))
LOOPS = frozenset(("for", "while", "until", "foreach", "foreach-object", "%", "select"))
VAR_RE = re.compile(r"\$[\w{(]|__AG_SUBST__|`")
URL_RE = re.compile(r"^(?:https?|ftp)://", re.I)
REDIR_RE = re.compile(r"^(\d*(>>?|<)|&>>?)")


def base(tok):
    b = re.split(r"[\\/]", str(tok))[-1].lower()
    return b[:-4] if b.endswith(".exe") else b


def strip_prefixes(toks):
    """-> (command tokens, reasons the command runs more than once, it starts a loop)."""
    i, many, loop = 0, [], False
    while i < len(toks):
        low = base(toks[i])
        if re.match(r"^\$?[A-Za-z_]\w*=", toks[i]) or low in PREFIXES:
            i += 1
            while i < len(toks) and toks[i].startswith("-"):
                i += 1
            if low == "timeout" and i < len(toks) and re.match(r"^\d+(\.\d+)?[smhd]?$", toks[i]):
                i += 1
        elif low in LOOPS:
            loop = True
            if low in ("for", "foreach", "select"):
                return [], many, True
            i += 1
        elif low in MULTIPLIERS:
            many.append(low)
            i += 1
            while i < len(toks) and toks[i].startswith("-"):
                i += 1
        else:
            break
    return toks[i:], many, loop


def drop_redirections(args):
    out, skip = [], False
    for a in args:
        if skip:
            skip = False
            continue
        m = REDIR_RE.match(a)
        if m:
            skip = m.end() == len(a)
            continue
        out.append(a)
    return out


CURL_VALUE_LONG = frozenset((
    "cacert capath cert ciphers config connect-timeout cookie cookie-jar data data-ascii data-binary data-raw "
    "data-urlencode dump-header form form-string header interface json key limit-rate max-filesize max-redirs "
    "max-time output output-dir proxy proxy-user range referer request resolve retry retry-delay upload-file "
    "url user user-agent write-out").split())
CURL_SHORT_VALUE = "AbcCdDeEFHKmoPQrtTuUwxXyYz"
CURL_METHOD_BY_BODY = {"d": "POST", "data": "POST", "data-ascii": "POST", "data-binary": "POST", "data-raw": "POST",
                       "data-urlencode": "POST", "F": "POST", "form": "POST", "form-string": "POST", "json": "POST",
                       "T": "PUT", "upload-file": "PUT"}


def read_curl(args, f, variables):
    method, targets, unknown = None, [], False
    i, n = 0, len(args)
    while i < n:
        a = args[i]
        if a == "--":
            targets += args[i + 1:]
            break
        if a.startswith("--") and len(a) > 2:
            name, eq, val = a[2:].partition("=")
            if name in CURL_VALUE_LONG:
                if not eq:
                    i += 1
                    val = args[i] if i < n else ""
                if name == "request":
                    method = substitute(val, variables)
                elif name == "url":
                    targets.append(val)
                elif name == "config":
                    unknown = True
                elif name in CURL_METHOD_BY_BODY and not method:
                    method = CURL_METHOD_BY_BODY[name]
            elif name == "head":
                method = method or "HEAD"
        elif a.startswith("-") and len(a) > 1:
            j = 1
            while j < len(a):
                ch = a[j]
                if ch in CURL_SHORT_VALUE:
                    val = a[j + 1:]
                    if not val:
                        i += 1
                        val = args[i] if i < n else ""
                    if ch == "X":
                        method = substitute(val, variables)
                    elif ch == "K":
                        unknown = True
                    elif ch in CURL_METHOD_BY_BODY and not method:
                        method = CURL_METHOD_BY_BODY[ch]
                    break
                if ch == "I":
                    method = method or "HEAD"
                j += 1
        else:
            targets.append(a)
        i += 1
    if unknown:
        f.unknown_targets.append("curl reads its URLs from a config file")
    _add_targets(targets, method or "GET", "curl", f, variables, bare_ok=True)


def read_wget(args, f, variables):
    method, targets = "GET", []
    i, n = 0, len(args)
    while i < n:
        a = args[i]
        if a in ("-r", "--recursive", "-m", "--mirror", "-p", "--page-requisites", "-i", "--input-file"):
            f.uncounted.append("wget %s" % a)
            if a in ("-i", "--input-file"):
                i += 1
        elif a.startswith("--post-data") or a.startswith("--post-file"):
            method = "POST"
            if "=" not in a:
                i += 1
        elif a.startswith("--method"):
            method = a.partition("=")[2] if "=" in a else (args[i + 1] if i + 1 < n else "GET")
            i += 0 if "=" in a else 1
        elif a.startswith("--") and "=" not in a and a[2:] in ("output-document", "output-file", "tries", "timeout", "wait", "user-agent", "header", "referer", "user", "password", "directory-prefix", "load-cookies", "level"):
            i += 1
        elif re.match(r"^-[OoatTwUeDXIlAR]$", a):
            i += 1
        elif a.startswith("-"):
            pass
        else:
            targets.append(a)
        i += 1
    _add_targets(targets, method, "wget", f, variables, bare_ok=True)


HTTP_VERBS = frozenset(("GET", "HEAD", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"))
GENERIC_SKIP = frozenset(("-headers", "-body", "-proxy", "-outfile", "-o", "--output", "--header", "--proxy",
                          "-infile", "-contenttype", "-credential", "-websession", "-sessionvariable", "-form",
                          "-useragent", "--user-agent", "-d", "--auth", "-a"))


def read_generic(name, args, f, variables):
    """httpie (`http POST url`), the PowerShell web cmdlets (`-Method`, `-Body`), lynx, w3m."""
    method, targets, prev, body = None, [], "", False
    for a in args:
        low = a.lower()
        if prev == "-method":
            method = a
        elif prev in GENERIC_SKIP:
            if prev == "-body":
                body = True
        elif a.upper() in HTTP_VERBS and not method and name in ("http", "https", "xh"):
            method = a.upper()
        elif a.startswith("-") and not URL_RE.match(a):
            pass
        else:
            targets.append(a)
        prev = low
    if body and not method:
        method = "POST"
    _add_targets(targets, method or "GET", name, f, variables, bare_ok=False)


def _add_targets(tokens, method, via, f, variables, bare_ok):
    method = substitute(str(method), variables).upper()
    if not tokens:
        f.unknown_targets.append("%s without a written URL (stdin, xargs or a config file)" % via)
    for t in tokens:
        t = substitute(t, variables).strip()
        if not t or HEREDOC_RE.match(t) or t.lower().startswith("file:"):
            continue
        if not URL_RE.match(t) and not bare_ok:
            if VAR_RE.search(t):
                f.unknown_targets.append(t[:60])
            continue
        hostpart = re.match(r"^(?:[A-Za-z][\w+.-]*://)?([^/?#]*)", t).group(1)
        if VAR_RE.search(hostpart) or not hostpart:
            f.unknown_targets.append(t[:60])
            continue
        f.requests.append(Request(t if URL_RE.match(t) else "http://" + t, method, via, unknown=VAR_RE.search(method) is not None))


HTTP_CODE = re.compile(r"\brequests\s*\.\s*(?:get|post|put|patch|delete|head|request|Session)\b|\burllib\.request\b|"
                       r"\burlopen\s*\(|\bhttpx\b|\baiohttp\b|\bfetch\s*\(|\baxios\b|\bhttps?\.(?:get|request)\s*\(|"
                       r"\bInvoke-(?:WebRequest|RestMethod)\b|\bWebClient\b|\bHttpClient\b")
CODE_INTERP = re.compile(r"^(python|pypy|py|node|nodejs|deno|bun|ruby|perl|php)[\d.]*$")
SHELL_INTERP = re.compile(r"^(bash|sh|zsh|dash|ksh|pwsh|powershell)$")
CLIENTS = {"curl": read_curl, "wget": read_wget}
GENERIC_CLIENTS = frozenset(("http", "https", "xh", "lynx", "w3m", "invoke-webrequest", "invoke-restmethod", "iwr", "irm"))


def read_code(text, f, where):
    """Inline code or a script: the method cannot be read reliably, so record every URL it holds."""
    if not HTTP_CODE.search(text):
        return
    urls = re.findall(r"(?:https?)://[^\s\"'<>\\)`|;]+", text)
    f.code_calls.append(where)
    for u in urls:
        f.requests.append(Request(u.rstrip(".,;:"), "UNKNOWN", where, unknown=True))
    if not urls:
        f.unknown_targets.append("%s takes its target at run time" % where)


def scan(text, f, cwd, ps=False, depth=0, variables=None):
    if depth > 4:
        return
    variables = variables if variables is not None else {}
    inner = []
    bodies = []
    if not ps:
        text, bodies = extract_heredocs(text)
    text = extract_substitutions(text, inner)
    for sub in inner:
        scan(sub, f, cwd, ps, depth + 1, variables)
    segments = [tokenize(s, ps) for s in split_commands(text, ps)]
    for toks in segments:                                  # NAME=value assignments, for "$NAME/path"
        for t in toks:
            m = re.match(r"^\$?([A-Za-z_]\w*)=(.*)$", t)
            if m:
                variables[m.group(1).lower()] = substitute(m.group(2), variables)
            else:
                break
    loops = 0
    for toks in segments:
        if toks and toks[0] == "done":
            loops = max(0, loops - 1)
        toks, many, loop = strip_prefixes(toks)
        if loop:
            loops += 1
        if not toks:
            continue
        name, args = base(toks[0]), drop_redirections(toks[1:])
        before = len(f.requests)
        if name in CLIENTS and not ps:
            CLIENTS[name](args, f, variables)
        elif name in GENERIC_CLIENTS or name in CLIENTS:
            read_generic(name, args, f, variables)
        elif CODE_INTERP.match(name):
            code = None
            for k, a in enumerate(args):
                if a in ("-c", "-e", "--eval", "-r") and k + 1 < len(args):
                    code = args[k + 1]
                    break
                if HEREDOC_RE.match(a):
                    code = bodies[int(HEREDOC_RE.match(a).group(1))]
                    break
                if not a.startswith("-"):
                    p = Path(a) if os.path.isabs(a) else Path(cwd or ".") / a
                    if p.is_file() and p.stat().st_size < 512 * 1024:
                        code = p.read_text(encoding="utf-8", errors="replace")
                    break
            if code:
                read_code(code, f, "%s code" % name)
        elif SHELL_INTERP.match(name):
            for k, a in enumerate(args):
                if (re.match(r"^-[A-Za-z]*c$", a) or a.lower() == "-command") and k + 1 < len(args):
                    sub_ps = name in ("pwsh", "powershell")
                    scan(" ".join(args[k + 1:]) if sub_ps else args[k + 1], f, cwd, sub_ps, depth + 1, variables)
                    break
        elif name == "ssh" and len(args) > 1:
            rest = [a for a in args if not a.startswith("-")]
            if len(rest) > 1:
                scan(" ".join(rest[1:]), f, cwd, ps, depth + 1, variables)
        if len(f.requests) > before and (loops > 0 or many):
            f.uncounted.append("a loop" if loops else ", ".join(many))


# ---------------------------------------------------------------- reading a tool call

def findings_for(inp, cfg, tabs):
    """-> Findings: the requests a call would send, browser events on known tabs, or an MCP tool."""
    tool = str(inp.get("tool_name") or "")
    ti = inp.get("tool_input") or {}
    f = Findings()
    if not isinstance(ti, dict):
        return f
    if tool == "WebFetch":
        if ti.get("url"):
            f.requests.append(Request(str(ti["url"]), "GET", "WebFetch"))
    elif tool in ("Bash", "PowerShell"):
        cmd = ti.get("command")
        if isinstance(cmd, str) and cmd.strip():
            scan(cmd, f, str(inp.get("cwd") or ""), ps=(tool == "PowerShell"))
    elif tool.startswith("mcp__claude-in-chrome__"):
        short = tool.rsplit("__", 1)[-1]
        calls = [(short, ti)]
        if short == "browser_batch":
            calls = [(str(a.get("name") or "").rsplit("__", 1)[-1], a.get("input") or {})
                     for a in (ti.get("actions") or []) if isinstance(a, dict)]
        for name, args in calls:
            args = args if isinstance(args, dict) else {}
            tab = str(args.get("tabId") or "")
            if name == "navigate":
                url = str(args.get("url") or "")
                if url and url.lower() not in ("back", "forward"):
                    r = Request(url if URL_RE.match(url) else "https://" + url, "GET", "navigate")
                    f.requests.append(r)
                    if tab and r.host:
                        tabs[tab] = r.host
            else:
                f.browser.append((name, str(args.get("action") or ""), tab))
    elif tool.startswith("mcp__"):
        parts = tool.split("__", 2)
        if len(parts) == 3:
            f.mcp = (parts[1], parts[2])
    return f


# ---------------------------------------------------------------- the policy (change this part)

def _deny(reason):
    return {"hookSpecificOutput": {"hookEventName": "PreToolUse", "permissionDecision": "deny",
                                   "permissionDecisionReason": "action-guard: refused. " + reason},
            "systemMessage": "action-guard: held back a call -- " + reason[:120]}


def _ask(reason):
    return {"hookSpecificOutput": {"hookEventName": "PreToolUse", "permissionDecision": "ask",
                                   "permissionDecisionReason": "action-guard: " + reason}}


def policy(f, cfg, tabs, tag):
    """Decide. Return None (no opinion), or the JSON to print. Order: deny beats ask beats silence."""
    allowed = set(m.upper() for m in cfg.get("allowed_methods") or [])
    protected = [r for r in f.requests if is_protected(r.host, cfg)]
    ask = None

    # 1. Requests to the protected application, read from the call
    for r in protected:
        hit = path_matches(r.path, cfg.get("denied_paths"))
        if hit:
            log("%s DENY path %s %s" % (tag, r.method, r.url))
            return _deny("%s matches the protected path pattern %r on %s. That part of the application is "
                         "off limits to this session. Do not try another address or another tool for it; "
                         "tell the user what you needed." % (r.path, hit, r.host))
        if r.unknown:
            if cfg.get("on_unreadable_method") == "ask":
                ask = ask or _ask("a request to %s whose method the guard cannot read (made from code); "
                                  "the user decides." % r.host)
                continue
            log("%s DENY unreadable %s" % (tag, r.url))
            return _deny("a request to %s made from code, whose method the guard cannot read. Against this "
                         "application use curl (or WebFetch) with the URL written out, one request per call, "
                         "so the guard can see the method." % r.host)
        if r.method not in allowed and not path_matches(r.path, cfg.get("write_ok_paths")):
            log("%s DENY method %s %s" % (tag, r.method, r.url))
            article = "an" if r.method[:1] in "AEIOU" else "a"
            return _deny("%s %s request to %s. Only %s requests are allowed against this application from a "
                         "session; anything that changes its state is for the user to do. Say what change you "
                         "wanted." % (article, r.method, r.host, ", ".join(sorted(allowed))))
        hit = path_matches(r.path, cfg.get("ask_paths"))
        if hit:
            ask = ask or _ask("%s %s on %s matches the pattern %r, which needs the user's approval."
                              % (r.method, r.path, r.host, hit))
    if protected and f.uncounted:
        log("%s DENY uncounted" % tag)
        return _deny("this call would send an unreadable number of requests to %s (%s). Send them one per "
                     "call, each URL written out." % (protected[0].host, "; ".join(f.uncounted)[:160]))
    if f.unknown_targets:
        # A request whose target cannot be read might be aimed at the protected application.
        what = "; ".join(f.unknown_targets)[:160]
        if cfg.get("on_unknown_target") == "deny":
            log("%s DENY unknown target" % tag)
            return _deny("a request whose target the guard cannot read (%s). Write the URL out in the call." % what)
        if cfg.get("on_unknown_target") == "ask":
            ask = ask or _ask("a request whose target the guard cannot read (%s); the user decides." % what)

    # 2. Browser actions on a tab that last navigated to the protected application
    for name, action, tab in getattr(f, "browser", []):
        host = tabs.get(tab)
        if not is_protected(host, cfg):
            continue
        if name in (cfg.get("deny_browser_tools_on_protected_hosts") or []):
            log("%s DENY browser %s tab=%s host=%s" % (tag, name, tab, host))
            return _deny("%s on a page of %s. Changing that application through the browser is for the user; "
                         "reading it (screenshots, page text, find) is fine." % (name, host))
        if name == "computer" and action in (cfg.get("ask_browser_actions_on_protected_hosts") or []):
            ask = ask or _ask("a %s in the browser on a page of %s; the user decides whether that action is allowed." % (action, host))

    # 3. MCP tools of the application's own server
    if f.mcp:
        server, tool = f.mcp
        for rule in cfg.get("mcp_rules") or []:
            if str(rule.get("server", "")).lower() != server.lower():
                continue
            if tool in (rule.get("deny_tools") or []):
                log("%s DENY mcp %s/%s" % (tag, server, tool))
                return _deny("the tool %s of server %s is off limits to this session." % (tool, server))
            if tool in (rule.get("ask_tools") or []):
                ask = ask or _ask("the tool %s of server %s needs the user's approval." % (tool, server))
    if ask:
        log("%s ASK %s" % (tag, ask["hookSpecificOutput"]["permissionDecisionReason"][:100]))
    return ask


# ---------------------------------------------------------------- entry points

def do_hook(inp):
    if disabled():
        return None
    cfg = load_config()
    with Lock():
        tabs = load_tabs()
        f = findings_for(inp, cfg, tabs)
        save_tabs(tabs)
    sid = "".join(ch for ch in str(inp.get("session_id") or "?") if ch.isalnum())[:8]
    tag = "hook sid=%s who=%s tool=%s" % (sid, inp.get("agent_type") or "main", str(inp.get("tool_name") or "?").rsplit("__", 1)[-1])
    return policy(f, cfg, tabs, tag)


def do_check(args):
    cfg = load_config()
    for cmd in args:
        f = findings_for({"tool_name": "Bash", "tool_input": {"command": cmd}, "cwd": os.getcwd()}, cfg, {})
        out = policy(f, cfg, {}, "check")
        for r in f.requests:
            print("  %-7s %s  (%s)%s" % (r.method, r.url, r.via, "  protected" if is_protected(r.host, cfg) else ""))
        print("%s -> %s" % (cmd[:70], "no opinion" if out is None else out["hookSpecificOutput"]["permissionDecision"] + ": " + out["hookSpecificOutput"]["permissionDecisionReason"]))
    return 0


def do_status():
    cfg = load_config()
    print("action-guard  disabled-by-env=%s  state=%s" % (disabled(), state_dir()))
    print("protected hosts: %s" % (", ".join(cfg.get("protected_hosts") or []) or "(none: the guard does nothing)"))
    print("allowed methods: %s | denied paths: %s | ask paths: %s | writes allowed under: %s"
          % (cfg.get("allowed_methods"), cfg.get("denied_paths"), cfg.get("ask_paths"), cfg.get("write_ok_paths")))
    print("browser tools denied on protected pages: %s" % cfg.get("deny_browser_tools_on_protected_hosts"))
    print("tabs known: %s" % (load_tabs() or "{}"))
    lg = state_dir() / "guard.log"
    if lg.exists():
        for ln in lg.read_text(encoding="utf-8", errors="replace").splitlines()[-8:]:
            print("  " + ln)
    return 0


def main(argv):
    cmd = (argv[1] if len(argv) > 1 else "").strip().lower()
    if cmd == "check":
        return do_check(argv[2:])
    if cmd == "status":
        return do_status()
    if cmd != "hook":
        sys.stderr.write("usage: action_guard.py hook  (hook JSON on stdin) | check '<command>' | status\n")
        return 0
    raw = ""
    try:
        if not sys.stdin.isatty():
            raw = sys.stdin.buffer.read().decode("utf-8", "replace")
    except Exception:
        raw = ""
    try:
        inp = json.loads(raw) if raw.strip() else {}
    except ValueError:
        inp = {}
    out = None
    try:
        out = do_hook(inp if isinstance(inp, dict) else {})
    except Exception as ex:                    # fail open, never lock a session
        log("hook ERROR %r" % (ex,))
        out = None
    if out:
        sys.stdout.write(json.dumps(out))
        sys.stdout.flush()
    return 0


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

### `action_guard.json`

Edit this one. The example protects `app.example.com` (and, by suffix, `api.app.example.com`), allows reads, allows writes under `/api/v1/comments` only, refuses anything under `/admin`, `/api/v1/users` and `/delete`, asks about `/api/v1/orders`, and names one MCP server.

```json
{
  "_about": "Policy for action-guard. Hosts are matched by suffix (app.example.com also covers api.app.example.com). Paths are prefix patterns: /admin covers /admin and everything under it, /adm* covers any path starting with /adm. Machine-local overrides go in ~/.claude/action-guard/config.json: protected_hosts, denied_paths, ask_paths and mcp_rules add up; every other value replaces. on_unreadable_method and on_unknown_target take deny, ask or ignore.",

  "protected_hosts": ["app.example.com"],

  "allowed_methods": ["GET", "HEAD", "OPTIONS"],

  "denied_paths": ["/admin", "/api/v1/users", "/delete"],

  "ask_paths": ["/api/v1/orders"],

  "write_ok_paths": ["/api/v1/comments"],

  "deny_browser_tools_on_protected_hosts": ["form_input", "javascript_tool", "file_upload", "shortcuts_execute"],

  "ask_browser_actions_on_protected_hosts": ["left_click", "right_click", "double_click", "triple_click", "type", "key"],

  "mcp_rules": [
    {"server": "myapp", "deny_tools": ["delete_item", "purge"], "ask_tools": ["update_item", "create_item"]}
  ],

  "log": true
}
```

Keys:

| Key | Meaning |
|---|---|
| `protected_hosts` | Hosts the policy applies to, matched by suffix. Everything else is ignored by this guard. |
| `allowed_methods` | Methods that pass without a prompt. |
| `denied_paths` | Path prefixes refused with any method (`/admin` covers `/admin` and `/admin/...`; `/adm*` covers anything starting `/adm`). |
| `ask_paths` | Path prefixes that always prompt. |
| `write_ok_paths` | Path prefixes where any method is allowed (writes the agent is supposed to make). |
| `deny_browser_tools_on_protected_hosts` | Claude in Chrome tools refused on a tab that is on the application. |
| `ask_browser_actions_on_protected_hosts` | `computer` actions that prompt on such a tab (clicks, typing, keys). Screenshots, scrolling, reading always pass. |
| `mcp_rules` | Per MCP server: `deny_tools` and `ask_tools`. |
| `on_unreadable_method` | `deny` (default), `ask` or `ignore` for a request to the application made from code. |
| `on_unknown_target` | `ask` (default), `deny` or `ignore` for a request whose host cannot be read. |

A second file, `~/.claude/action-guard/config.json`, is read after this one when it exists: hosts, paths and MCP rules add up; every other value replaces. Use it for per-machine differences, or put the whole policy there and leave `action_guard.json` empty.

### `install.py`

```python
"""Install, verify or remove the action-guard hook in ~/.claude/settings.json.

    python3 install.py            # install / update (idempotent; backs up settings.json first)
    python3 install.py --dry-run  # show what would change
    python3 install.py --uninstall
    (Windows: py install.py ...)

Adds one PreToolUse group whose matcher covers the shell tools, WebFetch, the Claude in Chrome
browser tools and every MCP tool, running action_guard.py from this folder as
`<interpreter> "<script>" hook || exit 1`. Every other key in settings.json is preserved.
Set CLAUDE_SETTINGS to install into another file (used by the tests).
"""
import json
import os
import shutil
import sys
import time
from pathlib import Path

HERE = Path(__file__).resolve().parent
SCRIPT = HERE / "action_guard.py"
MARK = "action_guard.py"
MATCHER = "Bash|PowerShell|WebFetch|mcp__.*"


def command():
    if os.name == "nt":
        return 'py "%s" hook || exit 1' % str(SCRIPT).replace("\\", "/")
    return "python3 '%s' hook || exit 1" % SCRIPT


def strip_ours(groups):
    out = []
    for g in groups or []:
        hs = [h for h in (g.get("hooks") or []) if MARK not in str(h.get("command") or "").replace("\\", "/")]
        if hs:
            g2 = dict(g)
            g2["hooks"] = hs
            out.append(g2)
    return out


def main(argv):
    dry = "--dry-run" in argv
    remove = "--uninstall" in argv
    settings = Path(os.environ.get("CLAUDE_SETTINGS") or Path.home() / ".claude" / "settings.json")
    data = json.loads(settings.read_text(encoding="utf-8")) if settings.exists() else {}
    before = json.dumps(data.get("hooks", {}), sort_keys=True)
    hooks = data.setdefault("hooks", {})
    groups = strip_ours(hooks.get("PreToolUse"))
    if not remove:
        groups.append({"matcher": MATCHER, "hooks": [
            {"type": "command", "command": command(), "timeout": 20, "statusMessage": "action-guard: checking the call..."}]})
    if groups:
        hooks["PreToolUse"] = groups
    else:
        hooks.pop("PreToolUse", None)
    if not hooks:
        data.pop("hooks", None)
    if before == json.dumps(data.get("hooks", {}), sort_keys=True):
        print("action-guard: %s -- no change needed" % settings)
        return 0
    print("%s PreToolUse [%s] -> %s" % ("remove" if remove else "set", MATCHER, "(dropped)" if remove else command()))
    if dry:
        print("dry run: %s not written" % settings)
        return 0
    if settings.exists():
        bak = settings.with_name("settings.json.bak-" + time.strftime("%Y%m%d-%H%M%S"))
        shutil.copy2(str(settings), str(bak))
        print("backup: %s" % bak)
    tmp = settings.with_suffix(".json.tmp")
    tmp.write_text(json.dumps(data, indent=2, ensure_ascii=False) + "\n", encoding="utf-8")
    os.replace(str(tmp), str(settings))
    json.loads(settings.read_text(encoding="utf-8"))      # re-parse: never leave a broken settings file
    print("written: %s" % settings)
    return 0


if __name__ == "__main__":
    sys.exit(main(sys.argv[1:]))
```

### `test_action_guard.py`

```python
"""Tests for action-guard. Run:  python3 test_action_guard.py   (Windows: py test_action_guard.py)

Everything runs against a scratch state directory and the policy in TEST_CONFIG, never against
~/.claude. The hook tests start action_guard.py as a subprocess the way Claude Code does.
"""
import json
import os
import subprocess
import sys
import tempfile
import unittest
from pathlib import Path

HERE = Path(__file__).resolve().parent
SCRIPT = HERE / "action_guard.py"
STATE = tempfile.mkdtemp(prefix="action-guard-test-")
CONFIG = Path(STATE) / "test-config.json"
TEST_CONFIG = {
    "protected_hosts": ["app.example.com"],
    "allowed_methods": ["GET", "HEAD", "OPTIONS"],
    "denied_paths": ["/admin", "/api/v1/users"],
    "ask_paths": ["/api/v1/orders"],
    "write_ok_paths": ["/api/v1/comments"],
    "mcp_rules": [{"server": "myapp", "deny_tools": ["delete_item"], "ask_tools": ["update_item"]}],
}
CONFIG.write_text(json.dumps(TEST_CONFIG), encoding="utf-8")
os.environ["ACTION_GUARD_STATE"] = STATE
os.environ["ACTION_GUARD_CONFIG"] = str(CONFIG)
os.environ.pop("ACTION_GUARD", None)
sys.path.insert(0, str(HERE))
import action_guard as ag  # noqa: E402

CFG = ag.load_config()
PY = sys.executable


def decide(command, tool="Bash", tabs=None):
    inp = {"tool_name": tool, "tool_input": {"command": command}, "cwd": STATE}
    if tool == "WebFetch":
        inp["tool_input"] = {"url": command}
    tabs = tabs if tabs is not None else {}
    f = ag.findings_for(inp, CFG, tabs)
    out = ag.policy(f, CFG, tabs, "test")
    return None if out is None else out["hookSpecificOutput"]["permissionDecision"], f


class Hosts(unittest.TestCase):
    def test_protected_by_suffix(self):
        self.assertTrue(ag.is_protected("app.example.com", CFG))
        self.assertTrue(ag.is_protected("api.app.example.com", CFG))
        self.assertFalse(ag.is_protected("example.com", CFG))
        self.assertFalse(ag.is_protected("notapp.example.com", CFG))
        self.assertFalse(ag.is_protected(None, CFG))

    def test_host_and_path(self):
        self.assertEqual(ag.host_of("https://App.Example.com:8443/x?y=1"), "app.example.com")
        self.assertEqual(ag.host_of("app.example.com/admin"), "app.example.com")
        self.assertEqual(ag.path_of("https://app.example.com"), "/")
        self.assertEqual(ag.path_of("https://app.example.com/api/v1/users?x=1"), "/api/v1/users")

    def test_config_merge(self):
        base = {"protected_hosts": ["a.example.com"], "allowed_methods": ["GET", "HEAD", "OPTIONS"], "log": True}
        ag._merge(base, {"protected_hosts": ["b.example.com"], "allowed_methods": ["GET"], "log": False})
        self.assertEqual(base["protected_hosts"], ["a.example.com", "b.example.com"])   # adds
        self.assertEqual(base["allowed_methods"], ["GET"])                             # replaces
        self.assertIs(base["log"], False)

    def test_path_patterns(self):
        self.assertEqual(ag.path_matches("/admin", ["/admin"]), "/admin")
        self.assertEqual(ag.path_matches("/admin/users", ["/admin"]), "/admin")
        self.assertIsNone(ag.path_matches("/administrator", ["/admin"]))
        self.assertEqual(ag.path_matches("/administrator", ["/admin*"]), "/admin*")
        self.assertIsNone(ag.path_matches("/api/v2/users", ["/api/v1/users"]))


class Reading(unittest.TestCase):
    """What the reader sees in a shell command: method and target."""

    def seen(self, command, ps=False):
        f = ag.Findings()
        ag.scan(command, f, STATE, ps=ps)
        return [(r.method, r.url) for r in f.requests], f

    def test_curl_methods(self):
        cases = [
            ("curl https://app.example.com/api/items", "GET"),
            ("curl -s -o out.json https://app.example.com/api/items", "GET"),
            ("curl -I https://app.example.com/", "HEAD"),
            ("curl --head https://app.example.com/", "HEAD"),
            ("curl -X POST https://app.example.com/api/items -d '{}'", "POST"),
            ("curl -XDELETE https://app.example.com/api/items/3", "DELETE"),
            ("curl --request PATCH https://app.example.com/api/items/3", "PATCH"),
            ("curl -d 'a=1' https://app.example.com/api/items", "POST"),
            ("curl --data-binary @file https://app.example.com/api/items", "POST"),
            ("curl -F file=@x.png https://app.example.com/upload", "POST"),
            ("curl --json '{\"a\":1}' https://app.example.com/api/items", "POST"),
            ("curl -T local.txt https://app.example.com/files/", "PUT"),
            ("curl -sS -H 'Accept: application/json' https://app.example.com/api/items", "GET"),
            ("curl -u user:pass -X PUT https://app.example.com/api/items/3", "PUT"),
        ]
        for cmd, method in cases:
            seen, _ = self.seen(cmd)
            self.assertEqual(len(seen), 1, cmd)
            self.assertEqual(seen[0][0], method, cmd)
            self.assertTrue(seen[0][1].startswith("https://app.example.com"), cmd)

    def test_bare_host_and_variables(self):
        seen, _ = self.seen("curl app.example.com/admin")
        self.assertEqual(seen, [("GET", "http://app.example.com/admin")])
        seen, _ = self.seen('BASE=https://app.example.com; curl -X POST "$BASE/api/items"')
        self.assertEqual(seen, [("POST", "https://app.example.com/api/items")])
        seen, _ = self.seen('M=DELETE; curl -X $M https://app.example.com/api/items/1')
        self.assertEqual(seen, [("DELETE", "https://app.example.com/api/items/1")])
        seen, f = self.seen('curl "$URL"')
        self.assertEqual(seen, [])
        self.assertTrue(f.unknown_targets)

    def test_wget_httpie_powershell(self):
        seen, _ = self.seen("wget -q https://app.example.com/report.csv")
        self.assertEqual(seen, [("GET", "https://app.example.com/report.csv")])
        seen, _ = self.seen("wget --post-data 'a=1' https://app.example.com/api/items")
        self.assertEqual(seen[0][0], "POST")
        seen, _ = self.seen("http POST https://app.example.com/api/items name=x")
        self.assertEqual(seen[0][0], "POST")
        seen, _ = self.seen("http https://app.example.com/api/items")
        self.assertEqual(seen[0][0], "GET")
        seen, _ = self.seen("Invoke-RestMethod -Uri https://app.example.com/api/items -Method Delete", ps=True)
        self.assertEqual(seen, [("DELETE", "https://app.example.com/api/items")])
        seen, _ = self.seen("iwr https://app.example.com/api/items -Body $b", ps=True)
        self.assertEqual(seen[0][0], "POST")
        seen, _ = self.seen("Invoke-WebRequest https://app.example.com/page -OutFile p.html", ps=True)
        self.assertEqual(seen, [("GET", "https://app.example.com/page")])

    def test_code_is_unreadable(self):
        seen, f = self.seen("python3 -c \"import requests; requests.post('https://app.example.com/api/items', json={})\"")
        self.assertEqual(seen, [("UNKNOWN", "https://app.example.com/api/items")])
        self.assertTrue(f.requests[0].unknown)
        seen, f = self.seen("node -e \"fetch('https://app.example.com/api/items', {method: 'DELETE'})\"")
        self.assertEqual(seen[0][0], "UNKNOWN")
        seen, f = self.seen("python3 - <<'EOF'\nimport urllib.request\nurllib.request.urlopen('https://app.example.com/x')\nEOF")
        self.assertEqual(seen, [("UNKNOWN", "https://app.example.com/x")])
        seen, f = self.seen("python3 -c \"import requests; requests.get(url)\"")
        self.assertEqual(seen, [])
        self.assertTrue(f.unknown_targets)

    def test_loops_and_wrappers(self):
        _, f = self.seen("for i in 1 2 3; do curl https://app.example.com/api/items/$i; done")
        self.assertEqual(len(f.requests), 1)
        self.assertTrue(f.uncounted)
        _, f = self.seen("cat ids.txt | xargs -I{} curl -X DELETE https://app.example.com/api/items/{}")
        self.assertTrue(f.uncounted)
        _, f = self.seen("cat urls.txt | xargs curl -s")
        self.assertTrue(f.unknown_targets)
        _, f = self.seen("bash -c 'curl -X POST https://app.example.com/api/items'")
        self.assertEqual(f.requests[0].method, "POST")
        _, f = self.seen("ssh box 'curl -X POST https://app.example.com/api/items'")
        self.assertEqual(f.requests[0].method, "POST")
        _, f = self.seen("x=$(curl -X POST https://app.example.com/api/items); echo $x")
        self.assertEqual(f.requests[0].method, "POST")
        _, f = self.seen("echo 'curl -X POST https://app.example.com/api/items'")
        self.assertEqual(f.requests, [])
        _, f = self.seen("# curl -X POST https://app.example.com/api/items\nls")
        self.assertEqual(f.requests, [])


class Policy(unittest.TestCase):
    def test_reads_pass(self):
        for cmd in ("curl https://app.example.com/api/items",
                    "curl -I https://app.example.com/",
                    "wget -q https://app.example.com/report.csv",
                    "git status",
                    "curl -X POST https://other.example.org/api/items -d '{}'"):
            self.assertIsNone(decide(cmd)[0], cmd)
        self.assertIsNone(decide("https://app.example.com/api/items", tool="WebFetch")[0])

    def test_writes_denied(self):
        for cmd in ("curl -X POST https://app.example.com/api/items -d '{}'",
                    "curl -X DELETE https://api.app.example.com/items/3",
                    "curl -d 'a=1' app.example.com/api/items",
                    "curl -T f.txt https://app.example.com/files/",
                    "python3 -c \"import requests; requests.post('https://app.example.com/api/items')\"",
                    "for i in 1 2; do curl https://app.example.com/api/items/$i; done"):
            self.assertEqual(decide(cmd)[0], "deny", cmd)

    def test_paths(self):
        self.assertEqual(decide("curl https://app.example.com/admin/settings")[0], "deny")
        self.assertEqual(decide("curl https://app.example.com/api/v1/users")[0], "deny")
        self.assertEqual(decide("curl https://app.example.com/api/v1/orders/7")[0], "ask")
        self.assertIsNone(decide("curl https://app.example.com/api/v1/products")[0])
        self.assertEqual(decide("https://app.example.com/admin", tool="WebFetch")[0], "deny")
        self.assertIsNone(decide("curl -X POST https://app.example.com/api/v1/comments -d 'text=hi'")[0])
        self.assertEqual(decide("curl -X POST https://app.example.com/api/v1/commentsX -d 'text=hi'")[0], "deny")

    def test_unknown_target_asks(self):
        self.assertEqual(decide('curl "$URL"')[0], "ask")
        self.assertEqual(decide("cat urls.txt | xargs curl -s")[0], "ask")

    def test_deny_beats_ask(self):
        out = decide("curl https://app.example.com/api/v1/orders/1 && curl -X POST https://app.example.com/api/v1/orders")
        self.assertEqual(out[0], "deny")

    def test_browser(self):
        tabs = {}
        nav = {"tool_name": "mcp__claude-in-chrome__navigate", "tool_input": {"url": "https://app.example.com/items", "tabId": 7}}
        f = ag.findings_for(nav, CFG, tabs)
        self.assertIsNone(ag.policy(f, CFG, tabs, "t"))
        self.assertEqual(tabs, {"7": "app.example.com"})
        nav_admin = {"tool_name": "mcp__claude-in-chrome__navigate", "tool_input": {"url": "https://app.example.com/admin", "tabId": 7}}
        self.assertEqual(ag.policy(ag.findings_for(nav_admin, CFG, tabs), CFG, tabs, "t")["hookSpecificOutput"]["permissionDecision"], "deny")
        form = {"tool_name": "mcp__claude-in-chrome__form_input", "tool_input": {"tabId": 7, "fields": []}}
        self.assertEqual(ag.policy(ag.findings_for(form, CFG, tabs), CFG, tabs, "t")["hookSpecificOutput"]["permissionDecision"], "deny")
        click = {"tool_name": "mcp__claude-in-chrome__computer", "tool_input": {"tabId": 7, "action": "left_click", "coordinate": [1, 1]}}
        self.assertEqual(ag.policy(ag.findings_for(click, CFG, tabs), CFG, tabs, "t")["hookSpecificOutput"]["permissionDecision"], "ask")
        shot = {"tool_name": "mcp__claude-in-chrome__computer", "tool_input": {"tabId": 7, "action": "screenshot"}}
        self.assertIsNone(ag.policy(ag.findings_for(shot, CFG, tabs), CFG, tabs, "t"))
        text = {"tool_name": "mcp__claude-in-chrome__get_page_text", "tool_input": {"tabId": 7}}
        self.assertIsNone(ag.policy(ag.findings_for(text, CFG, tabs), CFG, tabs, "t"))
        other_tab = {"tool_name": "mcp__claude-in-chrome__form_input", "tool_input": {"tabId": 8, "fields": []}}
        self.assertIsNone(ag.policy(ag.findings_for(other_tab, CFG, tabs), CFG, tabs, "t"))
        batch = {"tool_name": "mcp__claude-in-chrome__browser_batch", "tool_input": {"actions": [
            {"name": "navigate", "input": {"url": "https://app.example.com/x", "tabId": 9}},
            {"name": "form_input", "input": {"tabId": 9, "fields": []}}]}}
        self.assertEqual(ag.policy(ag.findings_for(batch, CFG, tabs), CFG, tabs, "t")["hookSpecificOutput"]["permissionDecision"], "deny")

    def test_mcp_rules(self):
        d = {"tool_name": "mcp__myapp__delete_item", "tool_input": {"id": 3}}
        self.assertEqual(ag.policy(ag.findings_for(d, CFG, {}), CFG, {}, "t")["hookSpecificOutput"]["permissionDecision"], "deny")
        u = {"tool_name": "mcp__myapp__update_item", "tool_input": {"id": 3}}
        self.assertEqual(ag.policy(ag.findings_for(u, CFG, {}), CFG, {}, "t")["hookSpecificOutput"]["permissionDecision"], "ask")
        r = {"tool_name": "mcp__myapp__read_item", "tool_input": {"id": 3}}
        self.assertIsNone(ag.policy(ag.findings_for(r, CFG, {}), CFG, {}, "t"))
        o = {"tool_name": "mcp__otherserver__delete_item", "tool_input": {}}
        self.assertIsNone(ag.policy(ag.findings_for(o, CFG, {}), CFG, {}, "t"))


class Hook(unittest.TestCase):
    """The script as Claude Code runs it: JSON in on stdin, JSON or nothing out, exit 0."""

    def run_hook(self, inp, env_extra=None):
        env = dict(os.environ)
        env.update(env_extra or {})
        p = subprocess.run([PY, str(SCRIPT), "hook"], input=json.dumps(inp).encode(), stdout=subprocess.PIPE,
                           stderr=subprocess.PIPE, env=env, timeout=30)
        return p.returncode, p.stdout.decode("utf-8", "replace"), p.stderr.decode("utf-8", "replace")

    def test_deny_json(self):
        code, out, err = self.run_hook({"session_id": "abc", "tool_name": "Bash",
                                        "tool_input": {"command": "curl -X POST https://app.example.com/api/items"}})
        self.assertEqual(code, 0, err)
        data = json.loads(out)
        self.assertEqual(data["hookSpecificOutput"]["permissionDecision"], "deny")
        self.assertIn("POST", data["hookSpecificOutput"]["permissionDecisionReason"])

    def test_silent_pass(self):
        code, out, err = self.run_hook({"tool_name": "Bash", "tool_input": {"command": "curl https://app.example.com/api/items"}})
        self.assertEqual((code, out), (0, ""), err)
        code, out, err = self.run_hook({"tool_name": "Read", "tool_input": {"file_path": "/x"}})
        self.assertEqual((code, out), (0, ""))

    def test_disabled_and_broken_input(self):
        code, out, _ = self.run_hook({"tool_name": "Bash", "tool_input": {"command": "curl -X POST https://app.example.com/a"}},
                                     {"ACTION_GUARD": "off"})
        self.assertEqual((code, out), (0, ""))
        p = subprocess.run([PY, str(SCRIPT), "hook"], input=b"not json", stdout=subprocess.PIPE, stderr=subprocess.PIPE, timeout=30)
        self.assertEqual((p.returncode, p.stdout), (0, b""))
        p = subprocess.run([PY, str(SCRIPT), "hook"], input=b"", stdout=subprocess.PIPE, stderr=subprocess.PIPE, timeout=30)
        self.assertEqual((p.returncode, p.stdout), (0, b""))

    def test_tab_memory_persists_between_calls(self):
        self.run_hook({"tool_name": "mcp__claude-in-chrome__navigate", "tool_input": {"url": "https://app.example.com/", "tabId": 41}})
        code, out, _ = self.run_hook({"tool_name": "mcp__claude-in-chrome__form_input", "tool_input": {"tabId": 41, "fields": []}})
        self.assertEqual(json.loads(out)["hookSpecificOutput"]["permissionDecision"], "deny")

    def test_check_and_status(self):
        p = subprocess.run([PY, str(SCRIPT), "check", "curl -X POST https://app.example.com/api/items"], stdout=subprocess.PIPE, timeout=30)
        self.assertIn("deny", p.stdout.decode())
        p = subprocess.run([PY, str(SCRIPT), "status"], stdout=subprocess.PIPE, timeout=30)
        self.assertIn("app.example.com", p.stdout.decode())


class Installer(unittest.TestCase):
    def test_install_and_uninstall(self):
        settings = Path(STATE) / "settings.json"
        settings.write_text(json.dumps({"model": "keep-me", "hooks": {"PreToolUse": [
            {"matcher": "Bash", "hooks": [{"type": "command", "command": "echo other"}]}]}}), encoding="utf-8")
        env = dict(os.environ, CLAUDE_SETTINGS=str(settings))
        subprocess.run([PY, str(HERE / "install.py"), "--dry-run"], env=env, check=True, stdout=subprocess.PIPE)
        self.assertNotIn("action_guard", settings.read_text())
        subprocess.run([PY, str(HERE / "install.py")], env=env, check=True, stdout=subprocess.PIPE)
        data = json.loads(settings.read_text())
        self.assertEqual(data["model"], "keep-me")
        groups = data["hooks"]["PreToolUse"]
        self.assertEqual(len(groups), 2)
        self.assertIn("action_guard.py", groups[1]["hooks"][0]["command"])
        self.assertTrue(groups[1]["hooks"][0]["command"].endswith("|| exit 1"))
        subprocess.run([PY, str(HERE / "install.py")], env=env, check=True, stdout=subprocess.PIPE)   # idempotent
        self.assertEqual(len(json.loads(settings.read_text())["hooks"]["PreToolUse"]), 2)
        subprocess.run([PY, str(HERE / "install.py"), "--uninstall"], env=env, check=True, stdout=subprocess.PIPE)
        data = json.loads(settings.read_text())
        self.assertEqual(len(data["hooks"]["PreToolUse"]), 1)
        self.assertEqual(data["model"], "keep-me")


if __name__ == "__main__":
    unittest.main(verbosity=1)
```

## Setup

1. **Save the four files** into one folder and edit `action_guard.json` for your application.

2. **Run the tests** from that folder:

   ```
   python3 test_action_guard.py        # Windows: py test_action_guard.py
   ```

   Expect `Ran 22 tests ... OK`. They use a scratch directory and their own policy; nothing under `~/.claude` is touched.

3. **Dry-run the reader** against commands you expect the agent to type:

   ```
   python3 action_guard.py check "curl -X POST https://app.example.com/api/items -d '{}'" "curl https://app.example.com/api/items"
   ```

   Each line shows the requests found (method, URL, which client) and the decision. Do this for anything you are unsure the reader understands (a CLI of your own, an unusual client): a client it does not know is invisible to it.

4. **Install the hook**:

   ```
   python3 install.py --dry-run
   python3 install.py
   ```

   This writes one `PreToolUse` group into `~/.claude/settings.json` (after backing it up as `settings.json.bak-<timestamp>`) and leaves every other key alone. To install into a project instead, run it from the project root with `CLAUDE_SETTINGS=.claude/settings.json python3 install.py` (PowerShell: `$env:CLAUDE_SETTINGS=".claude\settings.json"; py install.py`). What it writes, on macOS/Linux:

   ```json
   {
     "hooks": {
       "PreToolUse": [
         {
           "matcher": "Bash|PowerShell|WebFetch|mcp__.*",
           "hooks": [
             {
               "type": "command",
               "command": "python3 '/home/you/claude-hooks/action-guard/action_guard.py' hook || exit 1",
               "timeout": 20,
               "statusMessage": "action-guard: checking the call..."
             }
           ]
         }
       ]
     }
   }
   ```

   On Windows the command is `py "C:/Users/you/claude-hooks/action-guard/action_guard.py" hook || exit 1` (forward slashes, which both Git Bash and PowerShell accept). You can also paste this by hand.

5. **Prove it in a real session.** Start `claude` (any project) and ask it, in these words:

   > Run exactly this shell command with the Bash tool and quote the tool result verbatim: `curl -s -X POST https://app.example.com/api/items -d '{}'`

   The Bash call is refused before it runs and the model reports something like `PreToolUse:Bash hook error: action-guard: refused. a POST request to app.example.com. Only GET, HEAD, OPTIONS requests are allowed ...`. Then `python3 action_guard.py status` shows the policy in force and the log line (`DENY method POST https://app.example.com/api/items`). A headless check works too: `claude -p --model haiku "<the same request>"`.

   If nothing happens: run the tests (step 2), check `jq .hooks ~/.claude/settings.json`, and open `/hooks` in the session once so it re-reads settings.

6. **Turn it off** with `python3 install.py --uninstall`, or for one process with the environment variable `ACTION_GUARD=off`.

State and log live in `~/.claude/action-guard/` (`guard.log`, rotated at 512 KB; `tabs.json`, the tab → host memory). Override the location with `ACTION_GUARD_STATE=<dir>`.

## Adapting the policy

Everything below is a change to `policy()` or to the JSON; the reader and the hook plumbing stay as they are.

- **Only in one project.** Install into the project's `.claude/settings.json` (step 4) rather than the user file; the hook then runs only for sessions started in that project.
- **Only for subagents, or stricter for them.** The hook input carries `agent_type` when a subagent made the call (`do_hook` already puts it in the log tag as `who=`). Pass it into `policy()` and, for instance, turn every `ask` into a `deny` when `who != "main"`, since a subagent cannot answer a prompt usefully.
- **Different rules for different parts of the app.** Add a list such as `"rules": [{"path": "/api/v1/orders", "methods": ["GET"], "decision": "ask"}, ...]` to the JSON and loop over it at the top of `policy()` before the general method check; `path_matches()` and `Request.method` are all you need.
- **Ask instead of deny, or the reverse.** Every branch ends in `_deny(...)` or `_ask(...)`; swap them. Keep in mind who sees what: a deny goes to the model, an ask goes to you.
- **Rewrite a call instead of refusing it.** A PreToolUse hook may also return `"updatedInput"` inside `hookSpecificOutput` with a modified `tool_input` (for example, appending `--dry-run` to a CLI, or adding a header). Not used here; it is the tool for "let it through, but safely".
- **The application's own MCP server.** The tool name Claude Code passes is `mcp__<server>__<tool>`; `mcp_rules` is matched on the server name. If the server's tools take a target inside their arguments (an id, a path), read `tool_input` in `findings_for()` and give the policy something to judge.
- **Browser reads you still want to see.** Add `screenshot` or `scroll` to `ask_browser_actions_on_protected_hosts`, or `get_page_text`, `read_page`, `find` to the deny list. They are silent by default because a guard that prompts on every read is switched off within a day.
- **Your own CLI that talks to the app** (`mycli deploy ...`). Add a small reader: in `scan()`, `elif name == "mycli": read_mycli(args, f, variables)` that appends `Request(...)` objects with a method of your choosing, or a plain `f.uncounted.append("mycli")` if it should always be judged by the strict path.

## What it cannot see

Say this to whoever relies on it.

- **The method inside code.** `python -c`, `node -e`, scripts, test suites, `npm run`, `make`, containers: the reader finds URLs written in them but not what is done with them, and finds nothing when the URL is built at run time. That is why requests from code to the application are refused by default (`on_unreadable_method`) and unreadable targets prompt (`on_unknown_target`). If the agent must run a test suite against the app, point the suite at a staging host that is not in `protected_hosts`.
- **What the server does with a request.** A GET that changes state on the server (`/api/delete?id=3` as a link) looks like a read. The guard sees the request as written; the application has to keep reads safe.
- **Redirects and second-order effects.** A request to another host that then calls your application, a webhook, a queued job.
- **Tabs the human navigated.** The tab → host memory is filled by the agent's own `navigate` calls; a tab you opened by hand is unknown until the agent navigates it. A page that redirects to another host after loading is also not seen.
- **Other clients and other machines.** Hooks run inside Claude Code only. Anything the agent can reach through a different tool (another MCP server, an SSH session to a host without the hook) is outside it — which is why the refusal text says not to route around it, and why the permission prompt remains the second line.
- **A determined adversary.** This is a guardrail against an agent's mistakes and drift, not a security boundary. Anything that must be impossible belongs in the application's own authorization (a read-only API token for the agent, a role without delete rights), with this hook in front of it to keep the session's behavior sensible and its reasons visible.

## If you also want request limits

The same skeleton runs a rate-limited variant: instead of `policy()` deciding on method and path, a `reserve()` step keeps a per-machine ledger (a JSON file behind the same `mkdir` lock) with, per site, the minimum spacing, an hourly and a daily cap, plus a machine-wide cap; the hook books the next slot and sleeps until it (up to a bound, otherwise refuses with the retry time), and a `PostToolUse` hook on `WebFetch` feeds status codes back so a 403/429/503 puts the site into back-off for a while. It also checks identifiers (a `User-Agent` must look like a real client; a personal name or address must never appear in a header, URL or body). That variant is about 1,600 lines and was replayed against 2,700 real tool calls before going live; it is not part of this package. Start with this one — the policy is easier to reason about, and the reader, the hook contract and the fail-open plumbing are identical, so pacing can be added to this skeleton later.

## Verified

On 30 September 2026: the 22 tests pass on macOS with Python 3.13.7 and 3.9.6 and on Windows 11 with Python 3.14.6; in a headless Claude Code session (2.1.283, model Haiku 4.5) started with the hook in a temporary settings file, the POST in step 5 was refused before it ran and the model quoted the reason; the log carried `DENY method POST`. The hook's own run time is 0.03 s per matching call.
