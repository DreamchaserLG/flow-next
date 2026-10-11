# flow-next-drive and flow-next-qa drive mobile apps through agent-device

## Goal & Context

<!-- Source: 10% user / 10% [paraphrase] / 80% [inferred] -->

flow-next-drive declines native mobile apps today: "iOS / iPadOS app driving is out of scope — the request is declined and deferred to the community iOS simulator skills" (`plugins/flow-next/skills/flow-next-drive/SKILL.md:107`, 8.5.3). flow-next-qa drives through flow-next-drive, so a spec for a React Native, Expo or native mobile app gets no live QA pass. [inferred]

agent-device ([callstack/agent-device](https://github.com/callstack/agent-device), MIT, maintained by Callstack; v0.21.25 on 2026-10-11, about 5k stars, pre-1.0 and releasing several times a week) is a CLI, MCP server and typed Node API that lets an agent drive iOS, Android, tvOS, macOS and Linux apps on simulators, emulators and real devices (XCTest on iOS, ADB on Android, AT-SPI on Linux), through semantic snapshots with element refs rather than coordinates. Docs: [oss.callstack.com/agent-device](https://oss.callstack.com/agent-device/docs/agent-setup). [inferred]

T3 Code ships it (read at `f23d0084b6`, 2026-10-11): a pinned `agent-device` version (`apps/server/src/device/deviceToolVersions.json`), installed after a consent step; the agent gets four lifecycle tools (`device_list`, `device_open`, `device_screenshot`, `device_close`, `apps/server/src/mcp/toolkits/device/tools.ts`) and does all tapping and typing through the `agent-device` CLI on its PATH; `device_open` returns the device-pinned command and its usage (`handlers.ts:22-63`), so the instructions cost nothing until a device is opened. [inferred]

MergeFoundry asked for this upstream after reviewing it on 2026-10-11: it would run a mobile QA pass on the machine that has the simulator (its delegate-to-machine children) and needs the driver to come from flow-next, not from MergeFoundry. The owner asked for the spec in this repo. [paraphrase]

## Architecture & Data Models

- **A mobile route in flow-next-drive.** Surface detection gains native mobile apps (iOS simulator or device, Android emulator or device, and the other platforms agent-device covers). The route drives through the `agent-device` CLI, at a pinned version the plugin documents, installed by the operator (drive never installs it silently). The SKILL.md line that declines iOS is replaced. [inferred]
- **Lazy instructions.** Like T3's `device_open`, the mobile route's usage lives in a reference file the skill reads only when the surface is mobile, so web and desktop drive runs carry no mobile text. [inferred]
- **flow-next-qa** uses the route unchanged: scenarios from the spec, evidence as screenshots and snapshots, the same `qa_verdict` receipt. [inferred]
- **Feature map:** `.flow/features/` entries can name a mobile surface and how to reach a screen (deep link, launch arguments, navigation steps). [inferred]

## Edge Cases & Constraints

- No simulator, emulator or device available: the route says what is missing and stops, never boots one without the request asking for it. [inferred]
- agent-device not installed or at another version: a clear message with the install command for the pinned version. [inferred]
- Real devices need the platform's signing and pairing; out of scope beyond saying so. [inferred]
- agent-device is pre-1.0: pin, and record the version in each QA receipt. [inferred]

## Acceptance Criteria

- **R1:** flow-next-drive detects a native mobile surface and drives it through the pinned `agent-device` CLI: open the app, read a snapshot, tap, type, scroll, capture a screenshot; the iOS-out-of-scope line is replaced. [inferred]
- **R2:** The mobile instructions load only on the mobile route (a reference file), and the base SKILL.md grows by at most a few lines. [inferred]
- **R3:** flow-next-qa runs a spec's scenarios on a mobile app through R1 and writes its usual receipt, with the agent-device version recorded. [inferred]
- **R4:** Missing driver, missing device and version mismatch each end in a one-line reason with the next step; nothing is installed or booted silently. [inferred]
- **R5:** The feature map format documents a mobile surface entry. [inferred]
- **R6:** Docs (flow-next.dev drive and QA pages) describe the route, the pinned version and the licence (MIT). Verified live on at least one iOS simulator and one Android emulator. [inferred]

## Boundaries

- No device panel or screen streaming in flow-next; that belongs to a host UI (T3's panel, a cockpit). [inferred]
- No bundled copy of agent-device; the operator installs the pinned version. [inferred]
- No real-device signing or provisioning flows. [inferred]

## Decision Context

- 2026-10-11, MergeFoundry coordinator's research (t3code `f23d0084b6`, agent-device v0.21.25): verdict "borrow the pattern": add agent-device as the mobile route in drive and QA upstream; no cockpit device panel. [inferred]
- 2026-10-11, owner: "do a spec inside of the flow-next repo, get it pushed there". [user]
- Trigger: Michał Pierzchała (@thymikee, Callstack), 2026-10-10, on T3 Code's device panel "powered by agent-device" (https://x.com/thymikee/status/2108971903968845894). [inferred]

## Parked unknowns

- When to build: MergeFoundry suggested waiting for a mobile spec that needs it; the owner decides. [inferred]
- Whether to use agent-device's MCP server instead of the CLI (T3 uses the CLI with four lifecycle tools). [inferred]
