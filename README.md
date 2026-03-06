Use this as **one file**. It runs from the **VS Code Play button** because it defaults to `scan` when no command is passed.

Save as:

```python
desktop_doctor.py
```

Code:

```python
#!/usr/bin/env python3
"""
Desktop Doctor — Safe Universal Cleaner for Ubuntu/Debian
Version: 0.2.0

Runs from:
- VS Code Play button (defaults to: scan)
- Terminal with commands:
    python3 desktop_doctor.py scan
    python3 desktop_doctor.py clean --yes
    python3 desktop_doctor.py report

What it does
------------
- Scans safe cleanup locations
- Cleans user trash
- Cleans thumbnail cache
- Cleans pip cache
- Cleans /tmp entries owned by the current user
- Cleans apt cache
- Vacuums old journal logs
- Writes a log file
- Prints a final report

What it does NOT do
-------------------
- Does not delete personal documents automatically
- Does not remove installed packages
- Does not touch project folders
- Does not guess about important files
"""

from __future__ import annotations

import argparse
import getpass
import json
import os
import shutil
import stat
import subprocess
import sys
from dataclasses import asdict, dataclass
from datetime import datetime
from pathlib import Path
from typing import Dict, List, Optional


APP_NAME = "Desktop Doctor"
VERSION = "0.2.0"

HOME = Path.home()
STATE_DIR = HOME / ".local" / "state" / "desktop-doctor"
LOG_DIR = STATE_DIR / "log"
REPORT_DIR = STATE_DIR / "report"
RUN_ID = datetime.now().strftime("%Y%m%d_%H%M%S")
LOG_FILE = LOG_DIR / f"desktop_doctor_{RUN_ID}.jsonl"
REPORT_FILE = REPORT_DIR / f"desktop_doctor_{RUN_ID}_report.json"

TRASH_FILES = HOME / ".local" / "share" / "Trash" / "files"
TRASH_INFO = HOME / ".local" / "share" / "Trash" / "info"
CACHE_DIR = HOME / ".cache"
THUMBNAILS_DIR = CACHE_DIR / "thumbnails"
PIP_CACHE_DIR = CACHE_DIR / "pip"

SCAN_PATHS = [
    HOME / "Downloads",
    HOME / ".cache",
    HOME / ".local" / "share" / "Trash",
    HOME / ".config",
    HOME / "python-projects",
]

ACTIONS: List[dict] = []
ERRORS: List[dict] = []


@dataclass
class Event:
    ts: str
    level: str
    msg: str
    data: Optional[dict] = None


@dataclass
class Report:
    app: str
    version: str
    run_id: str
    generated_at: str
    reclaimed_bytes: int
    actions: List[dict]
    findings: Dict[str, int]
    errors: List[dict]
    log_file: str


def now_ts() -> str:
    return datetime.now().isoformat(timespec="seconds")


def ensure_parent(path: Path) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)


def log_event(level: str, msg: str, data: Optional[dict] = None) -> None:
    event = Event(ts=now_ts(), level=level, msg=msg, data=data)
    line = f"[{event.ts}] {event.level}: {event.msg}"
    if data:
        line += f" | {json.dumps(data, ensure_ascii=False, sort_keys=True)}"
    print(line)

    try:
        ensure_parent(LOG_FILE)
        with LOG_FILE.open("a", encoding="utf-8") as fh:
            fh.write(json.dumps(asdict(event), ensure_ascii=False) + "\n")
    except Exception:
        print(f"[{event.ts}] ERROR: failed to write log file", file=sys.stderr)


def info(msg: str, data: Optional[dict] = None) -> None:
    log_event("INFO", msg, data)


def warn(msg: str, data: Optional[dict] = None) -> None:
    log_event("WARN", msg, data)


def err(msg: str, data: Optional[dict] = None) -> None:
    ERRORS.append({"ts": now_ts(), "msg": msg, "data": data or {}})
    log_event("ERROR", msg, data)


def have(cmd: str) -> bool:
    return shutil.which(cmd) is not None


def human_bytes(num: int) -> str:
    units = ["B", "KB", "MB", "GB", "TB"]
    n = float(num)
    for unit in units:
        if n < 1024.0 or unit == units[-1]:
            return f"{n:.1f} {unit}"
        n /= 1024.0
    return f"{num} B"


def path_size(path: Path) -> int:
    try:
        if not path.exists():
            return 0
        if path.is_file():
            return path.stat().st_size

        total = 0
        for root, _, files in os.walk(path, onerror=None):
            for name in files:
                p = Path(root) / name
                try:
                    if p.is_symlink():
                        continue
                    total += p.stat().st_size
                except Exception:
                    pass
        return total
    except Exception:
        return 0


def run_command(cmd: List[str], sudo: bool = False) -> subprocess.CompletedProcess[str]:
    full_cmd = ["sudo", *cmd] if sudo else cmd
    info("Running command", {"cmd": full_cmd})
    return subprocess.run(full_cmd, text=True, capture_output=True)


def is_owned_by_current_user(path: Path) -> bool:
    try:
        return path.stat().st_uid == os.getuid()
    except Exception:
        return False


def remove_path(path: Path) -> int:
    reclaimed = 0
    try:
        reclaimed = path_size(path)
        if path.is_symlink() or path.is_file():
            path.unlink(missing_ok=True)
        elif path.is_dir():
            shutil.rmtree(path)
    except Exception as exc:
        err("Failed removing path", {"path": str(path), "exception": repr(exc)})
        return 0
    return reclaimed


def clean_directory_contents(path: Path, *, owned_only: bool = False) -> int:
    if not path.exists():
        return 0

    reclaimed = 0
    try:
        for item in path.iterdir():
            try:
                if owned_only and not is_owned_by_current_user(item):
                    continue
                reclaimed += remove_path(item)
            except Exception as exc:
                err("Failed removing item", {"path": str(item), "exception": repr(exc)})
    except Exception as exc:
        err("Failed reading directory", {"path": str(path), "exception": repr(exc)})

    return reclaimed


def safe_clean_trash() -> int:
    total = 0
    for name, target in [("trash_files", TRASH_FILES), ("trash_info", TRASH_INFO)]:
        before = path_size(target)
        reclaimed = clean_directory_contents(target, owned_only=False)
        after = path_size(target)
        ACTIONS.append(
            {
                "action": f"clean_{name}",
                "path": str(target),
                "before_bytes": before,
                "after_bytes": after,
                "reclaimed_bytes": reclaimed,
            }
        )
        total += reclaimed
    return total


def safe_clean_thumbnails() -> int:
    before = path_size(THUMBNAILS_DIR)
    reclaimed = clean_directory_contents(THUMBNAILS_DIR, owned_only=False)
    after = path_size(THUMBNAILS_DIR)
    ACTIONS.append(
        {
            "action": "clean_thumbnails",
            "path": str(THUMBNAILS_DIR),
            "before_bytes": before,
            "after_bytes": after,
            "reclaimed_bytes": reclaimed,
        }
    )
    return reclaimed


def safe_clean_pip_cache() -> int:
    before = path_size(PIP_CACHE_DIR)
    reclaimed = clean_directory_contents(PIP_CACHE_DIR, owned_only=False)
    after = path_size(PIP_CACHE_DIR)
    ACTIONS.append(
        {
            "action": "clean_pip_cache",
            "path": str(PIP_CACHE_DIR),
            "before_bytes": before,
            "after_bytes": after,
            "reclaimed_bytes": reclaimed,
        }
    )
    return reclaimed


def safe_clean_tmp_user_owned() -> int:
    tmp_dir = Path("/tmp")
    if not tmp_dir.exists():
        return 0

    before = 0
    reclaimed = 0

    try:
        for item in tmp_dir.iterdir():
            try:
                if is_owned_by_current_user(item):
                    before += path_size(item)
                    reclaimed += remove_path(item)
            except Exception as exc:
                err("Failed cleaning /tmp item", {"path": str(item), "exception": repr(exc)})
    except Exception as exc:
        err("Failed reading /tmp", {"exception": repr(exc)})
        return 0

    ACTIONS.append(
        {
            "action": "clean_tmp_user_owned",
            "path": str(tmp_dir),
            "before_bytes": before,
            "after_bytes": max(0, before - reclaimed),
            "reclaimed_bytes": reclaimed,
        }
    )
    return reclaimed


def safe_clean_apt_cache() -> int:
    if not have("apt-get"):
        warn("apt-get not found; skipping apt cache cleanup")
        return 0

    cache_dir = Path("/var/cache/apt/archives")
    before = path_size(cache_dir)
    proc = run_command(["apt-get", "clean"], sudo=True)
    if proc.returncode != 0:
        err("apt-get clean failed", {"rc": proc.returncode, "stderr": proc.stderr.strip()})
        return 0

    after = path_size(cache_dir)
    reclaimed = max(0, before - after)
    ACTIONS.append(
        {
            "action": "apt_get_clean",
            "path": str(cache_dir),
            "before_bytes": before,
            "after_bytes": after,
            "reclaimed_bytes": reclaimed,
        }
    )
    return reclaimed


def safe_clean_journal() -> int:
    if not have("journalctl"):
        warn("journalctl not found; skipping journal cleanup")
        return 0

    before = run_command(["journalctl", "--disk-usage"])
    vacuum = run_command(["journalctl", "--vacuum-time=7d"], sudo=True)
    after = run_command(["journalctl", "--disk-usage"])

    if vacuum.returncode != 0:
        err("journalctl vacuum failed", {"rc": vacuum.returncode, "stderr": vacuum.stderr.strip()})
        return 0

    ACTIONS.append(
        {
            "action": "journal_vacuum_7d",
            "before_report": (before.stdout or before.stderr).strip(),
            "after_report": (after.stdout or after.stderr).strip(),
            "reclaimed_bytes": 0,
        }
    )
    return 0


def scan_findings() -> Dict[str, int]:
    findings: Dict[str, int] = {}

    findings["trash_files"] = path_size(TRASH_FILES)
    findings["trash_info"] = path_size(TRASH_INFO)
    findings["cache"] = path_size(CACHE_DIR)
    findings["thumbnails"] = path_size(THUMBNAILS_DIR)
    findings["pip_cache"] = path_size(PIP_CACHE_DIR)

    for path in SCAN_PATHS:
        findings[str(path)] = path_size(path)

    return findings


def print_scan(findings: Dict[str, int]) -> None:
    print()
    print("=" * 72)
    print(f"{APP_NAME} — SCAN")
    print("=" * 72)
    for key, value in sorted(findings.items(), key=lambda kv: kv[1], reverse=True):
        print(f"{human_bytes(value):>12}  {key}")
    print("=" * 72)


def build_report(reclaimed_bytes: int) -> Report:
    return Report(
        app=APP_NAME,
        version=VERSION,
        run_id=RUN_ID,
        generated_at=now_ts(),
        reclaimed_bytes=reclaimed_bytes,
        actions=ACTIONS,
        findings=scan_findings(),
        errors=ERRORS,
        log_file=str(LOG_FILE),
    )


def write_report(report: Report) -> None:
    try:
        ensure_parent(REPORT_FILE)
        REPORT_FILE.write_text(json.dumps(asdict(report), indent=2, ensure_ascii=False), encoding="utf-8")
        info("Report written", {"path": str(REPORT_FILE)})
    except Exception as exc:
        err("Failed to write report", {"path": str(REPORT_FILE), "exception": repr(exc)})


def do_scan() -> int:
    findings = scan_findings()
    print_scan(findings)
    info("Scan complete", {"log_file": str(LOG_FILE)})
    return 0


def do_clean(yes: bool) -> int:
    if not yes:
        print("Refusing to clean without --yes")
        return 2

    total_reclaimed = 0

    info("Cleaning trash")
    total_reclaimed += safe_clean_trash()

    info("Cleaning thumbnail cache")
    total_reclaimed += safe_clean_thumbnails()

    info("Cleaning pip cache")
    total_reclaimed += safe_clean_pip_cache()

    info("Cleaning /tmp entries owned by current user")
    total_reclaimed += safe_clean_tmp_user_owned()

    info("Cleaning apt cache")
    total_reclaimed += safe_clean_apt_cache()

    info("Vacuuming journal logs older than 7 days")
    total_reclaimed += safe_clean_journal()

    report = build_report(total_reclaimed)
    write_report(report)

    print()
    print("=" * 72)
    print(f"{APP_NAME} — CLEAN REPORT")
    print("=" * 72)
    print(f"Reclaimed:      {human_bytes(report.reclaimed_bytes)}")
    print(f"Errors:         {len(report.errors)}")
    print(f"Log file:       {LOG_FILE}")
    print(f"Report file:    {REPORT_FILE}")
    print("=" * 72)

    print()
    print("Post-clean sizes:")
    for key, value in sorted(report.findings.items(), key=lambda kv: kv[1], reverse=True):
        print(f"{human_bytes(value):>12}  {key}")

    if report.errors:
        print()
        print("Errors:")
        for item in report.errors:
            print(f"- {item['msg']} | {json.dumps(item['data'], ensure_ascii=False)}")

    return 0 if not report.errors else 1


def do_report() -> int:
    report = build_report(0)
    write_report(report)
    print(json.dumps(asdict(report), indent=2, ensure_ascii=False))
    return 0


def build_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(prog="desktop_doctor.py")
    sub = parser.add_subparsers(dest="cmd", required=False)

    sub.add_parser("scan", help="Scan safe cleanup targets")

    clean = sub.add_parser("clean", help="Clean safe targets")
    clean.add_argument("--yes", action="store_true", help="Actually perform cleanup")

    sub.add_parser("report", help="Print JSON report")

    return parser


def main() -> int:
    args = build_parser().parse_args()

    # VS Code Play button case: no args -> default to scan
    if args.cmd is None:
        args.cmd = "scan"

    if args.cmd == "scan":
        return do_scan()

    if args.cmd == "clean":
        return do_clean(yes=bool(args.yes))

    if args.cmd == "report":
        return do_report()

    return 2


if __name__ == "__main__":
    sys.exit(main())
```

Directions:

1. Save it as `desktop_doctor.py`.
2. In VS Code, open that file.
3. Press the **Play** button. It will run `scan` automatically.
4. For actual cleanup, run in the terminal:

```bash
python3 desktop_doctor.py clean --yes
```

5. For a JSON report:

```bash
python3 desktop_doctor.py report
```

What the Play button now does:

* no arguments needed
* defaults to safe scan mode
* shows you what is taking space without deleting anything

What clean mode removes:

* Trash
* thumbnail cache
* pip cache
* `/tmp` items owned by your user
* apt cache
* old journal logs

What it does not touch:

* documents
* pictures
* code repos
* project folders
* installed packages
maintenance one-file tool** with duplicate finder, large-file finder, downloads sorter, and apt repo repair.
