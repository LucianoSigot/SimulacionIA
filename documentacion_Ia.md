# 🧠 Arquitectura y Pipeline de Entrenamiento - Ajedrito IA

Este documento detalla la estructura, flujo de datos y decisiones técnicas del nuevo pipeline de Machine Learning independiente. A diferencia de la arquitectura acoplada al servidor, este pipeline divide el trabajo en fases claras de **Ingesta, Transformación (ETL) y Entrenamiento**, permitiendo iterar rápidamente y entrenar modelos separados por ELO.

---

## 🏗️ Flujo de Ejecución (Pipeline)

El orden estricto en el que los scripts deben ejecutarse (y cómo fluyen los datos) es el siguiente:

1. `pgn_a_csv.py` ➔ genera `datasetchess.csv`
2. `procesador.py` (usa `extractor.py`) ➔ genera archivos `.npz` (ej. `matrices_facil.npz`)
3. `entrenador.py` ➔ genera modelos `.joblib` (ej. `modelo_ia_facil.joblib`)

---

## 📂 Detalle de Archivos y Responsabilidades

### 1. `pgn_a_csv.py` (Ingesta y Extracción Base)
* **Objetivo:** Leer el archivo gigante `.pgn` descargado de Lichess y convertirlo a un formato tabular manejable.
* **Transformaciones y Por Qué:** 
  * *Lectura en Streaming:* Se lee partida por partida para evitar cargar todo en la RAM y que la computadora colapse.
  * *Filtro de ELO Dinámico:* Se extrae el ELO del jugador activo en el momento exacto del turno (Blancas o Negras). **¿Por qué?** Porque necesitamos que la IA asocie la jugada a la habilidad real de quien tomó la decisión, no a un promedio.
  * *Deduplicación:* Se eliminan tableros repetidos (ej. posiciones iniciales) para evitar sobreajuste en las aperturas.
* **Tipo de Dato de Salida:** Archivo `.csv` con columnas `fen` (string), `move` (string UCI), y `elo` (entero).

### 2. `extractor.py` (Vectorización Matemática)
* **Objetivo:** Contiene la lógica matemática pura para traducir un tablero de ajedrez a números que un algoritmo entienda. *Nota: Este script no se ejecuta directamente, es importado.*
* **Transformaciones y Por Qué:**
  * *One-Hot Encoding (773 Dimensiones):* En lugar de usar un solo tablero donde las piezas suman o restan valores (lo cual confunde a los modelos matemáticos porque un Caballo (3) no es matemáticamente "el triple" que un Peón (1)), se crean **12 mapas binarios** (uno por cada tipo de pieza y color).
* **Tipo de Dato de Salida:** `numpy.ndarray` homogéneo de tipo `float32`.

### 3. `procesador.py` (ETL y Particionamiento)
* **Objetivo:** Segmentar el dataset general en dificultades (Fácil, Medio, Difícil) y aplicar la vectorización matemática masiva.
* **Transformaciones y Por Qué:**
  * *Filtrado por ELO:* Usa Pandas para dividir el dataset según umbrales (ej. < 1200 para Fácil).
  * *Conversión a Formato Binario:* Transforma los FEN usando el `extractor.py` y guarda todo en archivos `.npz`. **¿Por qué?** Guardar los datos ya normalizados en tensores homogéneos binarios de NumPy comprime drásticamente el peso y permite que el script de entrenamiento cargue los datos a la RAM en milisegundos, sin tener que recalcular el tablero en cada prueba.
* **Tipo de Dato de Salida:** Archivos binarios comprimidos `.npz` que contienen internamente matrices `X` (features) e `y` (etiquetas).

### 4. `entrenador.py` (Machine Learning)
* **Objetivo:** Entrenar el algoritmo `RandomForestClassifier` para cada nivel de dificultad y exportar los "cerebros" listos para producción.
* **Transformaciones y Por Qué:**
  * *Train/Test Split:* Divide la data (85% entrenamiento, 15% evaluación) para medir la precisión en posiciones que la IA nunca vio.
  * *Serialización:* Empaqueta el modelo estadístico resultante usando `joblib`.
* **Tipo de Dato de Salida:** Archivos `.joblib` (Artefactos serializados del modelo Scikit-learn).

---

## 🚀 Pendientes y Próximos Pasos (Next Steps)

Para que todo el sistema esté completo y conectado, falta abordar los siguientes puntos con el equipo:

1. **Aumento de Volumen de Datos (Data Augmentation):**
   * *Problema:* El *accuracy_score* actual es bajo porque el modelo tiene pocas posiciones para aprender (ej. 1,500 en nivel fácil). 
   * *Acción:* Ejecutar el `pgn_a_csv.py` sobre un archivo de Lichess mucho más grande (procesar cientos de miles de jugadas) para subir la precisión.
2. **Desarrollar el "Selector" en la API (Backend):**
   * *Acción:* Modificar la ruta `/predict` en el servidor de FastAPI. Debe recibir por parámetro si el usuario juega contra Stockfish o contra la IA Propia.
   * Si es IA Propia: Cargar el `.joblib` correspondiente a la dificultad seleccionada.
3. **Integración de Motor Stockfish:**
   * *Acción:* Escribir el script que se comunique con el ejecutable de Stockfish local usando la librería `chess.engine` pasándole el límite de ELO correspondiente.
4. **Validación de Filtro Legal FIDE:**
   * *Acción:* Asegurarse de que el backend intercepte la predicción probabilística de los archivos `.joblib` y verifique que la jugada existe dentro de `board.legal_moves` de `python-chess` para evitar jugadas alucinadas/ilegales.