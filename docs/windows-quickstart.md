# Running NInfer from Windows

A practical runbook for getting either registered artifact serving on a chosen port from a Windows
machine. NInfer has no native Windows build (see [Windows port](windows-port.md)), so every path
here runs the Linux build under WSL2.

Verification status is stated per path. Do not assume a path was tested end to end unless it says
so.

## What you need

- Windows 11 with an RTX 5090 and a current NVIDIA driver. Install the driver on **Windows only**.
  Never install a Linux display driver inside WSL; the Windows driver projects the GPU into WSL.
- WSL2 with a distribution. Ubuntu 24.04 matches the container base image, which matters for
  path C.
- Roughly 40 GB of free space inside the WSL filesystem if you want both artifacts.
- Docker Desktop, for paths B and C only.

Confirm the GPU is visible from inside WSL before anything else:

```bash
wsl -e nvidia-smi --query-gpu=name,driver_version,memory.total --format=csv
```

If that does not report your GPU, stop and fix it. Nothing below will work until it does.

## Where the artifact must live

Put the `.ninfer` file on the **WSL filesystem** (for example `~/models`), not on a mounted Windows
drive under `/mnt/c`.

The loader opens the artifact with `O_DIRECT` and has no buffered fallback
(`src/artifact/reader.cpp`). On WSL ext4 this is measured at 3.3 to 4.1 GiB/s, so a 20 GiB artifact
uploads in about 5 seconds. Whether a Windows drive mount honors or rejects the flag is untested,
and routing 20 GiB through the filesystem translation layer would be slow regardless.

## Choosing a path

| | A: native WSL build | B: Docker | C: Docker build, native run |
|---|---|---|---|
| Builds tests and benchmarks | yes | no | no |
| Needs CUDA toolkit installed in WSL | yes | no | no |
| Needs Docker Desktop | no | yes | yes (build only) |
| Artifact on ext4 without extra setup | yes | needs WSL integration enabled | yes |
| Verified end to end here | no | partially | yes |

Path C is the one validated end to end in this repository's testing, and it is the least
configuration for a working server. Path A is the right choice if you intend to develop.

## Path A: build inside WSL

Follow the project README's build instructions inside the WSL distribution. Install the CUDA toolkit
from NVIDIA's **WSL-specific** repository, which deliberately omits the display driver, then the
build dependencies the `Dockerfile` lists (`cmake`, `ninja-build`, `pkg-config`,
`libavformat-dev`, `libavcodec-dev`, `libavutil-dev`, `libswscale-dev`, `libcurl4-openssl-dev`),
then configure and build as the README describes.

This is the only path that builds `BUILD_TESTING`, `NINFER_BUILD_BENCHMARKS`, and
`NINFER_BUILD_TOOLS`. It requires root in the distribution.

Not executed during this repository's testing. The commands are the upstream README's, unmodified.

## Path B: Docker

```powershell
docker build --tag ninfer:local .
```

Verified: the image builds from the committed `Dockerfile`, both binaries respond to `--help`, and
`--gpus all` correctly exposes the GPU to a container.

The caveat is the artifact mount. Docker Desktop's per-distro WSL integration is **off by default**.
Without it, the `docker` CLI inside a WSL distribution is the Windows executable and cannot
bind-mount that distribution's paths, so `-v /home/you/models:/models` silently fails to resolve.
Either enable WSL integration for your distribution in Docker Desktop settings, or keep the artifact
in a Docker named volume.

Running a model inside the container was not validated here, because of that mount issue. The image
itself is known good.

## Path C: build in Docker, run natively in WSL

This gets a working server without installing the CUDA toolkit in WSL and without root. It works
because the runtime image and Ubuntu 24.04 share a base, so the shared libraries are compatible.

Build the image, then lift the binaries out:

```powershell
docker build --tag ninfer:local .
docker create --name ninfer-extract ninfer:local
docker cp ninfer-extract:/usr/local/bin/ninfer       .\ninfer
docker cp ninfer-extract:/usr/local/bin/ninfer-serve .\ninfer-serve
docker rm ninfer-extract
```

