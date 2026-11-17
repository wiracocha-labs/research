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

## Whitepapers

- **[0001 — Compressing AI model weights for decentralized networks](whitepapers/0001-compresion-pesos-ia-introduccion/)**
  · v1.0 · November 2026 · [English](whitepapers/0001-compresion-pesos-ia-introduccion/README.md)
  · [Español](whitepapers/0001-compresion-pesos-ia-introduccion/README.es.md)

## Published articles

Technical findings from this repo are adapted and published on
[wiracochalabs.substack.com](https://wiracochalabs.substack.com).

- [Bienvenidos a Wiracocha Labs](https://wiracochalabs.substack.com/p/bienvenidos-a-wiracocha-labs)
- [¿Qué hace pesado a un modelo de IA?](https://wiracochalabs.substack.com/p/que-hace-pesado-a-un-modelo-de-ia)
- [DeepSeek no es "más chico" que Opus](https://wiracochalabs.substack.com/p/deepseek-no-es-mas-chico-que-opus)
- [Los tres niveles de hacer un modelo más pequeño](https://wiracochalabs.substack.com/p/los-tres-niveles-de-hacer-un-modelo)

This repo only documents what we discover technically (whitepapers,
experiments, results). Editorial and management writing — essays,
governance, economics — lives outside this repo and is published only as
articles.

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
are adapted and published on [wiracochalabs.substack.com](https://wiracochalabs.substack.com)
as a distribution layer. The repo always comes first; Substack is a derivative.
