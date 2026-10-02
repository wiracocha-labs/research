# Compresión de pesos de modelos de IA para redes descentralizadas: motivación y mapa de investigación

**Whitepaper 0001** · Junio 2026 — actualizado septiembre 2026 · Wiracocha Labs

---

## 1. Resumen

El avance de modelos de lenguaje grandes (LLMs) y modelos fundacionales está
concentrado en organizaciones con acceso a clusters masivos de GPUs. Esta
centralización no es solo económica sino técnica: los modelos requieren
decenas de GB de memoria y ancho de banda que hardware de consumo no puede
sostener. Este whitepaper descompone el problema de compresión de pesos en
tres niveles —cuantización, compresión lossless post-cuantización y compresión
de deltas entre versiones—, revisa el estado del arte en cada nivel, e
identifica las brechas donde hay espacio de investigación original. El
proyecto `chaka` de Wiracocha Labs se enfoca en los niveles 1 y 3, donde
las soluciones existentes no cubren el caso de uso de redes descentralizadas,
y en una restricción transversal identificada durante el diseño: el formato
debe soportar ejecución por acceso aleatorio (mmap/streaming), no solo
compresión para almacenamiento.

## 2. Motivación

La inferencia y el entrenamiento de modelos grandes requieren, hoy,
hardware especializado. Una GPU de 80 GB (A100 u H100) cuesta del orden de
USD 15,000–40,000+ según modelo y mercado. Esto significa que:

- Solo grandes empresas y laboratorios pueden entrenar *y* servir modelos
  de última generación.
- El control del modelo (pesos, arquitectura, datos de entrenamiento) queda
  en manos de pocos actores.
- La diversidad de modelos, fine-tunings comunitarios y variantes locales
  se reduce porque el ecosistema asume hardware de datacenter.

La alternativa es una red descentralizada donde nodos con hardware modesto
(una laptop con GPU de 8-16 GB, o incluso CPU) puedan servir, compartir y
actualizar fragmentos de modelos. Esto requiere compresión agresiva que
lleve los pesos a un tamaño factible para transmisión P2P e inferencia en
el borde.

No aspiramos a reemplazar los datacenters. La tesis es que debe existir un
espectro: desde hardware especializado hasta hardware de consumo, pasando
por redes de nodos que colaboran. La compresión es la llave técnica para
que ese espectro exista.

## 3. El problema técnico en tres niveles

Es común hablar de "compresión de modelos" como si fuera un solo problema.
En realidad son tres problemas distintos, con métricas y restricciones
diferentes:

### Nivel 1: Cuantización (quantization)

Reduce la precisión numérica de los pesos: de FP32 (32 bits por peso) a
FP16, INT8, INT4 o incluso binario. El resultado directo es que el modelo
ocupa menos memoria y, crucialmente, **requiere menos hardware para
inferencia**: un modelo cuantizado a INT4 ocupa ~¼ de la VRAM que su
versión FP16 (y ~⅛ que FP32), o incluso puede correr en CPU con
instrucciones SIMD.

Métrica principal: *perplejidad o pérdida en task específicas vs. tamaño
en bits por peso*.

Trade-off: precisión vs. tamaño (y por tanto, vs. hardware mínimo
requerido).

### Nivel 2: Compresión lossless post-cuantización

Una vez que los pesos están en INT4 (por decir un formato), se pueden
comprimir con algoritmos de propósito general (zstd, gzip, brotli) para
reducir el tamaño en disco o en tránsito. Esto **no reduce los requisitos
de hardware de inferencia**: al cargar el modelo hay que descomprimir
todo a su representación cuantizada.

Métrica principal: *ratio de compresión sobre el binario cuantizado*.

Trade-off: velocidad de descompresión vs. ratio. Para uso en disco da
igual; para transmisión en red puede importar.

### Nivel 3: Compresión de deltas entre versiones

Cuando un modelo se actualiza (fine-tuning, entrenamiento distribuido,
corrección de pesos), los cambios suelen ser pequeños respecto al total.
En lugar de transmitir el modelo completo cada vez, se puede transmitir
solo la diferencia (delta) entre la versión anterior y la nueva.

Esto es irrelevante en el caso tradicional (descargas monolíticas de
Hugging Face) pero **crítico en redes P2P** donde los nodos tienen
ancho de banda limitado y actualizaciones frecuentes.

Métrica principal: *tamaño del delta vs. tamaño del modelo completo*.

