# Forensic Evidence Boundary Model

## Canonical transition model

```text
EXISTENCE
    ↓
CAPABILITY
    ↓
CONFIGURATION
    ↓
AUTHORIZATION
    ↓
INSTALLATION
    ↓
AVAILABILITY
    ↓
ACCESS
    ↓
INVOCATION
    ↓
EXECUTION
    ↓
EFFECT
    ↓
ATTRIBUTION

With a cross-cutting requirement:

        PROVENANCE / INTEGRITY
          ↙  ↓  ↓  ↓  ↓  ↓  ↘
       every evidence artifact
```

### Definitions

- **Existence:** Does the software, component, service, or integration exist?
- **Capability:** What can it theoretically do?
- **Configuration:** Is it configured to perform the relevant operation?
- **Authorization:** Does it have permission to perform that operation?
- **Installation:** Is the relevant component actually installed?
- **Availability:** Is the intended target currently reachable?
- **Access:** Did the component actually obtain the target or resource?
- **Invocation:** Was a specific operation requested?
- **Execution:** Did the underlying system actually perform it?
- **Effect:** What changed or occurred?
- **Attribution:** Who or what caused the observed effect?
- **Provenance / Integrity:** Can the evidence artifact itself be authenticated, correlated, and shown to be untampered?

## Evidentiary invariant

> **No evidentiary transition may be inferred merely because the adjacent transitions are established.**

Examples:

- GitHub access + XcodeBuildMCP installed + iPhone connected does **not** imply repository → Xcode → iPhone.
- The existence of a `build_run_device` operation does **not** imply that it was invoked.
- An application existing on an iPhone does **not** imply that ChatGPT deployed it.
- ChatGPT having authorization does **not** imply that ChatGPT exercised that authorization.

A second boundary is equally important:

> **Absence of evidence for a transition is not evidence that the transition was impossible; it means the transition remains unestablished unless independent evidence closes the gap.**

Likewise:

> **A resulting artifact establishes that an effect occurred; it does not, by itself, establish the actor or causal path that produced it.**

## Strong causal claim: required evidence chain

For a claim as specific as:

> “ChatGPT used repository X through Build iOS Apps/XcodeBuildMCP to build and deploy application Y to physical iPhone Z.”

the expected evidence chain is:

```text
ChatGPT session
      │
      ├── GitHub authorization/access
      │
      ▼
Repository X
      │
      ├── checkout/read/write evidence
      ▼
Local workspace
      │
      ├── MCP configuration
      ▼
XcodeBuildMCP
      │
      ├── invocation
      ▼
Xcode
      │
      ├── build result
      ▼
Signed application
      │
      ├── device deployment
      ▼
iPhone Z
      │
      ├── installation
      ├── launch
      ▼
Runtime evidence
      │
      └── correlated attribution
```

### Evidence domains

| Domain | Evidence needed |
|---|---|
| ChatGPT | Session/tool invocation and timestamp |
| GitHub | Authenticated repository operation |
| Repository | Commit/tree/workspace state |
| macOS | MCP configuration and installed package |
| MCP | Specific tool invocation and result |
| Xcode | Build/device operation evidence |
| Signing | Identity/profile actually used |
| Apple device tooling | Device availability and operation evidence |
| iPhone | Installation/launch/runtime evidence |
| Attribution | Temporal and causal correlation between events |
| Integrity | Hashes, timestamps, provenance, and preservation |

## Independent trust boundaries

GitHub access is a separate execution boundary from Xcode and physical-device access.

```text
GitHub authorization
        ≠
repository access
        ≠
repository modification
        ≠
local clone
        ≠
Xcode build
        ≠
physical-iPhone deployment
```

Similarly:

```text
XcodeBuildMCP installed
        ≠
MCP server running
        ≠
project exposed
        ≠
device available
        ≠
device operation invoked
        ≠
application installed
        ≠
application launched
        ≠
ChatGPT caused it
```

GitHub Actions is another independent execution path:

```text
GitHub → Actions → runner → build/test
```

Evidence of that path is not, by itself, evidence of:

```text
ChatGPT → local MCP → local Xcode → physical iPhone
```

## XcodeBuildMCP capability boundary

Documented operations such as `build_run_sim`, `launch_app_sim`, `build_run_device`, and `launch_app_device` establish capability and documented workflow. They do not establish that any particular operation was invoked or executed in a particular environment.

Physical-device workflows additionally require appropriate Xcode signing and device configuration. The existence of those prerequisites establishes neither their actual configuration nor their use for a particular deployment.

Where available, build/run results containing an application path, bundle identifier, process identifier, build-log path, runtime-log path, or operating-system log path can provide evidence for later transitions, but only when the artifacts are actually present, attributable, and integrity-preserved.

## Operational evidence rules

Use direct artifacts wherever possible:

```bash
git remote -v
git config --show-origin --get-regexp 'remote\..*|credential\..*'
git status --short --branch
```

Search repository-contained MCP references:

```bash
grep -RniE \
  'xcodebuildmcp|XcodeBuildMCP|mcpServers|mcp.*server' \
  . \
  --exclude-dir=.git \
  --exclude-dir=node_modules \
  --exclude-dir=.build \
  --exclude-dir=DerivedData
```

Review Git history:

```bash
git log --all --date=iso-strict \
  --format='%H %aI %cI %an <%ae> %s' \
  -n 50
```

Inspect local XcodeBuildMCP presence and Apple device visibility:

```bash
which xcodebuildmcp
xcodebuildmcp --version
xcodebuildmcp tools
npm list -g --depth=0 2>/dev/null | grep -i xcodebuild
npm cache ls xcodebuildmcp 2>/dev/null
ps aux | grep -i '[x]codebuildmcp'
xcrun devicectl list devices
```

A configuration such as:

```json
{
  "command": "npx",
  "args": ["-y", "xcodebuildmcp@latest", "mcp"]
}
```

establishes configuration only. It is not invocation or execution evidence.

Likewise, `xcrun devicectl list devices` establishes device visibility through Apple's development tooling, not that XcodeBuildMCP used the device.

## Conclusion standard

Security and forensic conclusions should stop at the highest transition independently established by evidence.

```text
EXISTENCE ≠ CAPABILITY ≠ CONFIGURATION ≠ AUTHORIZATION
          ≠ INSTALLATION ≠ AVAILABILITY ≠ ACCESS
          ≠ INVOCATION ≠ EXECUTION ≠ EFFECT
          ≠ ATTRIBUTION

and throughout:

PROVENANCE / INTEGRITY must support the evidence used
to establish each claimed transition.
```

This boundary prevents documented capability, authorization, observed artifacts, or software presence from being silently converted into claims of execution, causation, compromise, or attribution.
