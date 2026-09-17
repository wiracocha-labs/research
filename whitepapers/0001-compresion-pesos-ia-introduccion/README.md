# Compresión de pesos de modelos de IA para redes descentralizadas: motivación y mapa de investigación

**Whitepaper 0001** · Junio 2026 · Wiracocha Labs

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
proyecto `compresor` de Wiracocha Labs se enfoca en los niveles 1 y 3, donde
las soluciones existentes no cubren el caso de uso de redes descentralizadas.

## 2. Motivación

La inferencia y el entrenamiento de modelos grandes requieren, hoy,
hardware especializado. Una GPU con 80 GB de memoria (A100, H100) cuesta
del orden de USD 30,000+. Esto significa que:

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
inferencia**: un modelo cuantizado a INT4 puede correr en una GPU con la
mitad de VRAM, o incluso en CPU con instrucciones SIMD.

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

## 4. Estado del arte

### Nivel 1 — Cuantización

El estándar de facto es GGUF (llama.cpp) con sus K-quants. GGUF soporta
múltiples niveles de cuantización (Q2_K a Q8_0) y es el formato que
permite correr LLMs en hardware de consumo hoy. Las técnicas subyacentes
incluyen GPTQ (cuantización post-entrenamiento con compensación de error),
LLM.int8() (de Tim Dettmers, que separa features outliers), y más
recientemente métodos como QuIP y AQLM que mejoran la calidad a bajos
bits.

El estado del arte académico está en ~2 bits por peso con pérdida
aceptable en tareas estándar [Dettmers, 2022; Frantar et al., 2023; Tseng
et al., 2024].

### Nivel 2 — Compresión lossless post-cuantización

Sobre pesos ya cuantizados, la ganancia de compresión lossless es
limitada (~1.1x–1.5x con zstd -19). La razón es fundamental: la
cuantización introduce ruido que aumenta la entropía del dato. Un peso
cuantizado a INT4 ya tiene solo 16 valores posibles; la redundancia
estructural que explotan los compresores lossless es baja. No se esperan
avances significativos aquí porque el límite es informacional.

### Nivel 3 — Compresión de deltas

Es el nivel donde la literatura es más escasa en el contexto de modelos
de IA. Existe trabajo en差分 (diferencia) de pesos para federated learning
[Konečný et al., 2016], pero asume servidores centralizados y agregación
promedio. Para redes P2P con topología dinámica y nodos que pueden tener
versiones ligeramente distintas del mismo modelo, no hay una solución
madura.

## 5. Dónde hay espacio real de investigación

El proyecto `compresor` se enfoca en dos frentes:

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

## 6. Próximos pasos de investigación

1. Caracterizar la entropía y distribución de pesos post-cuantización en
   modelos de diferentes familias (LLaMA, Qwen, DeepSeek, Mistral) para
   establecer una línea base.
2. Implementar un pipeline de evaluación de cuantización que permita
   comparar K-quants, GPTQ, AQLM y QuIP# bajo las mismas condiciones.
3. Diseñar la especificación del formato de delta (estructura de datos,
   algoritmo de diff/apply, soporte de merge).
4. Evaluar el tamaño de delta esperado en escenarios realistas (fine-tuning
   parcial, corrección de pesos, actualización de capas específicas).
5. Publicar resultados como benchmark reproducible en este repositorio.

## 7. Referencias

- [Dettmers et al., 2022] LLM.int8(): 8-bit Matrix Multiplication for
  Transformers at Scale. *Placeholder — completar DOI*.
- [Frantar et al., 2023] GPTQ: Accurate Post-Training Quantization for
  Generative Pre-trained Transformers. *Placeholder — completar DOI*.
- [Tseng et al., 2024] QuIP#: Even Better LLM Quantization with Hadamard
  Incoherence and Lattice Codebooks. *Placeholder — completar DOI*.
- [Konečný et al., 2016] Federated Learning: Strategies for Improving
  Communication Efficiency. *Placeholder — completar DOI*.
- Especificación GGUF / ggml. https://github.com/ggerganov/ggml
- [ZipNN, 2024] Compresión lossless de pesos de modelos. *Placeholder*.