The binaries need the FFmpeg library closure, which a stock Ubuntu WSL install does not have.
Collect it from the image rather than installing packages, which avoids needing root. Run this
inside a container with a directory mounted for the output, collecting the closure of
`ninfer-serve` while excluding the loader, C library, and C++ runtime (overriding those on the host
causes hard-to-diagnose crashes):

```bash
ldd /usr/local/bin/ninfer-serve | awk '{print $3}' | grep '^/' | sort -u
```

Copy each result except `ld-linux*`, `libc`, `libm`, `libdl`, `librt`, `libpthread`, `libresolv`,
`libstdc++`, and `libgcc_s` into a staging directory, archive it, and unpack it to `~/lib` in WSL.
Place the two binaries in `~/bin`.

Then every invocation sets the library path:

```bash
export LD_LIBRARY_PATH=$HOME/lib
```

Verify:

```bash
ldd ~/bin/ninfer-serve | grep 'not found'   # expect no output
~/bin/ninfer-serve --help | head -2
```

The library closure is a one-time cost, not a per-build one. Rebuilding for a newer engine only
replaces the two binaries in `~/bin`; the staged `~/lib` continues to satisfy them, because the
runtime image's FFmpeg and libcurl majors are what the closure was cut from. Re-run the `not found`
check after any rebuild anyway, since a base-image bump is exactly the change that would invalidate
it. Keep the previous binaries alongside the new ones under a distinguishing suffix so a bad build
can be backed out without a rebuild.

## Downloading an artifact

All five artifacts are public and need no authentication.

| Model | File | Size (bytes) | SHA-256 |
|---|---|---:|---|
| Qwen3.6-27B | `qwen3_6_27b.ninfer` | 17,495,365,888 | `7b51600ffd10632b9660f56085efdd9b751d79733ad32036a652234b64bebe7b` |
| Qwen3.6-27B NVFP4 | `qwen3_6_27b_nvfp4.ninfer` | 18,324,064,000 | `bce5f00d066c0f20f1317bf1fdcb458264cf95837c3b1f3fbec163694627893a` |
| Qwen3.8-27B | `qwen3_8_27b.ninfer` | 18,210,531,328 | `eec39564993d6e9c7d5e383382a760f093465c9d163ec9a1bd6b80199514bf3e` |
| Qwen3.8-27B NVFP4 | `qwen3_8_27b_nvfp4.ninfer` | 21,492,695,040 | `bb3360522a06e136e0367f5703414d26272b7285c8a6ab6194135c17dbd81b32` |
| Qwen3.6-35B-A3B | `qwen3_6_35b_a3b.ninfer` | 22,783,246,080 | `1fb9ea0b5b8561e49d9604115ec89e5d9f2b6f6434e32c37c57fffd480a325d2` |

The URL pattern is `https://huggingface.co/neroued/<REPO>/resolve/main/<FILE>`, with repositories
`Qwen3.6-27B-NInfer`, `Qwen3.6-27B-nvfp4-NInfer`, `Qwen3.8-27B-NInfer`,
`Qwen3.8-27B-nvfp4-NInfer`, and `Qwen3.6-35B-A3B-NInfer`.

The project README is the authority for this table; check it before trusting these hashes, because
the container format has been revised once and the version-2 rewrite changed every artifact's
digest without changing its weights. A file whose size matches but whose hash does not is most
likely a version-1 download that needs the migration described in the README.

**Use a chunked or parallel downloader, not a single-stream `curl -O`.** A single connection was
observed dropping repeatedly against the CDN and restarting from byte zero, making no net progress
across several attempts. Fetching fixed-size byte ranges with independent short-lived requests
solved it, and running six such requests concurrently reached roughly 100 MiB/s against a single
stream's 15 MiB/s. The Hugging Face CLI is a reasonable alternative if you have it, since it
chunks and parallelizes internally.

Always verify before use. A truncated or mis-resumed artifact will otherwise surface as a confusing
runtime error:

```bash
sha256sum ~/models/qwen3_6_35b_a3b.ninfer
```

## Running the CLI

