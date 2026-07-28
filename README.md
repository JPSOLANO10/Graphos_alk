# Análisis de grafos del IVR – Alkosto

Esta carpeta contiene el código mínimo necesario para reconstruir el análisis de grafos
de navegación del IVR de Alkosto (quién pasa por qué opción, cuántas veces, y a dónde
llega). Se extrajo desde `Alkosto/codigo/` porque son los dos únicos notebooks de los
que dependen los grafos (`.gv` / `.gv.pdf`) que ya existen en esa carpeta.

## Resumen del pipeline

```
base/vw_interaxa_detalle_ivr_202403121110.csv   (crudo, export de Interaxa)
base/TrazaOpcionesIVR Alkosto_tipos.xlsx        (maestro de trazas -> Segmentación / Nodo 1..10)
                    │
                    ▼
        01_Alk_final.ipynb   (limpieza + segmentación + construcción de aristas origen→destino)
                    │
                    ├──> df_transformado.xlsx              (traza completa, granularidad "un paso por fila")
                    ├──> base/base_final_inicio_ivr.xlsx    (base ancha + métricas de recontacto)
                    └──> bases filtradas por escenario:
                         base_menu.xlsx
                         base_final_menu.xlsx
                         base_transferencia_hasta_4.xlsx / _mas_4.xlsx
                         base_producto_hasta_5.xlsx / _mas_5.xlsx
                         base_entrega_hasta_5.xlsx / _mas_5.xlsx
                         base_garantia_menor_hasta_6.xlsx / _mas_6.xlsx
                         base_grafos_estado_de_entrega.xlsx
                    │
                    ▼
        02_Grafos.ipynb   (agrupa origen→destino, cuenta frecuencia, dibuja el grafo)
                    │
                    ▼
        *.gv  y  *.gv.pdf   (grafo Graphviz, color por frecuencia de la arista)
```

Cada "escenario" (menú, transferencia Alkosto-Tuya, existencia de producto, estado de
entrega, garantías...) es simplemente un filtro distinto de `df_transformado` aplicado
en `01_Alk_final.ipynb`, exportado a su propio Excel, que luego se le pasa a
`02_Grafos.ipynb` cambiando la ruta de lectura.

## Notebooks

### `01_Alk_final.ipynb` (original: `codigo/Alk_final.ipynb`)

**Qué hace:** es la versión más reciente (jun-2024) y consolidada de la limpieza de la
traza del IVR. Reemplaza a `Alkosto_limpieza.ipynb` y sus copias, que eran borradores
previos del mismo proceso.

Pasos clave:
1. Lee el CSV crudo `vw_interaxa_detalle_ivr_202403121110.csv` (una fila por
   interacción, con la columna `opcionesnavegaciontrazaopciones` conteniendo toda la
   traza separada por `|`).
2. Explota (`stack`) esa columna en una fila por paso de navegación, y separa cada paso
   en `op_num`, `op_text`, `op_tiempo` (separados por `;`).
3. Numera los pasos de cada `idtransaccion` en la columna `bloque`.
4. Cruza cada `op_text` contra el maestro `TrazaOpcionesIVR Alkosto_tipos.xlsx`
   (hoja `MaestroTrazasOpciones`) para asignar `Segmentacion` (Navegación,
   ValidacionesFlujo, Paso asesor, Resolutiva, no identificado) y un `Nodo` (agrupación
   de más alto nivel, columnas `Nodo 1`...`Nodo 10` del maestro).
5. Marca manualmente como `Paso asesor` cualquier opción cuyo texto contenga "paso"
   (porque el maestro no las tiene mapeadas).
6. Construye la **arista origen→destino**: crea `op_text_final` desplazando `op_text`
   una posición hacia abajo dentro de cada `idtransaccion` (`shift(-1)`); la última fila
   de cada transacción se marca como `final`. Esta es la transformación central: convierte
   una traza secuencial en pares (origen, destino) que es lo que consume el grafo.
7. Exporta `df_transformado.xlsx` — esta es la base completa a nivel de paso, de la que
   se derivan todos los filtros posteriores.
8. Calcula métricas de recontacto (llamadas repetidas por `ani`, tiempo entre contactos)
   y exporta `base/base_final_inicio_ivr.xlsx`.
9. Para cada escenario de negocio (menú principal, transferencia Alkosto-Tuya,
   existencia de producto, estado de entrega, garantías por periodo) filtra
   `df_transformado` por `traza_final` (a qué opción llegó el usuario) y por cantidad de
   pasos (típico vs. atípico, ej. "hasta 4 opciones" vs "más de 4"), y exporta cada
   subconjunto a su propio `.xlsx` en `base/`. Estos son los que luego lee
   `02_Grafos.ipynb`.

**Inputs que debe tener disponibles:**
- `Alkosto/base/vw_interaxa_detalle_ivr_202403121110.csv`
- `Alkosto/base/TrazaOpcionesIVR Alkosto_tipos.xlsx` (hoja `MaestroTrazasOpciones`,
  columnas `TrazaOpcion`, `Segmentacion`, `Nodo 1`..`Nodo 10`)