Trade-off: eficiencia del delta vs. costo computacional de generarlo y
aplicarlo.

### Una restricción transversal: acceso aleatorio para ejecución

Los tres niveles anteriores describen compresión para *almacenamiento y
transmisión*. Hay una consideración adicional que no es un nivel sino una
restricción que los atraviesa: **si el objetivo es ejecutar el modelo
directamente desde el formato comprimido** —pesos residentes en SSD que
se cargan bajo demanda vía `mmap` o streaming—, el formato debe permitir
**acceso aleatorio a nivel de bloque**. Si un tensor se comprime como un
flujo continuo con un codec entrópico (zstd, LZ77+Huffman), se destruye
esa propiedad: no se puede leer "el bloque 47 de la capa 23" sin
descomprimir el flujo desde el inicio. (zstd sí admite bloques
independientes y un modo seekable, pero eso no equivale a un layout
direccionable por página compatible con `mmap`.)

Esto explica una decisión de diseño de GGUF que no es evidente a primera
vista: el formato de ejecución más usado del ecosistema **no aplica
compresión entrópica en absoluto** — solo cuantización con bloques
direccionables. Sacrifica ratio de compresión a cambio de que el motor
pueda mapear el archivo en memoria y leer solo las páginas que necesita.

Implicación para `chaka`: un único formato optimizado para ratio
máximo no puede servir a la vez para ejecución en streaming. La dirección
correcta es un formato con dos perfiles sobre la misma representación
cuantizada:

- **Perfil de ejecución** (`.wcx-exec`): cuantización + bloques
  direccionables + layout compatible con mmap. Comparable en espíritu
  a GGUF.
- **Perfil de transferencia** (`.wcx-ship`): el mismo contenido con codec
  entrópico por bloque + deltas entre versiones. Optimizado para
  transmisión P2P; se desenvuelve al perfil de ejecución al llegar al
  nodo destino.

## 4. Estado del arte

### Nivel 1 — Cuantización

El estándar de facto es GGUF (llama.cpp) con sus K-quants. GGUF soporta
múltiples niveles de cuantización (Q2_K a Q8_0) y es el formato que
permite correr LLMs en hardware de consumo hoy. Las técnicas subyacentes
incluyen GPTQ (cuantización post-entrenamiento con compensación de error),
LLM.int8() (de Tim Dettmers, que separa features outliers), y más
recientemente métodos como QuIP y AQLM que mejoran la calidad a bajos
bits.

