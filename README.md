# mpv AI chain — browser extension

Send a page, a link or the video under the cursor from a Chromium-family
browser (Helium, Chromium, Chrome, Brave, Vivaldi, Edge) to the local
**mpv + NVIDIA VFX** chain: NVDEC decode, VSR super-resolution, TrueHDR,
DLSS 5 Neural Render and VFG frame generation.

The browser does no video processing. It hands the URL to a native messaging
host, which starts `mpv-ai` (or `mpv-nr`, or plain `mpv`) detached, and the
player's filter chain does everything else. YouTube and other `yt-dlp` sites
work because mpv's own `ytdl_hook` drives `yt-dlp`.

```
browser tab ──Alt+Shift+M──▶ service worker ──native messaging──▶ mpv_ai_host.py
                                                                    │ execve(2)
                                                                    ▼
                                                        mpv-ai  (mpv-dovi-vfx-git)
                                          hwup → vsr → truehdr → dlssnr → vfg
```

## Requirements

| Thing | Why |
| --- | --- |
| [`mpv-nvidia-ai-chain`](https://github.com/HyperRamzey/mpv-nvidia-ai-chain) | provides `mpv`, the `mpv-ai` / `mpv-nr` wrappers and `~/.config/mpv/ai.conf`. Without them the host refuses with a named reason. |
| `python3` | the host is stdlib-only Python, no daemon, no dependencies |
| `yt-dlp` on `PATH` | only needed for site URLs (YouTube etc.) |
| Chromium ≥ 102 | Manifest V3, `nativeMessaging` |

`mpv-ai`, `mpv-nr` and `yt-dlp` usually live in `~/.local/bin`; the host adds
that directory to its search path because a browser-launched process does not
inherit your shell's `PATH`.

## Install

```sh
# 1. register the native host (copies it to ~/.local/share/mpv-ai-extension)
./native/install_native_host.sh          # only browsers whose profile dir exists
./native/install_native_host.sh --all    # register for every known browser
./native/install_native_host.sh --uninstall

# 2. load the extension
#    chrome://extensions -> Developer mode -> Load unpacked -> select extension/
```

The installer derives the extension ID from the public key embedded in
`extension/manifest.json`, writes it into the host manifest's `allowed_origins`,
and then round-trips a `ping` through the *installed* host to prove the whole
path works. With that key the ID is stable for everyone and does not depend on
where the checkout lives:

```
laaljncfmpifjkegkcolkmifmbdpkece
```

Re-run the installer after editing `native/mpv_ai_host.py` — it copies the file,
so the installed copy is what runs.

## Use

| Action | How |
| --- | --- |
| Send the current tab | toolbar icon, or <kbd>Alt</kbd>+<kbd>Shift</kbd>+<kbd>M</kbd> |
| Send a specific thing | right-click a page / link / `<video>` → **Send to mpv** |
| Choose the chain per launch | the right-click menu overrides the profile |
| Defaults | right-click the icon → Options: profile, fullscreen, extra mpv args, `--ytdl-format` |

Feedback is the toolbar badge: `OK` after a successful spawn, `!` on failure,
with the reason in the service-worker console and in
`$XDG_STATE_HOME/mpv-ai-extension/launch.log`.

## Three things worth knowing

- **No shell in the path.** The host accepts `http(s)` URLs only and can only
  start `mpv-ai`, `mpv-nr` or `mpv`, chosen by a profile name and resolved on
  `PATH`; the argument vector goes straight to `execve(2)`.
- **Playback is detached.** Closing the tab or the browser does not stop mpv.
- **`mpv-ai` needs a GPU video output.** `--vo=null` with the chain segfaults
  (the filters need the VO's CUDA context); plain `mpv --vo=null` is fine.

## Layout

| Path | What |
| --- | --- |
| `extension/manifest.json` | MV3 manifest; the embedded `key` pins the extension ID |
| `extension/background.js` | service worker: context menus, keyboard command, badge |
| `extension/options.{html,js}` | defaults page (`chrome.storage.sync`) |
| `native/mpv_ai_host.py` | native messaging host: framed JSON in, framed JSON out |
| `native/extension_id.py` | derives the extension ID from the manifest key |
| `native/install_native_host.sh` | registers the host per browser profile root, then pings it |

## Development

```sh
python3 native/extension_id.py                 # print the pinned extension ID
python3 -m py_compile native/*.py
shellcheck -x native/*.sh
```

`.github/workflows/ci.yml` runs those, validates every JSON manifest, and
installs the host into a throwaway `$HOME` and round-trips a real `ping`
through it, so a broken host fails CI instead of failing silently in the
browser.

## Provenance

Split out of [`mpv-nvidia-ai-chain`](https://github.com/HyperRamzey/mpv-nvidia-ai-chain)
(commit `ee87216` added the extension and the native host there; `main` no
longer carries them). The player, the filters, the profile set and the Arch
package all live in that repository.