```bash
export LD_LIBRARY_PATH=$HOME/lib
~/bin/ninfer ~/models/qwen3_6_35b_a3b.ninfer \
  --prompt "Explain prefill and decode in three sentences." \
  --max-context 16384 \
  --max-new 256
```

Answer content goes to stdout; progress, reasoning, timings, and memory go to stderr. Redirect them
separately when capturing output.

A quick correctness check, run from the repository root because the fixture paths are
repository-relative. This must print exactly `42`:

```bash
~/bin/ninfer ~/models/qwen3_6_35b_a3b.ninfer \
  --messages examples/cli/messages/text_smoke_zh.json \
  --no-thinking --greedy --max-new 8 --max-context 8192
```

## Running the server on a specific port

Use `--port`. Both targets take identical flags; only the artifact path and `--model-id` differ.

```bash
export LD_LIBRARY_PATH=$HOME/lib

# Qwen3.6-35B-A3B on port 9001
~/bin/ninfer-serve ~/models/qwen3_6_35b_a3b.ninfer \
  --model-id qwen3.6-35b-a3b \
  --host 127.0.0.1 --port 9001 \
  --max-context 16384 \
  --spec mtp --draft-tokens 3 --lm-head-draft

# Qwen3.6-27B on port 9002
~/bin/ninfer-serve ~/models/qwen3_6_27b.ninfer \
  --model-id qwen3.6-27b \
  --host 127.0.0.1 --port 9002 \
  --max-context 16384 \
  --spec mtp --draft-tokens 3 --lm-head-draft
```

Startup including weight upload takes about 5 seconds. The server is ready when it accepts a
request; there is no separate readiness endpoint, so poll the completions route.

Two artifacts cannot share one process. Run two processes on two ports, and size `--max-context`
so both fit in VRAM at once: at the full 262,144-token context a single target reserves 22 to 26 GiB
(see [Windows port](windows-port.md) for the per-configuration table), so two concurrent servers
need a much smaller context each.

`--model-id` sets the string clients must send as `"model"`. It does not have to match the table
above, but the client and server must agree.

## Reaching it from the same Windows machine

WSL2 forwards Windows `localhost` into the distribution, and this works even when the server binds
`127.0.0.1` inside WSL. Verified: a request from Windows PowerShell to `http://127.0.0.1:9001`
reached a server bound to `127.0.0.1:9001` inside WSL and returned a completion.

```powershell
Invoke-RestMethod -Uri 'http://127.0.0.1:9001/v1/chat/completions' `
  -Method Post -ContentType 'application/json' -TimeoutSec 120 -Body @'
{
  "model": "qwen3.6-35b-a3b",
  "messages": [{"role": "user", "content": "Say hi in three words."}],
  "max_tokens": 40,
  "enable_thinking": false
}
'@
```

For local-only use, keep `--host 127.0.0.1`. Nothing further is required.

## Reaching it from another machine

The obvious approach does not work, so read this before spending time on firewall rules.

**A WSL2 listener is not reachable from outside the host, whatever it binds.** Binding
`--host 0.0.0.0` inside WSL is not sufficient and, on its own, not even necessary. WSL2 runs in a
lightweight VM with its own network stack, so a socket opened there does not appear on the Windows
host's interfaces. Windows can reach it over loopback only, through WSL's localhost relay.

**Mirrored networking does not fix this.** It is tempting, because `networkingMode=mirrored` in
`%USERPROFILE%\.wslconfig` genuinely does make the host's interfaces visible from inside WSL,
including a VPN interface. But mirroring an interface *into* WSL is not the same as publishing a WSL
*listener* on the host. Measured with mirrored mode active and a server bound to `0.0.0.0:9001`
inside WSL: `Get-NetTCPConnection -LocalPort 9001` on the host returned nothing, connections to both
the host's LAN address and its VPN address timed out, and only `127.0.0.1:9001` answered. Firewall
rules are not the blocker in this configuration, because there is no host-side socket for inbound
packets to reach. Do not start here.

### What works: proxy from the host

Put something on the Windows side that listens on a host interface and forwards to
`127.0.0.1:<port>`, since that loopback path to WSL is reliable.

With Tailscale this is one command, and it is the verified configuration:

```powershell
tailscale serve --bg --tcp 9001 tcp://127.0.0.1:9001
```

Verified end to end: a peer on the tailnet posted to
`http://<host-tailnet-ip>:9001/v1/chat/completions` with a bearer token and received a completion.