El estado del arte académico en compresión extrema está en ~2 bits por
peso con pérdida aceptable en tareas estándar [Tseng et al., 2024 —
QuIP#; también AQLM]. Las técnicas de referencia a mayor precisión son
GPTQ (~4 bits) [Frantar et al., 2023] y LLM.int8() (8 bits) [Dettmers
et al., 2022].

### Nivel 2 — Compresión lossless post-cuantización

Sobre pesos ya cuantizados, la ganancia de compresión lossless se estima
limitada (~1.1x–1.5x con zstd -19; cifra a verificar con los benchmarks
comparativos pendientes del proyecto). La razón es que la cuantización
introduce ruido que aumenta la entropía del dato: un peso cuantizado a
INT4 tiene solo 16 valores posibles y la redundancia estructural que
explotan los compresores lossless es baja. La hipótesis de trabajo es que
el techo está cerca del límite informacional — confirmarlo con
mediciones es parte de los próximos pasos (§6).

### Nivel 3 — Compresión de deltas

Es el nivel donde la literatura es más escasa en el contexto de modelos
de IA. Existe trabajo en diferencias (diffs) de pesos para federated
learning [Konečný et al., 2016], pero asume servidores centralizados y
agregación promedio. Para redes P2P con topología dinámica y nodos que
pueden tener versiones ligeramente distintas del mismo modelo, no hay una
solución madura.

### Ejecución desde formato comprimido (streaming de pesos)

El ecosistema ya ejecuta modelos con pesos residentes en disco: GGUF se
abre con `mmap` y el sistema operativo trae páginas a RAM bajo demanda;
en modelos MoE los expertos se mapean bajo demanda con `mmap` (solo los
expertos activos residen en RAM, el resto permanece en disco); herramientas
como AirLLM ejecutan capa por capa leyendo del SSD; e investigación como
*LLM in a Flash* (Apple, 2023) formaliza el streaming de pesos desde
almacenamiento flash en dispositivos con memoria limitada. El patrón
común: el motor de inferencia es pequeño, los datos son grandes, y el
formato de ejecución debe soportar acceso aleatorio — no descompresión
secuencial. En el ecosistema Rust, mistral.rs implementa cuantización
in-situ (ISQ) y formatos ejecutables (UQFF) sobre candle, lo que demuestra
que un motor completo con estas propiedades es viable fuera de C++.

## 5. Dónde hay espacio real de investigación

El proyecto `chaka` se enfoca en tres frentes:

1. **Nivel 1: mejorar K-quants.** Los K-quants de GGUF son heurísticos
   y están optimizados para la familia de modelos LLaMA. Queremos explorar
   si esquemas de cuantización basados en búsqueda (como los de AQLM o
   QuIP#) pueden superar a K-quants en el caso general, especialmente para
   modelos fuera de la familia LLaMA (Qwen, DeepSeek, etc.). También
   interesa la cuantización mixta por capa guiada por métricas de
   importancia.

2. **Nivel 3: compresión de deltas para P2P.** Este es el terreno con
   menos solución previa. Diseñar un formato de delta que:
   - Sea eficiente cuando el cambio entre versiones es pequeño (<1% de
     los pesos).
   - No requiera tener ambas versiones completas en memoria para aplicar
     el delta.
   - Soporte merges: si el nodo A recibe un delta del nodo B y otro
     del nodo C, debe poder combinarlos.
   Esto no existe hoy como formato estándar.

3. **Formato de doble perfil (ejecución + transferencia).** Consecuencia
   directa de la restricción transversal descrita en §3: `.wcx` debe
   diseñarse como un formato con perfil de ejecución (bloques
   direccionables, compatible con mmap, dequant por bloque dentro del
   kernel) y perfil de transferencia (codec entrópico por bloque,
   deltas), compartiendo la misma representación cuantizada. El
   entregable aquí es una especificación — no solo una implementación —
   porque un formato solo tiene valor si un motor de inferencia existente
   (candle, mistral.rs) puede consumirlo.

## 6. Próximos pasos de investigación

1. Caracterizar la entropía y distribución de pesos post-cuantización en
   modelos de diferentes familias (LLaMA, Qwen, DeepSeek, Mistral) para
   establecer una línea base.
2. Implementar un pipeline de evaluación de cuantización que permita
   comparar K-quants, GPTQ, AQLM y QuIP# bajo las mismas condiciones.
3. Diseñar la especificación del formato de delta (estructura de datos,
   algoritmo de diff/apply, soporte de merge).
4. Especificar el perfil de ejecución de `.wcx` (layout de bloques,
   alineación, tabla de offsets compatible con mmap) comparándolo con el
   layout interno de GGUF, antes de escribir más código de compresión.
5. Evaluar el tamaño de delta esperado en escenarios realistas (fine-tuning
   parcial, corrección de pesos, actualización de capas específicas).
6. Publicar resultados como benchmark reproducible en este repositorio.

## 7. Referencias

- [Dettmers et al., 2022] LLM.int8(): 8-bit Matrix Multiplication for
  Transformers at Scale. arXiv:2208.07339
- [Frantar et al., 2023] GPTQ: Accurate Post-Training Quantization for
  Generative Pre-trained Transformers. arXiv:2210.17323
- [Tseng et al., 2024] QuIP#: Even Better LLM Quantization with Hadamard
  Incoherence and Lattice Codebooks. arXiv:2402.04396
- [Egiazarian et al., 2024] Extreme Compression of Large Language Models
  via Additive Quantization (AQLM). arXiv:2401.06118
- [Konečný et al., 2016] Federated Learning: Strategies for Improving
  Communication Efficiency. arXiv:1610.05492
- [Alizadeh et al., 2023] LLM in a Flash: Efficient Large Language Model
  Inference with Limited Memory. arXiv:2312.11514
- [Hershcovitch et al., 2024] ZipNN: Lossless Compression for AI Models.
  arXiv:2411.05239 — nota: sus ganancias (~33%) aplican sobre pesos en
  BF16/FP32 explotando la estructura del exponente; post-cuantización el
  límite informacional descrito en §4 (Nivel 2) se mantiene.
- Especificación GGUF / ggml. https://github.com/ggml-org/ggml
- AirLLM — inferencia capa por capa desde SSD.
  https://github.com/lyogavin/airllm
- mistral.rs — motor de inferencia en Rust con cuantización in-situ (ISQ)
  y formato ejecutable UQFF. https://github.com/EricLBuehler/mistral.rs
