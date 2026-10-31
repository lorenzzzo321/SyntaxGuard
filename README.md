# SyntaxGuard

Detector de spam basado en análisis sintáctico-estructural (DOM, POS, dependencias, URLs).

## Instalación (Python 3.10 o 3.11)

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python -m spacy download en_core_web_sm
```

Los recursos de NLTK se descargan automáticamente la primera vez.

## Dataset

Coloca el CSV de Kaggle en `data/dataset.csv` (por ejemplo "SMS Spam Collection": columnas `v1`/`v2`).
Se detectan automáticamente las columnas de texto y etiqueta. Si existe una columna `html`, se usa también.

## Entrenamiento

```bash
python scripts/train.py
# opcional: python scripts/train.py --max-muestras 2000
```

Genera `models/clasificador.pkl` y `models/metricas.json` (métricas reales sobre validación y test).

## Prueba de humo y ejecución

```bash
python scripts/smoke_test.py
uvicorn app.main:app --reload --port 8000
```

Documentación interactiva: http://localhost:8000/docs

## Enfoque de rasgos

- **DOM + zss:** el árbol DOM del HTML se compara contra un árbol de referencia de spam
  (tablas anidadas, enlaces con imagen, bloques centrados). Se usa la distancia de edición normalizada.
- **POS (NLTK):** proporciones de sustantivos, verbos, pronombres, números, etc.
- **Dependencias (spaCy):** profundidad del árbol y proporción de oraciones imperativas.
- **URLs:** longitud, IP, acortadores, caracteres especiales y similitud difusa (RapidFuzz) con dominios legítimos.
- **Texto:** mayúsculas, exclamaciones, palabras de lista de spam, dígitos y símbolos de moneda.

## Limitaciones

- Los datasets típicos de Kaggle (SMS) no traen HTML. En ese caso los rasgos `dom_*` valen lo mismo
  en todo el entrenamiento y el modelo no los usa. Se calculan y se muestran en el análisis, pero solo
  influyen en la predicción si el dataset incluye una columna `html`.
- El modelo en inglés (`en_core_web_sm`) está pensado para texto en inglés.

---

## Interfaz web (Fase 2: cascarón y sistema de diseño)

La interfaz está hecha con **HTML, CSS y JavaScript puros** (sin frameworks ni compilación) y la sirve el
mismo servidor FastAPI. No requiere instalar nada adicional: basta con arrancar la API.

```bash
uvicorn app.main:app --reload --port 8000
```

| Ruta | Qué es |
|---|---|
| http://localhost:8000/ | Interfaz web |
| http://localhost:8000/docs | Documentación interactiva de la API (FastAPI) |
| `GET /salud`, `GET /metricas`, `POST /predecir` | Endpoints originales, sin cambios |
| `/static/...` | Archivos CSS/JS de la interfaz (carpeta `frontend/`) |

### Estado de la interfaz

| Fase | Contenido | Estado |
|---|---|---|
| 2 | Menú lateral, barra superior con estado real de la API, indicador de pasos, tokens de diseño, router por hash y cliente de API | Terminada |
| 3 | Carga de correo (pegar / subir `.eml`) y pantalla de análisis | Pendiente |
| 4 | Resultado y evidencias | Pendiente |
| 5 | Historial, métricas, retroalimentación y Acerca de | Pendiente |
| 6 | Integración final, pruebas y documentación | Pendiente |

Hoy las secciones *Historial*, *Métricas*, *Retroalimentación* y *Acerca de* muestran una tarjeta
"En construcción". La sección *Análisis* incluye una comprobación temporal de conexión con la API
(`/salud`, `/metricas` y `/predecir`) que se sustituirá en la Fase 3.
Las métricas que mostrará la interfaz serán siempre las reales de `models/metricas.json`.

### Estructura de `frontend/`

```text
frontend/
├── index.html
├── css/   tokens.css · layout.css · componentes.css
└── js/    app.js · router.js · api.js · state.js · ui.js · iconos.js
          └── vistas/  analizar.js · historial.js · metricas.js · retroalimentacion.js · acerca.js
```

`app/main.py` solo añade la ruta `/` y el montaje de `/static` (y solo si existe la carpeta `frontend/`);
los endpoints originales no se modificaron.

### Ejecutar en Windows con VS Code (PowerShell)

```powershell
cd SyntaxGuard                                  # la carpeta que contiene app\, frontend\, models\ ...
.\.venv\Scripts\Activate.ps1                    # si PowerShell lo bloquea: Set-ExecutionPolicy -Scope Process Bypass
python scripts\smoke_test.py                    # prueba original (backend)
python scripts\smoke_test_frontend.py           # prueba de la interfaz y de que los endpoints originales siguen iguales
uvicorn app.main:app --reload --port 8000
```

Después abre http://localhost:8000/ . Arriba a la derecha debe aparecer **"Modelo cargado"** y, en la
sección *Análisis*, las tres filas deben mostrar **"Conectado"**.

### Pruebas de la Fase 2

- `scripts/smoke_test.py` (original, sin cambios) y `scripts/smoke_test_frontend.py` (nuevo).
- `docs/fase2/` contiene el reporte de pruebas, las capturas (escritorio y móvil) y los scripts usados con
  navegador real. Esos scripts necesitan Playwright (`pip install playwright` y `playwright install chromium`),
  que **no** forma parte de `requirements.txt` porque solo se usa para probar.
