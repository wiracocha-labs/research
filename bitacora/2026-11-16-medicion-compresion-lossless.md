# [2026-11-16] — Medición de compresión lossless sobre un modelo ya cuantizado

## Contexto

El whitepaper 0001 estimaba que comprimir lossless un modelo ya cuantizado
rendía entre 1.1x y 1.5x, sin medirlo. Era la única cifra del documento sin
respaldo, y justo la que sostiene el argumento del Nivel 2. Medirla convierte
la estimación en dato y define la línea base contra la que se va a evaluar
`chaka`.

## Setup

- **Componente**: `chaka` (investigación de compresión de pesos)
- **Qué probamos**: ratio de zstd sobre un GGUF ya cuantizado
- **Modelo**: Qwen2.5-Coder-7B-Instruct, GGUF Q4_K_M publicado por Qwen
- **Hardware/entorno**: MacBook M1 Pro, 8 núcleos, 16 GB RAM, macOS 27.0.1
- **Herramienta**: zstd 1.5.7, nivel 19, `-T0`

## Resultados

| Métrica | Valor esperado | Valor obtenido | Diferencia |
|---------|---------------|----------------|------------|
| Tamaño original | — | 4.683.073.536 bytes (4,36 GiB) | — |
| Tamaño comprimido | — | 4.572.033.589 bytes (4,26 GiB) | — |
| Ratio de compresión | 1.1x – 1.5x | **1,024x** | **muy por debajo** |
| Reducción | 10% – 50% | **2,4%** | — |
| Ahorro absoluto | — | 111 MB | — |
| Tiempo de compresión | — | 3m43s | — |

## Observaciones

- La estimación previa del whitepaper era optimista por un factor de 4 a 20.
  No venía de ninguna fuente: estaba puesta como expectativa.
- El resultado **respalda** la hipótesis del documento: después de cuantizar,
  la redundancia que explotan los compresores lossless casi no existe.
- Es consistente con la nota de ZipNN: su ~33% aplica sobre BF16/FP32, donde
  la estructura del exponente todavía está. Post-cuantización ese margen se
  cierra.
- Consecuencia de diseño: el perfil de transferencia de `.wcx` no puede
  apoyarse en compresión entrópica sobre el tensor completo. El ahorro de
  bytes tiene que venir de los deltas, no del codec.

## Siguiente paso

- Repetir la medición en las cuatro familias (LLaMA, Qwen, DeepSeek, Mistral)
  y en más niveles de cuantización (Q2_K a Q8_0) para ver si el 2,4% se mueve.
- Medir el mismo pipeline con `gzip -9` y `brotli` para tener la curva de los
  tres codecs que el whitepaper menciona.
- Con esa tabla, el Nivel 2 queda cerrado como línea base de `chaka`.
