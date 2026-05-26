# Python Knowledge

## Version Compatibility
- Python 3.8 does NOT support `list[dict]`, `dict[str, Any]`, etc. as type hints
  - Use `from typing import List, Dict, Optional` instead
  - Or remove type hints entirely
- `match/case` requires Python 3.10+
- f-string `=` debugging (e.g., `f"{x=}"`) requires Python 3.8+

## Windows Gotchas
- Default console encoding is CP1252, not UTF-8
- Set `PYTHONIOENCODING=utf-8` for scripts that output unicode/emoji
- `subprocess.run` with `text=True` uses system encoding — use `capture_output=True` without `text=True` and decode manually with `errors="replace"` for robustness
- `python3` alias doesn't exist on Windows — use full path or `py`
