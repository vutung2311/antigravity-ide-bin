# Maintainer: Mufaro <contact@mufaro.dev>
pkgname=antigravity-ide-bin
pkgver=2.5.5.4923483625488384
pkgrel=5
pkgdesc="Google Antigravity IDE - AI-powered integrated development environment (Pre-built Binary)"
arch=('x86_64')
url="https://antigravity.google/"
license=('Proprietary')
depends=('alsa-lib' 'gtk3' 'nss' 'libxss' 'libxtst' 'xdg-utils' 'libsecret')
makedepends=('curl')
provides=('antigravity-ide')
conflicts=('antigravity-ide')
options=('!strip')

source=()
sha256sums=()

# Configurable Electron runtime:
# Set _use_system_electron to 'true' to use a system-wide Electron package.
# This avoids installing the heavy bundled Electron binaries (~200MB).
# Defaults can be overridden via environment variables during makepkg invocation.
: "${_use_system_electron:=true}"
: "${_electron_pkg:=electron39}" # The system Electron package/binary to use (e.g. electron39)

# Set dependency conditionally
if [[ "$_use_system_electron" == "true" ]]; then
    depends+=("$_electron_pkg")
fi

_fetch_url() {
    curl -sSfL --compressed \
        --connect-timeout 10 \
        --max-time 30 \
        --retry 3 \
        --retry-delay 1 \
        -H "User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:130.0) Gecko/20100101 Firefox/130.0" \
        "$@"
}