This is the preferable route for several reasons beyond the fact that it works. It needs no
Administrator shell, no firewall rules, and no change to WSL networking, because `tailscaled`
terminates the connection itself and connects onward over loopback. Exposure is limited to the
tailnet by construction rather than by a firewall rule you have to get right.

Inspect and remove it with:

```powershell
tailscale serve status
tailscale serve --tcp=9001 off
```

Two things to know. The host **cannot reach its own tailnet address** through this proxy, because
`tailscale serve` listens inside tailscaled's userspace network stack rather than as an operating
system socket. Testing from the machine itself will time out even when the proxy is working
perfectly, and `Get-NetTCPConnection` will never show the listener. Verify from an actual peer. Also
note the forwarded connection arrives at the server from `127.0.0.1`, so the server cannot
distinguish remote callers by address.

Without Tailscale, the equivalent is a Windows port proxy plus a firewall rule, in an Administrator
shell:

```powershell
netsh interface portproxy add v4tov4 listenport=9001 listenaddress=0.0.0.0 `
  connectport=9001 connectaddress=127.0.0.1

New-NetFirewallRule -DisplayName "NInfer 9001" -Direction Inbound `
  -Protocol TCP -LocalPort 9001 -RemoteAddress <your-subnet> -Action Allow
```

Scope `-RemoteAddress` to the network you actually intend to serve. Omitting it exposes the port to
every network the host is attached to.

### Authentication is not optional here

Set `--api-key` on any server reachable beyond loopback:

```bash
export LD_LIBRARY_PATH=$HOME/lib
~/bin/ninfer-serve ~/models/qwen3_6_35b_a3b.ninfer \
  --model-id qwen3.6-35b-a3b \
  --host 127.0.0.1 --port 9001 \
  --api-key "$(cat ~/.ninfer_api_key)" \
  --max-context 16384 \
  --spec mtp --draft-tokens 3 --lm-head-draft
```

Verified: with a key configured the server rejects an unauthenticated request. Clients send
`Authorization: Bearer <key>`.

`--host 127.0.0.1` is sufficient when a host-side proxy is doing the forwarding, and it is the
tighter choice: nothing but the proxy can reach the server even from within the host. Bind `0.0.0.0`
only if something must reach WSL directly on its own interface.

### Verifying from the other machine

```bash
curl -s -m 60 http://<host-tailnet-ip>:9001/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_KEY' \
  -d '{"model":"qwen3.6-35b-a3b","messages":[{"role":"user","content":"hi"}],"max_tokens":32,"enable_thinking":false}'
```

A completion means it works. An `invalid_api_key` body also means the network path works and only
the key is wrong. A timeout means no host-side proxy is listening, or a firewall is dropping the
connection before it reaches one.

## Switching between models

One artifact per process, and any two sets of weights exceed a 32 GiB card, so the targets cannot
run at once. Swap instead. A wrapper keeps it to one command; the maintained copy lives at
`tools/ninfer-switch` in the repository, and the listing below is the shape of it:

