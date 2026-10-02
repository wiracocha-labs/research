# [2026-09-19] — Verificación del estado MVP de yachay, chaka y quipu-ipfs

## Contexto

Revisión del estado real del código (no de los READMEs) para decidir cuál
de los tres proyectos del ecosistema tiene un MVP testeable por usuarios
externos hoy.

## Setup

- **Componente**: yachay, chaka (compresor), quipu-ipfs
- **Qué probamos**: ejecución de los binarios compilados, corridas de
  `cargo test`, inspección de comandos disponibles y fixtures de prueba
- **Hardware/entorno**: macOS 27.0, arm64, 8 cores, 16 GB RAM
  (11.2 GB usable detectados por yachay)
- **Flags relevantes**: binarios debug en `target/`; no existen builds
  de release ni artefactos publicados

## Resultados

| Métrica | Valor esperado | Valor obtenido | Diferencia |
|---------|---------------|----------------|------------|
| yachay: recomendación correcta para `--task code` | Modelo que quepa en ~11 GB RAM | Qwen2.5-Coder 7B (6.8 GB), ~16 tok/s estimados, comando `ollama run` incluido | Correcto |
| yachay: tamaño de la DB curada | ≥10 modelos | 15 modelos | Superado |
| yachay: tests | Pasan | 7/7 | Correcto |
| wcx: pipeline compress → decompress → verificar | Round-trip completo | Solo existen `compress` y `stats` | **Decompress no existe** |
| wcx: tests | Pasan | 10/10 | Correcto |
| wcx: fixture .safetensors en repo | Al menos uno para testers | Ninguno | Faltante |
| quipu: demo put/get entre dos procesos | Intercambio por hash con firma válida | Funciona en LAN vía mDNS | Correcto |
| quipu: tests | Pasan | 15/15 | Correcto |

## Observaciones

- **yachay** es un MVP funcional de punta a punta: detecta hardware,
  recomienda con razón explicita, lista alternativas y da el comando de
  instalación. El único bloqueo para testers no técnicos es que requiere
  `cargo build` — faltan binarios de release (GitHub Actions).
- **chaka** comprime pero el viaje es de ida: sin `wcx decompress` no hay
  round-trip verificable, y al ser int8 lossy tampoco hay forma de
  comprobar que el modelo recuperado sigue funcionando. Un tester además
  debe traer su propio `.safetensors`.
- **quipu-ipfs** es una demo de principio, no un producto: `MemoryStore`
  es volátil (el objeto muere con el nodo) y mDNS lo limita a la LAN.
- **Insight de diseño registrado en el whitepaper 0001**: la compresión
  entrópica sobre el tensor completo (pipeline actual de wcx) destruye el
  acceso aleatorio a bloques, lo cual es incompatible con ejecución por
  streaming/mmap. Motiva el rediseño de `.wcx` en dos perfiles
  (ejecución / transferencia).

## Siguiente paso

- yachay → workflow de release con binarios para macOS/Linux/Windows y
  primer grupo de testers con hardware variado (criterio de salida de
  Phase 1).
- chaka → implementar `wcx decompress` + fixture mínimo + especificar el
  perfil de ejecución del formato (ver whitepaper 0001, §3).
- quipu-ipfs → sin acción inmediata; depende de Fase 1 (mensajería
  cifrada) según ROADMAP.

---

## Actualización (2026-10-01)

- **yachay**: el bloqueo de binarios quedó **resuelto**. Se publicó
  `v0.1.0` el 2026-09-20 con binarios para 5 plataformas (macOS
  Apple Silicon/Intel, Linux x86/ARM, Windows x64) e instaladores
  `.sh`/`.ps1`. El criterio de Phase 1 (instalación sin tocar el código)
  ya es cumplible por testers.
- **chaka**: sin cambios — `wcx decompress` y el fixture `.safetensors`
  mínimo siguen pendientes.
- **quipu-ipfs**: sin cambios respecto a esta entrada.
