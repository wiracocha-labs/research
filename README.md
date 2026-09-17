# Wiracocha Labs — Research

Research documentation for Wiracocha Labs projects. Whitepapers, experiment
logs, and technical references.

**This repo does not contain code.** Source code lives in its own repos:

- [`chasqui-app`](https://github.com/wiracocha-labs/chasqui-app) — decentralized communication platform
- [`chasqui-server`](https://github.com/wiracocha-labs/chasqui-server) — Rust backend for Chasqui
- [`quipu-ipfs`](https://github.com/wiracocha-labs/quipu-ipfs) — P2P storage layer over IPFS/libp2p
- [`yachay`](https://github.com/wiracocha-labs/yachay) — hardware-aware local AI model recommender
- [`chaka`](https://github.com/wiracocha-labs/chaka) — AI model weight compressor

## Structure

```
whitepapers/       → formal research documents, numbered
bitacora/          → experiment logs (dated entries)
plantillas/        → templates for new entries
```

## Published articles

Articles published on [wiracocha.substack.com](https://wiracocha.substack.com)
are written and maintained in a separate workspace, then published. This
section links to them once live.

### How we build economic systems

19-part series on economic systems, institutions, and governance design.
Connects historical analysis with technical research on DAOs and
decentralized infrastructure.

_(links will be added as articles are published)_

## License — CC-BY-SA 4.0

This repository's content (documentation, whitepapers, figures) is released
under [Creative Commons Attribution-ShareAlike 4.0
International](https://creativecommons.org/licenses/by-sa/4.0/).

Why CC-BY-SA and not AGPL-3.0? AGPL is designed for software: it governs
execution, distribution, and modification of code. CC-BY-SA is designed for
creative works: it governs copying, adaptation, and attribution. For
technical documentation and editorial content, CC-BY-SA is the standard
license in open source (Wikipedia uses it, for example). It allows others to
translate, adapt, and republish the content as long as they attribute and
share under the same license.

## Publishing workflow

This repo is the versioned source of truth. Whitepapers and experiment logs
are adapted and published on [wiracocha.substack.com](https://wiracocha.substack.com)
as a distribution layer. The repo always comes first; Substack is a derivative.