```bash
#!/usr/bin/env bash
# ninfer-switch 27b|38|35b
set -eu
case "${1:-}" in
  27b) ART=qwen3_6_27b.ninfer;     ID=qwen3.6-27b ;;
  38)  ART=qwen3_8_27b.ninfer;     ID=qwen3.8-27b ;;
  35b) ART=qwen3_6_35b_a3b.ninfer; ID=qwen3.6-35b-a3b ;;
  *) echo "usage: ninfer-switch 27b|38|35b"; exit 1 ;;
esac
PORT=9011
p=$(ps -eo pid,comm --no-headers | awk '$2=="ninfer-serve"{print $1}')
[ -n "$p" ] && kill $p
for _ in $(seq 1 30); do ss -ltn 2>/dev/null | grep -q ":$PORT " || break; sleep 1; done
export LD_LIBRARY_PATH="$HOME/lib"
setsid nohup "$HOME/bin/ninfer-serve" "$HOME/models/$ART" \
  --model-id "$ID" --host 127.0.0.1 --port "$PORT" \
  --api-key "${NINFER_KEY:-changeme}" \
  --max-context "${NINFER_CTX:-262144}" --kv-dtype int8 \
  --spec mtp --draft-tokens 3 --lm-head-draft \
  </dev/null >"$HOME/serve.log" 2>&1 &
for _ in $(seq 1 60); do
  ss -ltn 2>/dev/null | grep -q ":$PORT " && { echo "serving $ID on $PORT"; exit 0; }
  sleep 1
done
echo "failed to start; see ~/serve.log"; exit 1
```

Takes about 40 seconds, almost all of it weight upload. Clients must send the `--model-id` it
prints; the server rejects any other `model` value.

The wait-for-port loop is not optional. A new server started immediately after killing the old one
fails with `failed to bind`, because the socket has not been released yet.

## Gotchas

**Thinking is on by default, and it consumes the output budget.** A small `max_tokens` can be spent
entirely on `reasoning_content`, returning empty `content` with `finish_reason: "length"`. Send
`"enable_thinking": false` for short factual replies, or raise `max_tokens`. The CLI equivalent is
`--no-thinking`.

**One request at a time.** The engine holds one resident sequence and serializes execution. A second
concurrent request waits; it is not batched.

**Capabilities are fixed at startup.** Vision (`--vision`) and speculative decoding must be enabled
on the process that will use them. A later request cannot turn them on. DFlash is available only for
the 35B-A3B target and is text-only.

**`pkill -f ninfer-serve` can kill your own shell**, because the pattern appears in the invoking
shell's own command line. Match on the process name instead:

```bash
kill $(ps -eo pid,comm --no-headers | awk '$2=="ninfer-serve"{print $1}')
```

**Do not put the artifact on `/mnt/c`.** See the note near the top.

**Another GPU tenant can make the load fail late, after the weights are already resident.** The
engine sizes its runtime reservation against free device memory at startup, so a 27B artifact that
uploads its weights fine can still abort with `requested Engine runtime reservation requires N
bytes, but only 0 bytes are available for runtime capacity`. On a 32 GiB card, roughly 15.9 GiB of
weights plus the KV cache and workspace plan for about 17.4 GiB, which leaves very little room for a
second resident workload. Ollama, a training container, and the Windows desktop compositor all
qualify. Check `nvidia-smi --query-gpu=memory.used,memory.free --format=csv` before blaming the
artifact, and note the same contention depresses decode throughput well below the published figures
without failing outright. `--kv-dtype int8` lowers the KV cost but not the weight cost, so it does
not rescue a card that is already half occupied.

**Tool names are capped at 64 characters on the OpenAI route** (`src/serve/openai_schema.cpp`),
and at 128 on the Anthropic route (`src/serve/anthropic_schema.cpp`). Both allow only
`[A-Za-z0-9_-]`. This matches OpenAI's own specification, so a client sending longer names is
off-spec, but the failure reads as an opaque `400 function name must match [A-Za-z0-9_-]{1,64}`
that does not say which tool is at fault, and the request log records `tool_count` but not names.
Agent clients that namespace tools with a repeated plugin prefix hit this easily. Either drop the
offending tool sources, shorten the prefix, or use `/v1/messages` for the larger budget.

**Under mirrored networking, WSL and Windows share a port space.** A port claimed by a Windows
process, including one `tailscale serve` is publishing, cannot then be bound inside WSL. Give the
server its own internal port and let the proxy map the public one, rather than using the same
number on both sides.

## Related

- [Windows port](windows-port.md): native build status, measured WSL2 performance for both targets.
- [GPU portability](gpu-portability.md): what the `sm_120a` target requires.
- [CLI](cli.md) and [HTTP serving](serving.md): the full option and endpoint reference.
