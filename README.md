# brevity-examples

A small corpus for demonstrating the `brevity` skill on a range of documents.

## How to read this repo

- **`main`** holds four documents: three reproduced from permissively licensed sources and one original sample.
- The **`brevity-pass`** branch holds the same documents after each was run through the `brevity` skill, one commit per document.
- Open `brevity-pass` as a pull request against `main`. The diff is the comparison.

## Documents

| File                                       | Source                                     |
| ------------------------------------------ | ------------------------------------------ |
| `docs/01-saas-product-launch.md` | Fictional SaaS product launch post (original) |
| `docs/02-nodejs-dont-block-event-loop.md`  | Node.js "Don't Block the Event Loop" guide |
| `docs/03-deno-pr-34903.md`                 | Deno pull request #34903                   |
| `docs/04-kubernetes-security-checklist.md` | Kubernetes security checklist              |

Full provenance and licenses are in [ATTRIBUTION.md](ATTRIBUTION.md).
