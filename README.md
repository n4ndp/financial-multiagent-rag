# Sistema Multi-Agente ReAct para Auditoría de Inteligencia Financiera

**Curso:** Arquitecturas de Modelos de Lenguaje (LLMs)
**Proyecto Final — Opción 2:** Sistema Operativo de Agentes Cognitivos para Inteligencia Competitiva

---

## 1. Introducción

Los sistemas de generación aumentada por recuperación (RAG) de un solo paso presentan una
limitación conocida: la respuesta se genera y se entrega sin ningún mecanismo de verificación
posterior, por lo que una alucinación del modelo llega directamente al usuario final. Este
proyecto aborda esa limitación mediante un sistema multi-agente basado en el paradigma ReAct
(Reasoning and Acting), en el que la generación de una respuesta pasa por una etapa explícita
de auditoría antes de ser entregada.

El sistema está compuesto por tres agentes con roles diferenciados:

| Agente | Función |
|---|---|
| **Investigador** | Recupera evidencia de una base vectorial (ChromaDB) y de búsqueda web (DuckDuckGo), y redacta un borrador de respuesta citando sus fuentes |
| **Auditor de Hechos** | Contrasta el borrador contra el contexto recuperado; si encuentra afirmaciones sin respaldo, lo rechaza y devuelve una razón concreta |
| **Redactor** | Convierte el borrador ya aprobado en la respuesta final entregada al usuario |

El ciclo Investigador ↔ Auditor se repite hasta que el borrador es aprobado o se alcanza un
límite de reintentos. La orquestación de este ciclo se implementa como un grafo de estados con
**LangGraph**.

![Grafo de agentes: investigador, auditor y redactor](diagrama_grafo.png)

## 2. Arquitectura del sistema

| Componente | Elección | Justificación |
|---|---|---|
| Modelo orquestador | Mistral-7B-Instruct-v0.3, cuantizado en 4 bits | Corre en una sola GPU sin necesidad de ajuste fino; el mismo modelo cumple los tres roles cambiando únicamente el prompt de sistema |
| Orquestación de agentes | LangGraph (`StateGraph`) | Permite modelar el ciclo de rechazo/reintento entre Investigador y Auditor, algo que una cadena lineal de prompts no soporta |
| Base de conocimiento | ChromaDB + `all-MiniLM-L6-v2` | Índice vectorial liviano, sin competir por memoria de GPU con el modelo principal |
| Búsqueda web | DuckDuckGo Search | No requiere clave de API |
| Juez de evaluación | `gpt-4.1-mini` vía API de OpenAI | Modelo externo e independiente del que genera las respuestas, lo que evita el sesgo de auto-evaluación |

El sistema no entrena ni ajusta pesos en ningún momento: toda la capacidad proviene de la
orquestación de agentes sobre un modelo pre-entrenado.

## 3. Datos

Se utiliza el dataset `virattt/financial-qa-10K` (Hugging Face), compuesto por preguntas y
contextos extraídos de reportes 10-K de empresas reales. La base de conocimiento se construyó
indexando 3000 documentos del dataset; el conjunto de evaluación se extrajo de una porción
distinta, no incluida en el índice, para evitar fuga de información entre la construcción de la
base vectorial y la evaluación del sistema.

### Selección del conjunto de evaluación

El enunciado del proyecto exige evaluar el sistema sobre 50 consultas complejas. Como criterio
objetivo de complejidad se utilizó la longitud de la pregunta en número de palabras: dentro del
dataset, las preguntas más largas corresponden sistemáticamente a consultas que combinan varias
condiciones (múltiples cargos ejecutivos, rangos de fechas, transiciones entre roles), lo cual
exige mayor precisión de recuperación y de síntesis que una pregunta factual simple de una sola
variable. Se seleccionaron las 50 preguntas de mayor longitud dentro del conjunto no indexado.

## 4. Metodología de evaluación

Se compararon dos sistemas sobre el mismo conjunto de 50 consultas:

1. **Multi-Agente**: Investigador → Auditor → Redactor, con ciclo de reintento.
2. **Naive RAG**: recuperación seguida de una única generación, sin auditoría.

Cada respuesta se evaluó con un juez LLM externo (`gpt-4.1-mini`) en dos dimensiones, en una
escala de 0 a 100:

- **Faithfulness** — grado en que la respuesta está respaldada por el contexto recuperado.
- **Answer Relevance** — grado en que la respuesta aborda directamente la pregunta formulada.

## 5. Resultados

![Comparación de métricas entre Multi-Agente y Naive RAG](comparativa_metricas.png)

| Métrica | Multi-Agente | Naive RAG | Diferencia |
|---|---|---|---|
| Faithfulness | _completar con `tabla_comparativa.csv`_ | _completar_ | _completar_ |
| Answer Relevance | _completar_ | _completar_ | _completar_ |

Reintentos promedio del Auditor por consulta: _completar_.

## 6. Conclusiones

_A completar con los valores obtenidos tras la ejecución del notebook (la Sección 15 del
notebook contiene la misma estructura de conclusión con marcadores equivalentes):_

- Comparación de Faithfulness entre ambos sistemas y su interpretación.
- Tipo de errores detectados con mayor frecuencia por el Auditor.
- Limitación identificada: la métrica de Faithfulness evalúa consistencia con el contexto
  recuperado, no correspondencia con la realidad — si el fragmento correcto no está entre los
  documentos recuperados, el sistema puede aprobar una respuesta consistente pero incorrecta.
- Trabajo futuro: re-ranking de documentos recuperados, agente adicional de verificación
  numérica para cifras y fechas.

## 7. Ejecución

1. Abrir `proyecto_multiagente_reasoning.ipynb` en un entorno con GPU (mínimo ~8 GB de VRAM
   libre).
2. Ejecutar las celdas en orden. Se solicitará una clave de API de OpenAI, utilizada
   exclusivamente por el juez de evaluación.
3. La Sección 14 despliega una interfaz interactiva (Gradio) para la demostración del sistema.

Las imágenes y la tabla comparativa referenciadas en este documento (`diagrama_grafo.png`,
`comparativa_metricas.png`, `tabla_comparativa.csv`) se generan automáticamente al ejecutar el
notebook y quedan guardadas en la raíz del repositorio.

## 8. Estructura del repositorio

```
├── README.md
├── proyecto_multiagente_reasoning.ipynb
├── diagrama_grafo.png
├── comparativa_metricas.png
├── tabla_comparativa.csv
└── link_repositorio.txt
```
