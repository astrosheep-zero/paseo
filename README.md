# Pi tool-result images

Pi tool results carry image content blocks, but the provider mapped only text into the timeline. This fix uses the existing provider-image attachment and Markdown rendering path for completed tool results and history replay. It does not change the wire protocol or client UI.

## Real-provider verification

Environment: macOS arm64, Node 26.10.0, Pi 1.2.0, Paseo 0.11.2 source based on `d3efddb0b`, Chromium 156.0.8078.4. Model: `openai-codex/gpt-6-luna`, explicitly selected for this verification. Default Pi settings were not changed.

The unpatched and final patched daemons used separate homes and the same real Pi binary. The normal Paseo daemon was not restarted or replaced. Pi extension/skill discovery was disabled for this check; the real built-in `read` tool remained enabled.

Prompt:

```text
Use the read tool to read ./preview.png. Do not use any other tools. Then reply only "Image read complete.".
```

The input was a 960 × 540 browser screenshot of example.com. Both real model turns completed successfully.

| Check                      | Observed result                                                                                                                                                   |
| -------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Unpatched daemon           | Successful `read`, no image message or preview                                                                                                                    |
| Patched live event stream  | Completed `read`, image message, final answer; image base64 absent from tool output                                                                               |
| Patched web conversation   | Preview loaded at its original 960 × 540 dimensions; clicking it opened the existing lightbox                                                                     |
| Native-history reload      | Removed the input PNG and the materialized preview file, called `refreshAgent`, then fetched canonical history; its epoch changed and one image was reconstructed |
| Fresh browser after reload | Reconstructed image loaded with no previous browser cache                                                                                                         |
| Compact web, light theme   | Preview and surrounding conversation remained visible at 390 × 844                                                                                                |

The reconstructed bytes matched the original screenshot:

```text
SHA-256: ddfdc7bacd8bd186ae9061b24ac01098cf4cbadbac259c5f536f53051ff548b5
history epoch changed: 2aa0b093-d02e-43e3-b37b-7da97768c0be → d13a509b-a2fe-4d47-bc9a-fb26451e43d8
image count after native-history replay: 1
```

Screenshots are unmodified app captures:

- [Before](before-desktop.png)
- [After the real read](after-desktop.png)
- [Reopened native history, input and preview cache removed](history-desktop.png)
- [Existing image lightbox](image-lightbox.png)
- [Compact web in light theme](history-compact-light.png)

## Pi version compatibility

Checked the released tags from 0.99.0 through 1.2.0, including the 0.99.x and 1.0.x patch releases:

| Pi version   | Live `read` image result                                      | Saved `toolResult`                                        |
| ------------ | ------------------------------------------------------------- | --------------------------------------------------------- |
| 0.99.0–1.0.3 | `content`: text plus `{ type: "image", data, mimeType }`      | Image bytes retained in `content`                         |
| 1.0.4–1.2.0  | Same `content`, plus a duplicate image in `structuredContent` | Image bytes retained in `content`; no `structuredContent` |

The duplicate was introduced in [Pi's read/codemode fix](https://github.com/earendil-works/pi-mono/commit/021eae60a7f99a51612b5054818ee5ebfb75dc48), first released in 1.0.4. The adapter uses the common image block shape, clears the duplicate when present, and needs no version switch or history migration.

The published 0.99.0 and 1.0.4 built-in `read` tools were also executed directly, without model calls. Both actual results produced one preview through the patched result mapper. Each result's content was saved and reopened with that version's native `SessionManager`, after deleting the probe's source PNG and materialized preview. History replay reconstructed one image with the SHA-256 above, and no base64 remained in mapped tool output. This was an offline tool/history check, not an old-version model/UI end-to-end run; the real-provider browser evidence above uses Pi 1.2.0.

## Automated checks

The live regression failed on the unchanged base because the expected image message was absent. The history regression likewise failed before the history entry point was patched. Both passed with the fix. The live case covers results with and without `structuredContent`. A separate case checks image-only custom-tool output during partial updates and failure, including the error field and the duplicate `structuredContent` image.

```text
cd packages/server
../../node_modules/.bin/vitest run src/server/agent/providers/pi/agent.test.ts src/server/agent/providers/pi/history-mapper.test.ts --maxWorkers=1 --bail=1

Test Files  2 passed (2)
Tests       134 passed (134)
```

```text
npm run build:server                 passed
npm run typecheck                   passed, all workspaces
npm run lint -- <five changed Pi files>
Found 0 warnings and 0 errors.
npm run format:check:files -- <changed files>
```

## Coverage limits

The daemon and browser workflow were exercised on macOS, including dark desktop and light compact web layouts. Native iOS, Android, Electron desktop, Windows, and Linux were not exercised. Compact web is not a native-mobile test. No daemon restart was used as proof of history recovery; the check explicitly reloaded the native Pi session and used a fresh browser.