- `Alkosto/base/base_final.xlsx` (se usa para saber `traza_final` de cada
  `idtransaccion` al armar los filtros por escenario — lo genera
  `Alkosto_rellamadas.ipynb`, que no se incluyó aquí porque no es necesario para el
  grafo salvo por este archivo intermedio)

**Librerías:** `pandas`, `numpy`, `psycopg2` (import sin uso activo en las celdas
revisadas), `unidecode`, `networkx` (importado pero no usado directamente aquí — se
deja porque el notebook original lo trae).

### `02_Grafos.ipynb` (original: `codigo/Grafos.ipynb`)

**Qué hace:** toma una base ya filtrada (salida del notebook anterior) y dibuja el
grafo dirigido de navegación.

Pasos clave:
1. Lee un `base_*.xlsx` (por defecto `base_menu.xlsx`; hay una segunda celda de ejemplo
   que lee `base_transferencia_mas_4.xlsx` desde `Alkosto/base/`).
2. Agrupa por (`op_text`, `op_text_final`) y cuenta transacciones (`idtransaccion`) para
   obtener la **frecuencia de cada arista**.
3. Define un colormap (`YlOrRd` de matplotlib) normalizado en escala logarítmica sobre
   esa frecuencia, para que los nodos/aristas más transitados se vean más "calientes".
4. Construye el grafo con `graphviz.Digraph`: una arista por cada par (origen, destino)
   con la frecuencia como etiqueta, y colorea cada nodo destino según el volumen total
   que recibe.
5. `u.view()` renderiza y abre el PDF (`<filename>.gv` + `<filename>.gv.pdf`).

Para generar el grafo de otro escenario (garantías, estado de entrega, existencia de
producto, etc.) solo hay que cambiar la ruta del `pd.read_excel(...)` inicial y el
`filename=` del `graphviz.Digraph` por el `base_*.xlsx` correspondiente generado en el
paso anterior.

**Inputs:** cualquiera de los `base_*.xlsx` filtrados que produce `01_Alk_final.ipynb`.

**Librerías:** `graphviz` (requiere también el binario de Graphviz instalado en el
sistema y en el PATH, no solo el paquete de Python), `pandas`, `tqdm`, `matplotlib`,
`numpy`, `ipywidgets` (importa `interact`/`Dropdown` pero no se usan en las celdas
revisadas — probablemente para una versión interactiva no terminada).

## Instalación de Graphviz

`02_Grafos.ipynb` usa el paquete de Python `graphviz`, que es solo un envoltorio: para
que `u.view()` funcione y genere el `.pdf` hace falta además el **binario de Graphviz**
instalado en el sistema operativo y visible en el `PATH`. Si falta el binario, el
notebook falla con `graphviz.backend.execute.ExecutableNotFound`.

### 1. Binario de Graphviz (Windows)

Opción recomendada, con [Chocolatey](https://chocolatey.org/install) (PowerShell como
administrador):

```powershell
choco install graphviz -y
```

Alternativa sin Chocolatey: descargar el instalador desde
[graphviz.org/download](https://graphviz.org/download/) (sección Windows), instalarlo, y
durante la instalación marcar la opción **"Add Graphviz to the system PATH"** (para
todos los usuarios o solo el usuario actual). Si no aparece esa opción, agregar
manualmente la carpeta `bin` de la instalación (típicamente
`C:\Program Files\Graphviz\bin`) a la variable de entorno `PATH`.

Verificar que quedó bien instalado abriendo una terminal **nueva** y ejecutando:

```powershell
dot -V
```

Debe imprimir algo como `dot - graphviz version X.X.X`. Si da "no se reconoce como
comando", el PATH no se actualizó — cerrar y volver a abrir la terminal/VS Code, o
agregar la ruta manualmente.

### 2. Paquete de Python

```bash
pip install graphviz pandas numpy matplotlib tqdm ipywidgets networkx unidecode psycopg2-binary
```

(`graphviz` es el envoltorio de Python; `pandas`, `numpy`, `matplotlib` y `tqdm` los usa
`02_Grafos.ipynb` para agrupar datos y colorear el grafo; `unidecode`, `networkx` y
`psycopg2-binary` los importa `01_Alk_final.ipynb`.)

## Orden de ejecución

1. Verificar que existan los insumos en `Alkosto/base/` (CSV crudo, maestro de trazas,
   `base_final.xlsx`).
2. Ejecutar `01_Alk_final.ipynb` completo → genera `df_transformado.xlsx` y todos los
   `base_*.xlsx` por escenario.
3. Ejecutar `02_Grafos.ipynb`, ajustando la celda de lectura al escenario que se quiera
   graficar → genera el `.gv` / `.gv.pdf`.

## No incluidos (y por qué)

- `Alkosto_limpieza.ipynb` y sus copias: versiones previas/borrador, superadas por
  `Alk_final.ipynb`.
- `Alkosto_rellamadas.ipynb` (y su copia): análisis de recontactos/rellamadas, un
  entregable aparte. Solo es relevante aquí porque genera `base_final.xlsx`, que
  `01_Alk_final.ipynb` usa para saber la `traza_final` de cada transacción.
