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

**Qwen2.5-Coder-7B-Instruct · Q4_K_M**

| Métrica | Valor esperado | Valor obtenido | Diferencia |
|---------|---------------|----------------|------------|
| Tamaño original | — | 4.683.073.536 bytes (4,36 GiB) | — |
| Tamaño comprimido | — | 4.572.033.589 bytes | — |
| Ratio de compresión | 1.1x – 1.5x | **1,024x** | **muy por debajo** |
| Reducción | 10% – 50% | **2,37%** | — |
| Tiempo | — | 3m43s | — |

**Qwen2.5-Coder-7B-Instruct · Q2_K**

| Métrica | Valor esperado | Valor obtenido | Diferencia |
|---------|---------------|----------------|------------|
| Tamaño original | — | 3.015.940.032 bytes (2,81 GiB) | — |
| Tamaño comprimido | — | 2.937.087.848 bytes | — |
| Ratio de compresión | distinto al Q4_K_M | **1,027x** | **casi idéntico** |
| Reducción | — | **2,61%** | — |
| Tiempo | — | 2m04s | — |

## Observaciones

- La estimación previa del whitepaper era optimista por un factor de 4 a 20.
  No venía de ninguna fuente: estaba puesta como expectativa.
- **La curva es plana.** Pasar de 4 bits a 2 bits mueve la reducción de 2,37%
  a 2,61%. Dos décimas de punto. El techo no depende del nivel de cuantización
  en ese rango: se alcanza antes.
- El resultado **respalda** la hipótesis del documento: después de cuantizar,
  la redundancia que explotan los compresores lossless casi no existe.
- Es consistente con la nota de ZipNN: su ~33% aplica sobre BF16/FP32, donde
  la estructura del exponente todavía está. Post-cuantización ese margen se
  cierra.
- Consecuencia de diseño: el perfil de transferencia de `.wcx` no puede
  apoyarse en compresión entrópica sobre el tensor completo. El 2,5% no paga
  la complejidad. Los bytes tienen que venir de los deltas.

## Siguiente paso

- Repetir en las otras tres familias (LLaMA, DeepSeek, Mistral) para ver si el
  2,5% se sostiene fuera de Qwen.
- Medir el mismo pipeline con `gzip -9` y `brotli` para tener la curva de los
  tres codecs que el whitepaper menciona.
- Probar un nivel alto (Q8_0) para cerrar el rango: si también da ~2,5%, la
  curva es plana en todo el espectro y el argumento queda cerrado.
- Con esa tabla, el Nivel 2 queda cerrado como línea base de `chaka`.