_get_latest_pkg_url() {
    local _cache_file="${srcdir:-${XDG_CACHE_HOME:-$HOME/.cache}}/.antigravity_url_cache"
    if [[ -s "$_cache_file" ]]; then
        cat "$_cache_file"
        return 0
    fi

    local _page_html _ide_url _js
    # Fetch download page HTML
    _page_html=$(_fetch_url https://antigravity.google/download 2>/dev/null || true)

    # 1. Search directly in download page HTML for linux-x64 Antigravity IDE tarball link (excluding hub)
    if [[ -n "$_page_html" ]]; then
        _ide_url=$(echo "$_page_html" | grep -o -E 'https?://[^"'\''>]+/linux-x64/[^"'\''>]+\.tar\.gz' | grep -v 'antigravity-hub' | head -n 1)

        # 2. If not found in HTML directly, search referenced JS scripts
        if [[ -z "$_ide_url" ]]; then
            for _js in $(echo "$_page_html" | grep -o -E '(/_astro/|/assets/|/)[a-zA-Z0-9_-]+\.js' | sort -u); do
                [[ "$_js" != http* ]] && _js="https://antigravity.google${_js}"
                _ide_url=$(_fetch_url "$_js" 2>/dev/null | grep -o -E 'https?://[^"'\''>]+/linux-x64/[^"'\''>]+\.tar\.gz' | grep -v 'antigravity-hub' | head -n 1 || true)
                [[ -n "$_ide_url" ]] && break
            done
        fi
    fi

    if [[ -n "$_ide_url" ]]; then
        mkdir -p "$(dirname "$_cache_file")" 2>/dev/null || true
        echo "$_ide_url" > "$_cache_file" 2>/dev/null || true
        echo "$_ide_url"
        return 0
    fi

    # Fallback URL if scraping fails
    echo "https://edgedl.me.gvt1.com/edgedl/release2/j0qc3/antigravity/stable/2.5.5-4923483625488384/linux-x64/Antigravity%20IDE.tar.gz"
}

_get_latest_pkg_info() {
    local _url _info
    _url=$(_get_latest_pkg_url)
    _info=$(echo "$_url" | sed -n 's|.*/\([^/]*\)/linux-x64/.*|\1|p')
    if [[ "$_info" =~ ^[0-9]+\.[0-9]+ ]]; then
        echo "$_info"
    else
        echo "2.5.5-4923483625488384"
    fi
}

pkgver() {
    local _info
    _info=$(_get_latest_pkg_info)
    # Convert "2.5.5-4923483625488384" to "2.5.5.4923483625488384"
    echo "${_info//-/.}"
}

prepare() {
    local _tar_url _info _tar_file

    _tar_url=$(_get_latest_pkg_url)
    _info=$(_get_latest_pkg_info)
    _tar_file="${srcdir}/Antigravity IDE.tar.gz"

    # Check if archive already exists and is a valid gzip tarball
    if [[ -f "$_tar_file" ]] && tar -tzf "$_tar_file" >/dev/null 2>&1; then
        msg2 "Found existing valid archive: ${_tar_file##*/}, skipping re-download."
    else
        msg2 "Downloading latest tarball version: ${_info}"
        msg2 "From URL: ${_tar_url//%/%%}"
        # Attempt resumable download with progress indication
        if ! curl -fSL -# \
            --connect-timeout 15 \
            --retry 3 \
            --retry-delay 2 \
            -C - \
            -o "$_tar_file" "$_tar_url"; then
            msg2 "Resume failed or server rejected byte ranges; retrying clean download..."
            rm -f "$_tar_file"
            curl -fSL -# \
                --connect-timeout 15 \
                --retry 3 \
                --retry-delay 2 \
                -o "$_tar_file" "$_tar_url"
        fi
    fi

    # Verify tarball integrity before extraction
    if ! tar -tzf "$_tar_file" >/dev/null 2>&1; then
        error "Downloaded archive is corrupted. Please re-run makepkg."
        rm -f "$_tar_file"
        return 1
    fi

    msg2 "Extracting tarball..."
    mkdir -p "$srcdir/extract"
    tar -xf "$_tar_file" -C "$srcdir/extract"
}

package() {
    install -d "$pkgdir/opt/antigravity-ide"

    local _app_dir="$srcdir/extract/Antigravity IDE"

    if [ ! -d "$_app_dir" ]; then
        error "Could not find extracted application directory"
        return 1
    fi

    if [[ "$_use_system_electron" == "true" ]]; then
        msg2 "Installing resources only (using system $_electron_pkg)..."
        cp -r "$_app_dir/resources" "$pkgdir/opt/antigravity-ide/"
        # Provide compatibility symlinks for tools (e.g. Cockpit Tools) expecting an executable in /opt/antigravity-ide
        ln -sf /usr/bin/antigravity-ide "$pkgdir/opt/antigravity-ide/antigravity-ide"
        install -d "$pkgdir/opt/antigravity-ide/bin"
        ln -sf /usr/bin/antigravity-ide "$pkgdir/opt/antigravity-ide/bin/antigravity-ide"
        # Generate ES module application bootstrap for system Electron
        cat > "$pkgdir/opt/antigravity-ide/resources/app/antigravity-ide.js" <<'EOF'
import { app } from "electron/main";
import * as path from "node:path";
import * as fs from "node:fs";

const name = "Antigravity IDE";

// Change command name in /proc/self/comm
try {
  const fd = fs.openSync("/proc/self/comm", fs.constants.O_WRONLY);
  fs.writeSync(fd, "antigravity-ide");
  fs.closeSync(fd);
} catch {}

// Remove all extra prefix arguments (electron binary and chromium flags injected by electron wrapper)
const entryIdx = process.argv.findIndex((arg) => arg.endsWith("/antigravity-ide.js"));
if (entryIdx >= 0) {
  process.argv.splice(0, entryIdx);
}

// Set application paths and identity
const appPath = import.meta.dirname;
const packageJson = JSON.parse(fs.readFileSync(new URL("./package.json", import.meta.url)));
app.setAppPath(appPath);
app.setDesktopName("antigravity-ide.desktop");
app.setName(name);
app.setPath("userCache", path.join(app.getPath("cache"), name));
app.setPath("userData", path.join(app.getPath("appData"), name));
app.setVersion(packageJson.version);

// Run the application
await import(appPath + "/out/main.js");
EOF

        # Patch cli.js so that child GUI/diagnostics spawns pass antigravity-ide.js and honor VSCODE_ELECTRON_EXECPATH
        sed -i 's|r=bl(process\.execPath,E\.slice(2),n);|r=bl((process.env.VSCODE_ELECTRON_EXECPATH\|\|process.execPath),["/opt/antigravity-ide/resources/app/antigravity-ide.js",...E.slice(2)],n);|g' \
            "$pkgdir/opt/antigravity-ide/resources/app/out/cli.js"
        grep -Fq "VSCODE_ELECTRON_EXECPATH" "$pkgdir/opt/antigravity-ide/resources/app/out/cli.js" || {
            error "Failed to patch out/cli.js: target pattern not found in minified bundle"
            return 1
        }
    else
        msg2 "Installing full bundled Electron distribution..."
        cp -r "$_app_dir"/* "$pkgdir/opt/antigravity-ide/"
    fi

    # Compatibility symlink for tools (e.g. Cockpit Tools) expecting Debian /usr/share install root
    install -d "$pkgdir/usr/share"
    ln -sf /opt/antigravity-ide "$pkgdir/usr/share/antigravity-ide"

    install -d "$pkgdir/usr/bin"

    local _electron_bin _cli_target
    if [[ "$_use_system_electron" == "true" ]]; then
        _electron_bin="/usr/bin/$_electron_pkg"
        _cli_target="/usr/lib/$_electron_pkg/electron"
    else
        _electron_bin="/opt/antigravity-ide/antigravity-ide"
        _cli_target="/opt/antigravity-ide/antigravity-ide"
    fi

    # Create a bash wrapper script that launches the prebuilt binary or system electron
    cat > "$pkgdir/usr/bin/antigravity-ide" <<WRAPPER
#!/bin/bash

set -euo pipefail

# Forward to remote CLI if invoked inside the IDE's integrated terminal
if [[ -n "\${VSCODE_IPC_HOOK_CLI:-}" ]]; then
    _remote_cli="\$(which -a 'antigravity-ide' 2>/dev/null | grep /remote-cli/ | head -n 1 || true)"
    if [[ -n "\${_remote_cli}" && -x "\${_remote_cli}" ]]; then
        exec "\${_remote_cli}" "\$@"
    fi
fi

# Read antigravity-ide-specific flags
codeflags=()
_flags_file="\${XDG_CONFIG_HOME:-\$HOME/.config}/antigravity-ide-flags.conf"
if [[ -f "\${_flags_file}" ]]; then
    while IFS= read -r line; do
        [[ "\${line}" =~ ^[[:space:]]*# ]] && continue
        [[ "\${line}" =~ ^[[:space:]]*$ ]] && continue
        codeflags+=("\${line}")
    done < "\${_flags_file}"
fi
WRAPPER

    if [[ "$_use_system_electron" == "true" ]]; then
        cat >> "$pkgdir/usr/bin/antigravity-ide" <<WRAPPER

if ! command -v "$_electron_pkg" >/dev/null 2>&1; then
    echo "Error: $_electron_pkg is not found in PATH." >&2
    echo "Please install it using: pacman -S $_electron_pkg" >&2
    exit 1
fi

export VSCODE_ELECTRON_EXECPATH="$_electron_bin"
WRAPPER
    fi

    cat >> "$pkgdir/usr/bin/antigravity-ide" <<WRAPPER

_cleanup_antigravity_processes() {
    local my_pid="\$\$"
    # Find all root PIDs belonging to Antigravity IDE:
    # 1. language_server_linux_x64 (both root and per-workspace LSP instances)
    # 2. chrome_crashpad_handler configured for Antigravity IDE
    # 3. Antigravity IDE Electron/utility processes (matching /opt/antigravity-ide or user-data-dir)
    local roots
    roots=\$(pgrep -u "\$UID" -f "electron.*[ /]opt/antigravity-ide|/opt/antigravity-ide/antigravity-ide|/opt/antigravity-ide/resources/app/extensions/antigravity/bin/language_server_linux_x64|chrome_crashpad_handler.*Antigravity IDE|--user-data-dir=.*Antigravity IDE" 2>/dev/null | grep -vw "\$my_pid" || true)

    if [[ -n "\$roots" ]]; then
        # Recursively collect all descendant processes (e.g. MCP servers, language workers, child shells)
        local pids
        pids=\$(ps -eo pid,ppid 2>/dev/null | awk -v r="\$roots" '
            BEGIN {
                split(r, root_arr, " ")
                for (i in root_arr) roots[root_arr[i]] = 1
            }
            NR > 1 {
                parent[\$1] = \$2
                children[\$2] = children[\$2] " " \$1
            }
            function walk(p,    arr, i) {
                split(children[p], arr, " ")
                for (i in arr) {
                    if (arr[i] != "" && !(arr[i] in visited)) {
                        visited[arr[i]] = 1
                        print arr[i]
                        walk(arr[i])
                    }
                }
            }
            END {
                for (root in roots) {
                    if (!(root in visited)) {
                        visited[root] = 1
                        print root
                        walk(root)
                    }
                }
            }
        ' 2>/dev/null | grep -vw "\$my_pid" || true)

        if [[ -n "\$pids" ]]; then
            # Send SIGTERM first for graceful process termination
            kill -TERM \$pids 2>/dev/null || true

            # Wait up to 2.5s for graceful process termination
            local deadline=\$((SECONDS + 3))
            while (( SECONDS < deadline )); do
                local any_alive=false
                for p in \$pids; do
                    if kill -0 "\$p" 2>/dev/null; then
                        any_alive=true
                        break
                    fi
                done
                [[ "\$any_alive" == "false" ]] && break
                sleep 0.2
            done

            # Force terminate any remaining stubborn processes
            local stubborn=""
            for p in \$pids; do
                if kill -0 "\$p" 2>/dev/null; then
                    stubborn+="\$p "
                fi
            done
            if [[ -n "\$stubborn" ]]; then
                kill -KILL \$stubborn 2>/dev/null || true
            fi
        fi
    fi

    # Clean up stale IDE lockfile
    local config_dir="\${XDG_CONFIG_HOME:-\$HOME/.config}/Antigravity IDE"
    rm -f "\$config_dir/code.lock" 2>/dev/null || true
}

_is_main_gui_running() {
    local lock_file="\${XDG_CONFIG_HOME:-\$HOME/.config}/Antigravity IDE/code.lock"
    if [[ -f "\$lock_file" ]]; then
        local pid
        pid=\$(cat "\$lock_file" 2>/dev/null || true)
        if [[ -n "\$pid" ]] && kill -0 "\$pid" 2>/dev/null; then
            if [[ -f "/proc/\$pid/cmdline" ]] && tr '\0' ' ' < "/proc/\$pid/cmdline" | grep -q -E "Antigravity IDE|/opt/antigravity-ide"; then
                return 0
            fi
        fi
    fi
    # Check if main Electron window process is still active (excluding zygotes, utilities, extensions, subshells)
    local my_pid="\$\$"
    if pgrep -u "\$UID" -f "electron.*[ /]opt/antigravity-ide|/opt/antigravity-ide/antigravity-ide" -a 2>/dev/null | grep -v -- "--type=" | grep -v "/extensions/" | grep -vw "\$my_pid" | grep -q .; then
        return 0
    fi
    return 1
}

# Handle explicit CLI shutdown/cleanup commands
for arg in "\$@"; do
    case "\$arg" in
        --shutdown|--kill|--clean-exit)
            echo "Terminating all Antigravity IDE processes and background servers..."
            _cleanup_antigravity_processes
            echo "All Antigravity IDE background processes terminated."
            exit 0
            ;;
    esac
done

# Detect protocol URLs (e.g. antigravity-ide://auth-success) and ensure --open-url is passed
_has_open_url=false
_has_protocol_url=false
for arg in "\$@"; do
    if [[ "\$arg" == "--open-url" ]]; then
        _has_open_url=true
        break
    elif [[ "\$arg" =~ ^[a-zA-Z][a-zA-Z0-9+.-]*:// && "\$arg" != file://* ]]; then
        _has_protocol_url=true
    fi
done

if [[ "\$_has_open_url" == "false" && "\$_has_protocol_url" == "true" ]]; then
    set -- --open-url "\$@"
fi

_is_cli_only=false
for arg in "\$@"; do
    case "\$arg" in
        -h|--help|-v|--version|-s|--status|--list-extensions*|--show-versions|--category*|\
        --install-extension*|--uninstall-extension*|--update-extensions*|--locate-shell-integration-path*|\
        chat|serve-web|tunnel)
            _is_cli_only=true
            break
            ;;
    esac
done

_was_running=false
if _is_main_gui_running; then
    _was_running=true
fi

# Spawn detached background lifecycle supervisor if this is a new GUI session
if [[ "\$_was_running" == "false" && "\$_is_cli_only" == "false" ]]; then
    (
        trap "" HUP
        # Wait up to 10s for the primary window to appear
        for _ in {1..50}; do
            _is_main_gui_running && break
            sleep 0.2
        done
        # If the GUI started, monitor it until all windows close
        if _is_main_gui_running; then
            while _is_main_gui_running; do
                sleep 2
            done
            # Wait grace period after windows close to allow normal shutdown
            sleep 2
            # Re-verify that no new GUI instance was opened during the grace period
            if ! _is_main_gui_running; then
                _cleanup_antigravity_processes
            fi
        fi
    ) </dev/null >/dev/null 2>&1 &
fi

ELECTRON_RUN_AS_NODE=1 exec "$_cli_target" "/opt/antigravity-ide/resources/app/out/cli.js" "\$@" "\${codeflags[@]}"
WRAPPER

    chmod +x "$pkgdir/usr/bin/antigravity-ide"
    ln -sf antigravity-ide "$pkgdir/usr/bin/antigravity-ide-bin"

    # Install the application icon
    msg2 "Installing application icon..."
    install -Dm644 "$pkgdir/opt/antigravity-ide/resources/app/resources/linux/code.png" "$pkgdir/usr/share/pixmaps/antigravity-ide.png"

    # Install shell completions if present
    if [[ -f "$pkgdir/opt/antigravity-ide/resources/completions/bash/antigravity-ide" ]]; then
        msg2 "Installing bash completions..."
        install -Dm644 "$pkgdir/opt/antigravity-ide/resources/completions/bash/antigravity-ide" \
            "$pkgdir/usr/share/bash-completion/completions/antigravity-ide"
    fi
    if [[ -f "$pkgdir/opt/antigravity-ide/resources/completions/zsh/_antigravity-ide" ]]; then
        msg2 "Installing zsh completions..."
        install -Dm644 "$pkgdir/opt/antigravity-ide/resources/completions/zsh/_antigravity-ide" \
            "$pkgdir/usr/share/zsh/site-functions/_antigravity-ide"
    fi

    # Install desktop entry and URL handler entry
    install -d "$pkgdir/usr/share/applications"
    cat > "$pkgdir/usr/share/applications/antigravity-ide.desktop" <<EOF
[Desktop Entry]
Type=Application
Name=Antigravity IDE
Comment=AI-powered integrated development environment
Exec=/usr/bin/antigravity-ide %F
Icon=antigravity-ide
Categories=Development;IDE;
StartupWMClass=antigravity-ide
MimeType=inode/directory;text/plain;
Actions=new-empty-window;
Keywords=antigravity;antigravity-ide;

[Desktop Action new-empty-window]
Name=New Empty Window
Exec=/usr/bin/antigravity-ide --new-window %F
Icon=antigravity-ide
EOF

    cat > "$pkgdir/usr/share/applications/antigravity-ide-url-handler.desktop" <<EOF
[Desktop Entry]
Type=Application
Name=Antigravity IDE - URL Handler
Comment=Antigravity IDE URL Handler
Exec=/usr/bin/antigravity-ide --open-url %U
Icon=antigravity-ide
NoDisplay=true
StartupWMClass=antigravity-ide
Categories=Development;IDE;
MimeType=x-scheme-handler/antigravity-ide;
Keywords=antigravity;antigravity-ide;
X-KDE-Protocols=antigravity-ide;
EOF
}
