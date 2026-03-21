# Install Script Review

**Script:** `https://raw.githubusercontent.com/zubair-trabzada/geo-seo-claude/main/install.sh`
**Reviewed:** 2026-03-21

---

## Summary

The script installs the GEO-SEO Claude Code skill by cloning a GitHub repository and copying files into `~/.claude/`. Overall structure is reasonable, but there are security and robustness issues worth addressing.

---

## Security Issues

### 1. Curl-pipe execution model (high severity)
The script is designed to be run via `curl | bash`, which is a well-known security anti-pattern. There is no integrity verification (checksums, GPG signatures) before execution. A compromised CDN, DNS hijack, or repository compromise would silently execute malicious code.

**Recommendation:** Provide a checksum or GPG signature for the script, and instruct users to verify it before running. Alternatively, publish the tool as a versioned package.

### 2. No repository integrity check (medium severity)
The repository is cloned with `git clone --depth 1` but no commit hash or tag is pinned. Future pushes to `main` would affect new installations without any user notification.

```bash
# Current — always uses latest commit on main
git clone --depth 1 "$REPO_URL" "$TEMP_DIR/repo"

# Better — pin to a specific release tag or commit
git clone --depth 1 --branch v1.0.0 "$REPO_URL" "$TEMP_DIR/repo"
```

### 3. Unverified pip install (medium severity)
`requirements.txt` is installed without inspecting its contents. If the repository is compromised, arbitrary packages could be installed silently.

### 4. Making files executable without content verification (low severity)
```bash
chmod +x "$INSTALL_DIR/scripts/"*.py 2>/dev/null || true
chmod +x "$INSTALL_DIR/hooks/"* 2>/dev/null || true
```
Files are made executable without verifying their contents match expectations.

---

## Robustness Issues

### 5. Unsafe glob expansion with `set -euo pipefail`
Several copy commands use unquoted globs:
```bash
cp -r "$SOURCE_DIR/geo/"* "$INSTALL_DIR/"
cp -r "$SOURCE_DIR/scripts/"* "$INSTALL_DIR/scripts/"
```
If the source directory is empty, the glob expands to a literal `*`, causing `cp` to fail (which under `set -e` exits the script). The hooks section partially handles this with `ls -A` check, but the `geo/` and `scripts/` copies do not.

**Recommendation:** Check directory contents before copying:
```bash
if [ "$(ls -A "$SOURCE_DIR/geo/" 2>/dev/null)" ]; then
    cp -r "$SOURCE_DIR/geo/"* "$INSTALL_DIR/"
fi
```

### 6. Parsing `ls` output for verification
```bash
[ "$(ls "$AGENTS_DIR"/geo-*.md 2>/dev/null | wc -l)" -gt 0 ]
```
Parsing `ls` is fragile (filenames with newlines break it). Use shell glob count instead:
```bash
shopt -s nullglob
agent_files=("$AGENTS_DIR"/geo-*.md)
[ "${#agent_files[@]}" -gt 0 ]
```

### 7. Hardcoded verification path
```bash
[ -d "$SKILLS_DIR/geo-audit" ] && print_success "Sub-skills directory" || ...
```
The verification step hardcodes `geo-audit` as the expected sub-skills directory name. This will silently fail if the repository structure changes.

### 8. BASH_SOURCE edge case handling is complex
The `BASH_SOURCE[0]` check to distinguish local from remote install has multiple edge cases and a comment acknowledging the complexity. Consider using a dedicated `--local` flag instead.

### 9. Silent pip failure
```bash
$PYTHON_CMD -m pip install -r "$SOURCE_DIR/requirements.txt" --quiet 2>/dev/null && {
    print_success "Python dependencies installed"
} || {
    print_warning "Some Python dependencies failed to install."
```
`2>/dev/null` suppresses all pip error output, making it hard to diagnose dependency failures.

---

## Style / Minor Issues

### 10. Python version check is fragile
The `python` (non-`python3`) fallback version detection uses `grep -oE` and string arithmetic that could fail on unusual Python version strings (e.g., `3.10.1rc1`). The current code only checks for Python 3.8+, but the minor version comparison using `-ge` on string-split values is brittle.

### 11. No rollback on partial failure
If installation succeeds partway but then fails (e.g., during agent file copy), the partially installed state is left in `~/.claude/`. Adding a rollback step or installing to a temp location first and doing an atomic move would be cleaner.

---

## Positive Aspects

- `set -euo pipefail` at the top — strict error handling is good.
- `mktemp -d` for temp directory — no predictable path, prevents race conditions.
- `trap cleanup EXIT` — temp directory is always removed.
- `--depth 1` for the clone — faster and uses less bandwidth.
- Interactive detection (`[ ! -t 0 ]`) allows graceful non-interactive execution.
- Optional Playwright install is skipped in non-interactive mode.
- Verification step at the end confirms critical files are present.
- Clear, readable output with color coding and status symbols.

---

## Recommendations Summary

| Priority | Issue | Action |
|----------|-------|--------|
| High | Curl-pipe without integrity check | Add checksum/GPG verification or publish as versioned package |
| Medium | Unpinned repository clone | Pin to tagged release |
| Medium | Silent pip install errors | Remove `2>/dev/null` or capture and display on failure |
| Low | Unsafe glob expansion | Guard copies with directory non-empty checks |
| Low | `ls` output parsing | Use shell arrays with `nullglob` |
| Low | Hardcoded verification path | Derive dynamically from installed structure |
