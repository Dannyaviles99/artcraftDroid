# Android device validation matrix

Use this matrix before calling the Android port usable. Record Android version, ABI, RAM, GPU/driver, screen size, and exact build commit for every run.

| Test | Helio G81 / 4 GB | Other ARM64 phone | Tablet / desktop mode |
|---|---:|---:|---:|
| Clean install and cold launch | [ ] | [ ] | [ ] |
| Login and token persistence | [ ] | [ ] | [ ] |
| Model list and provider selection | [ ] | [ ] | [ ] |
| Submit image generation | [ ] | [ ] | [ ] |
| Submit video generation | [ ] | [ ] | [ ] |
| Poll task and display result | [ ] | [ ] | [ ] |
| Open and save via Android SAF | [ ] | [ ] | [ ] |
| Rotate portrait / landscape | [ ] | [ ] | [ ] |
| Touch-only operation | [ ] | [ ] | [ ] |
| Mouse and keyboard operation | [ ] | [ ] | [ ] |
| External display / wide layout | N/A | [ ] | [ ] |
| Stylus pressure and tilt (if exposed) | [ ] | [ ] | [ ] |
| Memory pressure / background-resume | [ ] | [ ] | [ ] |
| 30-minute repeated preview stress test | [ ] | [ ] | [ ] |

## Constrained-memory checks

- Capture peak and steady-state PSS while opening, switching, and closing large previews.
- Verify decoded image dimensions and pixel count are bounded before allocation.
- Verify thumbnail and GPU texture caches have explicit limits and eviction.
- Verify task polling and image/video downloads do not retain duplicate full-size buffers.
- Verify Android memory-pressure callbacks release recreatable caches.
- Test app backgrounding during generation and confirm task state recovers after resume.

A test is not passed merely because the app remains open. Confirm the resulting feature works end-to-end.
