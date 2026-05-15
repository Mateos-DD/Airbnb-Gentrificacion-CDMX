# Explicación técnica del pipeline de simulación ABM — CDMX v2

## 1. Archivos necesarios para las simulaciones

La simulación necesita **exactamente 5 archivos de código** y **1 archivo de datos**. Los archivos del clustering (`ageb_clusters_v2.csv`, `alcaldia_to_cells_v2.json`) **NO participan en la simulación** — solo se usan en el análisis posterior.

### 1.1 Inventario completo

| Archivo | Rol en la simulación | ¿Lo lee el ABM? |
|---|---|---|
| `income/income_clean_CDMX_v2.csv` | Distribución de ingresos (60 tramos ENIGH 2022) | **SÍ** — lo lee `model_cdmx.py` en `__init__` |
| `ABM_MeanField_Cells/model_cdmx.py` | Clase principal del modelo ABM | Código ejecutable |
| `ABM_MeanField_Cells/agent.py` | Clase del agente (lógica de movimiento) | Código ejecutable |
| `ABM_MeanField_Cells/utils.py` | Funciones auxiliares (percentiles, riqueza, placement) | Código ejecutable |
| `run_batch_cdmx.py` | Lanzador de simulaciones en paralelo | Código ejecutable |
| `batch_calculate_gs_ind_metrics.py` | Cálculo de métricas G_bin, G_net post-simulación | Código ejecutable (post-sim) |

**Archivos que NO participan en la simulación:**

| Archivo | Cuándo se usa |
|---|---|
| `income/ageb_clusters_v2.csv` | Análisis post-simulación: mapear cell_id → alcaldía en el heatmap de susceptibilidad |
| `income/alcaldia_to_cells_v2.json` | Análisis post-simulación: mismo mapeo en formato JSON |
| `income/kmeans_diagnostico_v2.csv` | Documentación de la tesis: métricas de calidad del clustering |
| `income/cdmx_cutoffs.json` | Referencia para documentación. Los cutoffs están hardcodeados en `model_cdmx.py` |

### 1.2 ¿Por qué el ABM no lee los archivos del clustering?

El ABM opera sobre un **grid abstracto** de `width × height` celdas. No tiene concepto de "alcaldía" ni de "AGEB". Cada celda del grid es simplemente una coordenada `(x, y)` donde los agentes viven y se mueven. La conexión entre celda `(1, 3)` del grid y "cluster 0 de Benito Juárez" existe solo en tu análisis posterior, cuando mapeas `cell_id → alcaldía` usando `alcaldia_to_cells_v2.json`.

Esto es idéntico al paper original: el ABM de Mauro et al. opera sobre un grid genérico; la interpretación geográfica (qué celda corresponde a qué barrio de la ciudad) es externa al modelo.

---

## 2. `income_clean_CDMX_v2.csv` — Cómo lo usa el modelo

Este CSV tiene 60 filas (tramos de ingreso) con 6 columnas:

```
bound_low      → Límite inferior del tramo (MXN/año)
bound_high     → Límite superior del tramo (MXN/año)
number         → Hogares ponderados en ese tramo
percent        → % del total de hogares en ese tramo (~1.67%)
percent_cumul  → % acumulado (1.67, 3.33, ..., 100.0)
average_amount → Ingreso promedio del tramo (MXN/año)
```

El modelo lo usa en dos momentos durante `__init__`:

**Momento 1 — Asignación de tramo:** `pick_random_row(df, model)` selecciona aleatoriamente un índice de tramo (0-59) usando `percent` como peso. Como cada tramo tiene ~1.67% de peso, la probabilidad es casi uniforme sobre los tramos — pero como los tramos representan percentiles, esto reproduce la distribución real de ingresos.

**Momento 2 — Asignación de ingreso:** `pick_random_amount(df, row, model)` genera un ingreso aleatorio uniforme entre `bound_low[row]` y `bound_high[row]`. Este ingreso se convierte en el atributo `wealth` del agente, que es permanente durante toda la simulación.

**Momento 3 — Clasificación L/M/H:** La función `_tipo(row)` clasifica al agente según su tramo:

```python
_ROW_L_MAX = 22   # tramos 0-22 → tipo "C" (L, bajo ingreso)
_ROW_H_MIN = 57   # tramos 57-59 → tipo "A" (H, alto ingreso)
                   # tramos 23-56 → tipo "B" (M, medio ingreso)
```

Estos cutoffs producen proporciones L=38.3%, M=56.7%, H=5.0%, que replican las del paper USA (38/57/5).

**Lo que NO cambia con el clustering:** Este CSV se construyó a partir de la ENIGH 2022 a nivel CDMX completa, no por alcaldía. No depende de qué AGEBs estén en qué cluster ni de cuántas celdas tenga el grid. Es el mismo archivo para grid 5×5, 4×6, o cualquier otro.

---

## 3. `model_cdmx.py` — Función por función

### 3.1 Equivalencia con el repo original

El repo original tiene `ABM_MeanField_Cells/model.py` con la clase `GentModel`. Nuestro `model_cdmx.py` tiene `GentModelCDMX`. Las diferencias son:

| Aspecto | Original (`GentModel`) | CDMX (`GentModelCDMX`) |
|---|---|---|
| Grid default | 9×9 (81 celdas) | 4×6 (24 celdas) |
| Income CSV | Hardcodeado internamente | Parámetro `income_csv` |
| Cutoffs L/M/H | Calibrados a datos USA | `_ROW_L_MAX=22`, `_ROW_H_MIN=57` |
| `coord_iter()` | `(content, (x,y))` en Mesa antigua | `(content, x, y)` en Mesa 1.2.1 |
| `super().__init__` | Sin seed | `super().__init__(seed)` |
| Arrays de riqueza | No inicializados (bug) | Inicializados como listas vacías |

**Toda la lógica de agentes, movimiento, percentiles, y métricas es idéntica.** Los cambios son de configuración (grid, cutoffs, CSV) y de compatibilidad (Mesa API).

### 3.2 `__init__()` — Inicialización del modelo

```python
def __init__(self, mode="improve", starting_deployment="centre_segr",
             width=4, height=6, num_agents=4096, h=20, p_g=0.1, ...):
```

**Paso 1 — Crear el grid:** `mesa.space.MultiGrid(width, height, torus=False)` crea una grilla rectangular donde cada celda `(x, y)` puede contener múltiples agentes. `torus=False` significa que los bordes no se conectan (no es un toroide). Esto es idéntico al original.

**Paso 2 — Calcular capacidad por celda:** `calculate_neighborhood_capacity(num_agents, width*height, occupancy_rate=0.75)` divide el total de agentes entre el número de celdas con un factor de ocupación del 75%. Para 4096 agentes en 24 celdas: `4096 / 0.75 / 24 ≈ 228` agentes máximo por celda. La capacidad es uniforme — todas las celdas tienen la misma capacidad sin importar su "tamaño real" (la heterogeneidad poblacional se captura por la distribución de ingresos, no por la capacidad).

**Paso 3 — Crear agentes:** El bucle `for i in range(num_agents)` crea 4096 agentes:

1. `pick_random_row()` → sortea un tramo del CSV ponderado por `percent`
2. `_tipo(row)` → clasifica en C/B/A según los cutoffs
3. `pick_random_amount()` → asigna ingreso aleatorio dentro del rango del tramo
4. `place_agent_2(model, tipo)` → coloca al agente en una celda según `starting_deployment`

**Paso 4 — Placement inicial (`place_agent_2` en `utils.py`):**
Con `starting_deployment="centre_segr"`, los agentes se colocan de forma segregada:

- **Tipo A (H):** En el centro exacto del grid y celdas adyacentes (bloque de 3×3). Para un grid 4×6, el centro es `(1, 2)` (calculado como `width//2 - 1, height//2 - 1` cuando las dimensiones son pares). Los agentes A se concentran aquí.
- **Tipo B (M):** En el interior del grid (no en los bordes), excluyendo el centro. Son las celdas con `0 < x < width-1` y `0 < y < height-1`.
- **Tipo C (L):** En los bordes del grid — las celdas donde `x=0`, `x=width-1`, `y=0`, o `y=height-1`.

Cada agente intenta colocarse en su zona preferente. Si está llena, recurre a un fallback: primero prueba otras celdas del mismo tipo, luego cualquier celda disponible. Esto implementa la segregación inicial descrita en el paper (Eq. 2 y sección de Métodos): la distribución centro-periferia donde los ricos están al centro y los pobres en la periferia.

**Paso 5 — Inicializar matrices de riqueza:** Se calculan 4 matrices de `width × height` (en nuestro caso, 4×6):

- `median_richness_matrix`: mediana de `wealth` de los agentes en cada celda
- `mean_richness_matrix`: media de `wealth` por celda
- `std_dev_richness_matrix`: desviación estándar por celda
- `total_richness_matrix`: suma total de riqueza por celda

Estas matrices se almacenan en listas (`_array`) que crecen en cada paso temporal. Los agentes A las usan para calcular tasas de crecimiento.

**Paso 6 — DataCollector:** Mesa registra en cada paso:

- Por agente: `pos`, `tipo`, `new_position`, `source`
- Por modelo: `median_richness_matrix`

Al final de la simulación, estos datos se exportan a CSV.

### 3.3 `step()` — Un paso temporal

```python
def step(self):
    # 1. Reset contadores
    self.unhappy_A = 0; self.unhappy_B = 0; self.unhappy_C = 0
    self.desire_to_move_A = 0; ...

    # 2. Todos los agentes ejecutan su lógica (orden aleatorio)
    self.schedule.step()

    # 3. Recalcular matrices de riqueza con las nuevas posiciones
    self.median_richness_matrix = calculate_median_richness_matrix(self)
    ...

    # 4. Recolectar datos
    self.datacollector.collect(self)

    # 5. Condición de terminación
    if self.unhappy_C == self.desire_to_move_C:
        self.running = False
```

La condición de terminación es crucial: la simulación para cuando **todos** los agentes C (L) que querían moverse pudieron hacerlo. Si `unhappy_C == desire_to_move_C`, significa que ningún C encontró una celda donde reubicarse, lo cual indica que el sistema alcanzó un estado estacionario (ya no hay espacio para desplazamiento). Esto es idéntico al paper.

### 3.4 `run_model(n)` — Ejecutar n pasos

```python
def run_model(self, n):
    for _ in range(n):
        if self.running: self.step()
        else: break
```

El modelo corre hasta `n` pasos (300 por default) o hasta que la condición de terminación se active. El paper reporta que la mayoría de simulaciones convergen entre los pasos 50-150.

---

## 4. `agent.py` — Lógica de movimiento por tipo

Este archivo es **idéntico al repo original** excepto por el fix de `coord_iter()` (Mesa API). La lógica de decisión es el corazón del ABM.

### 4.1 Estructura general del `step()` de cada agente

En cada paso temporal, cada agente:

1. Registra su posición actual como `source`
2. Calcula su percentil de riqueza dentro de su celda actual
3. Según su tipo (C/B/A), decide si quiere moverse y a dónde

### 4.2 Tipo C (L, bajo ingreso) — Ecuación 3 del paper

```python
my_prob = 1 - (self.my_percent ** (1/2))
```

**`my_percent`** es el percentil del agente dentro de su celda, normalizado a [0, 1]. Si un agente C tiene ingreso de $100k y los otros agentes en su celda tienen ingresos de $50k, $80k, $150k, $200k, su percentil es ~40% → `my_percent = 0.40`.

**La probabilidad de querer moverse** es `1 - √(percentil)`:
- Si está en el fondo de su celda (percentil ≈ 0): `P ≈ 1 - 0 = 1` → se quiere ir siempre
- Si está en la mitad (percentil ≈ 0.5): `P ≈ 1 - 0.71 = 0.29` → 29% de chance
- Si está arriba (percentil ≈ 1): `P ≈ 1 - 1 = 0` → nunca se va

Interpretación: un agente pobre en una celda donde es relativamente "rico" (alto percentil) se queda; un agente pobre en una celda donde es el más pobre se va. Esto modela el desplazamiento por presión económica.

**Destino en mode "improve":** Si decide moverse, calcula para cada celda disponible cuál sería su percentil allí (`calculate_arriving_percentile`). Luego pondera las celdas con `weight = √(percentil_destino)` — prefiere celdas donde sería relativamente alto. Esto es racional: un agente C se muda a donde su ingreso le alcanza más.

### 4.3 Tipo B (M, medio ingreso) — Ecuación 4 del paper

```python
my_prob = 4 * (self.my_percent - 0.5) ** 2
```

**La probabilidad de moverse** es una parábola centrada en 0.5:
- Si está cerca de la mediana (percentil ≈ 0.5): `P ≈ 4×0 = 0` → no se mueve
- Si está en un extremo (percentil ≈ 0 o ≈ 1): `P ≈ 4×0.25 = 1` → se mueve

Interpretación: un agente de ingreso medio está cómodo donde es "normal" (cerca de la mediana). Se incomoda si es demasiado rico para su celda (se aburre) o demasiado pobre (siente presión). Esto modela el comportamiento de clase media que busca homofilia socioeconómica.

**Destino:** Pondera con `weight = 1 - 4×(percentil - 0.5)²` — prefiere celdas donde estaría cerca del percentil 50. Exactamente lo opuesto a C.

### 4.4 Tipo A (H, alto ingreso) — Ecuación 5 del paper

```python
if step >= self.model.h and random.random() < self.model.p_g:
```

**Dos condiciones para activarse:**
1. Deben haber pasado al menos `h` pasos (20 por default) — los agentes ricos "observan el mercado" antes de actuar
2. Con probabilidad `p_g` (parámetro variable: 0.0, 0.01, 0.05, 0.1, 0.15)

**Destino en mode "improve":** Es fundamentalmente diferente a C y B. No mira percentiles sino **tasas de crecimiento de riqueza mediana** en los últimos `h` pasos:

```python
# Extraer las últimas h matrices de riqueza mediana
cells_last_h_list = self.model.median_richness_matrix_array[step-h : step]

# Para cada celda, construir la serie temporal de riqueza mediana
# y calcular la tasa de crecimiento promedio
cells_growth_rates = {
    cell: average_growth_rate(series)
    for cell, series in cells_median_richnesses_last_h.items()
}
```

El agente A se mueve a la celda con mayor tasa de crecimiento de riqueza mediana, siempre que esa tasa sea (a) positiva y (b) mayor que la de su celda actual. Esto modela la **gentrificación activa**: los ricos buscan vecindarios que se están "valorizando" — no los más ricos, sino los que crecen más rápido. Esto es el mecanismo de feedback positivo que produce gentrificación emergente.

### 4.5 Movimiento efectivo

```python
if self.new_position != (-1, -1):
    self.model.grid.move_agent(self, self.new_position)
```

Si el agente encontró destino, se mueve. Si no (todas las celdas llenas o sin opciones mejores), se queda y se registra como `unhappy`.

---

## 5. `utils.py` — Funciones auxiliares

### 5.1 Funciones que sí usa la simulación

**`calculate_neighborhood_capacity(num_agents, num_neighborhoods, occupancy_rate)`:**
Calcula la capacidad máxima por celda. `total_capacity = num_agents / 0.75`, dividido entre el número de celdas. Con 4096 agentes y 24 celdas: `4096 / 0.75 / 24 ≈ 228`. Cada celda puede tener como máximo 228 agentes. La función es idéntica al original.

**`eligibles_cells_with_space(model, eligibles)`:**
Filtra una lista de celdas para quedarse solo con las que tienen espacio disponible (menos agentes que su capacidad). Cada agente la llama antes de elegir destino. Idéntica al original.

**`calculate_arriving_percentile(model, agent, cells_with_space)`:**
Calcula qué percentil tendría el agente en cada celda candidata si se mudara ahí. Usa `scipy.stats.percentileofscore` sobre la lista de riquezas de cada celda. Retorna un diccionario `{(x,y): percentil}`. Idéntica al original.

**`calculate_median_richness_matrix(model)` / `_mean_` / `_std_dev_` / `_total_`:**
Recorre todas las celdas del grid y calcula la estadística correspondiente de las riquezas de los agentes en cada celda. Retorna una matriz numpy de `width × height`. Se llaman en cada `step()`. Idénticas al original.

**`pick_random_row(df, model)` y `pick_random_amount(df, row, model)`:**
Asignación de ingreso (explicadas en sección 2). Idénticas al original excepto que operan sobre nuestro CSV de 60 tramos vs el CSV del paper con tramos diferentes.

**`place_agent_2(model, tipo)`:**
Placement inicial con segregación centro/periferia. Es una versión mejorada de `place_agent()` del original que maneja correctamente la capacidad (intenta múltiples posiciones antes de declarar fallback). La lógica de zonas es la misma.

**`average_growth_rate(arr)`:**
Calcula la tasa de crecimiento promedio de una serie temporal: `Σ(arr[i] - arr[i-1]) / (len - 1)`. La usan los agentes A para evaluar celdas. Idéntica al original.

### 5.2 Funciones que solo usa el análisis posterior

`get_median_richness_df()`, `get_gini_richness_df()`, `fill_with_missing_pairs()`, `str_to_np_array()`: estas funciones parsean los CSVs de resultados para el notebook de análisis. No se ejecutan durante la simulación.

---

## 6. `run_batch_cdmx.py` — Lanzador de simulaciones

### 6.1 Equivalencia con el repo original

| Aspecto | Original (`run_batch.py`) | CDMX (`run_batch_cdmx.py`) |
|---|---|---|
| Clase del modelo | `GentModel` | `GentModelCDMX` |
| Grid default | 9×9 | 4×6 |
| Parámetro `income_csv` | No existe (hardcodeado) | `"income/income_clean_CDMX_v2.csv"` |
| `num_agents` | Variable (2⁷ a 2¹²) | Fijo por argumento (4096) |
| `p_g` | [0.01, 0.0, 0.05, 0.1, 0.15] | [0.0, 0.01, 0.05, 0.1, 0.15] |
| Paralelismo | `ProcessPoolExecutor()` sin límite | `--workers N` configurable |
| Skip existentes | Sí | Sí (misma lógica) |
| Output path | Idéntico | Idéntico |

La estructura de salida es exactamente la misma:
```
out/batch_results/improve/centre_segr-big/4096agents/exps/
  pg_0.0_h_20_rep_0_results_model.csv
  pg_0.0_h_20_rep_0_results_agents.csv
  pg_0.01_h_20_rep_0_results_model.csv
  ...
```

### 6.2 `run_one()` — Una réplica individual

```python
def run_one(rep, p_g, num_agents, mode, h, steps, width, height, seed, deployment):
```

1. **Construir path de salida:** `out/batch_results/{mode}/{deployment}-big/{n}agents/exps/pg_{p_g}_h_{h}_rep_{rep}_results_{model|agents}.csv`
2. **Skip si existe:** Si ambos CSVs ya existen, retorna sin hacer nada. Esto permite reanudar un batch interrumpido.
3. **Crear modelo:** Instancia `GentModelCDMX` con todos los parámetros.
4. **Correr:** `model.run_model(steps)` ejecuta hasta 300 pasos.
5. **Exportar:** `datacollector.get_model_vars_dataframe()` genera el CSV del modelo (1 fila por paso, con la matriz de riqueza mediana serializada como string). `get_agent_vars_dataframe()` genera el CSV de agentes (1 fila por agente por paso, con pos, tipo, source, new_position).

### 6.3 `main()` — Orquestador del batch

```python
pgs = [0.0, 0.01, 0.05, 0.1, 0.15]  # 5 valores de p_g
reps = list(range(rep_start, rep_end))  # 0..149
tasks = list(itertools.product(reps, pgs))  # 150 × 5 = 750 tareas
random.shuffle(tasks)  # Balancear carga entre workers
```

Las 750 tareas se distribuyen entre `N` workers usando `ProcessPoolExecutor`. El shuffle aleatorio evita que un worker reciba todas las tareas con p_g=0.15 (que tienden a tardar más porque los agentes A se mueven más).

**¿Por qué 150 réplicas?** El paper original usa 150 réplicas por configuración de parámetros para estabilidad estadística. Cada réplica usa la misma semilla (`seed=67`) pero diferente número de réplica (`rep`), lo cual genera diferentes secuencias aleatorias en el scheduler de Mesa.

---

## 7. `batch_calculate_gs_ind_metrics.py` — Métricas post-simulación

Este script se ejecuta **después** de las simulaciones. Lee los CSVs de agentes y calcula las métricas G_bin y G_net del paper.

### 7.1 ¿Qué calcula?

Para cada réplica y cada valor de delta (10, 15, 20), genera 3 CSVs:

- **`results_chi_hat.csv`** → G_bin: ¿La fracción de agentes M+H en la celda supera el umbral n*_MH = 0.62? Es una variable binaria (0/1) por celda por paso temporal, suavizada con una ventana de delta pasos.
- **`results_chi.csv`** → Fracción continua de M+H por celda (antes de binarizar).
- **`results_net_avg_prod.csv`** → G_net: Media geométrica de φ_out (salida neta de L) y φ_in (entrada neta de M+H). Captura el proceso activo de desplazamiento.

### 7.2 ¿Qué lee?

Solo los CSVs de agentes (`*_results_agents.csv`). Cada fila tiene `Step, tipo, pos, source, new_position`. De aquí extrae: la composición demográfica por celda (para G_bin), y los flujos de entrada/salida por tipo (para G_net).

### 7.3 Adaptación CDMX vs original

El código es el mismo del repo original con 3 fixes de compatibilidad:

1. `.values[i] = 0` → `.iat[i, col_idx] = 0` (fix numpy read-only en versiones ≥1.24)
2. `fillna(inplace=True)` → `= df.fillna(0)` (deprecación en pandas ≥2.0)
3. `applymap` → `map` (deprecación en pandas ≥2.0)

No necesita conocer el tamaño del grid — lo infiere de `df["pos"].max()`.

---

## 8. Flujo de datos entre archivos

```
income_clean_CDMX_v2.csv
         │
         ▼
model_cdmx.py ◄── agent.py (lógica de movimiento)
         │         ◄── utils.py (percentiles, placement, riqueza)
         │
         ▼
run_batch_cdmx.py
         │
         │  Genera por cada réplica:
         │    pg_X_h_20_rep_Y_results_model.csv   (matrices de riqueza por paso)
         │    pg_X_h_20_rep_Y_results_agents.csv   (posiciones y tipos por paso)
         │
         ▼
batch_calculate_gs_ind_metrics.py
         │
         │  Genera por cada réplica y delta:
         │    *_results_chi_hat.csv    (G_bin)
         │    *_results_chi.csv        (fracción M+H)
         │    *_results_net_avg_prod.csv (G_net)
         │
         ▼
batch_analysis.ipynb ◄── ageb_clusters_v2.csv (aquí SÍ se usa)
                      ◄── alcaldia_to_cells_v2.json
         │
         ▼
  Figuras y tablas para la tesis
```

---

## 9. Por qué el código produce resultados equivalentes al original

### 9.1 Misma lógica de agentes

`agent.py` es el mismo archivo del repo original con un único cambio: `coord_iter()` de Mesa 1.2.1 retorna `(content, x, y)` en lugar del `(content, (x, y))` de versiones anteriores. La línea `cell_list = [(x, y) for content, x, y in self.model.grid.coord_iter()]` produce exactamente la misma lista de celdas. Las ecuaciones de probabilidad de movimiento (secciones 4.2-4.4) son idénticas carácter por carácter al paper.

### 9.2 Misma estructura de modelo

`model_cdmx.py` difiere del original en configuración (grid, cutoffs, CSV), no en lógica. La secuencia de operaciones en `__init__` y `step()` es la misma: crear grid → asignar agentes → en cada paso actualizar matrices → recolectar datos → verificar terminación.

### 9.3 Misma distribución estadística

Los 60 tramos de ingreso CDMX reproducen la misma estructura que los tramos USA del paper: percentiles equiponderados con cola pesada. Las proporciones L/M/H son 38.3/56.7/5.0 vs 38/57/5 del paper (diferencia <0.5 puntos porcentuales).

### 9.4 Diferencias reales (para documentar en la tesis)

1. **Tamaño del grid:** 4×6 = 24 celdas vs 7×7 = 49 celdas (o 9×9 = 81 en suplementarios). Un grid más pequeño significa menos celdas para que los agentes migren, lo cual puede acelerar las transiciones de gentrificación (el desfase entre G_bin y G_net puede ser menor).

2. **Distribución de ingresos:** ENIGH 2022 vs datos USA. La cola pesada de CDMX (ratio 47x entre tramo más rico y más pobre) es comparable a la de USA, pero los valores absolutos son diferentes.

3. **Topología del grid:** `torus=False` en ambos casos, pero la proporción de celdas de borde vs interior cambia. En un grid 4×6, las 16 celdas de borde representan el 67% del total (vs 53% en un 7×7). Esto significa más espacio para C (L) inicialmente, lo cual puede retrasar la saturación de las celdas periféricas.

4. **`num_agents`:** Fijo en 4096 (2¹²), mientras que el paper también reporta resultados con 2⁷ a 2¹³. La densidad por celda es 4096/24 ≈ 170 agentes promedio vs 4096/49 ≈ 84 en el paper (para el grid de 49 celdas).

---

## 10. Qué mirar en los resultados de la validación

Cuando corras `validacion_pre_simulacion.py`, los checks críticos son:

1. **Proporciones L/M/H en t=1:** Deben estar dentro de ±2% de 38.3/56.7/5.0. Si no, hay un problema con los cutoffs o el CSV de ingresos.

2. **Shape de la matriz de riqueza:** Debe ser `(4, 6)`, no `(5, 5)`. Si sale `(5, 5)`, el `model_cdmx.py` no se actualizó.

3. **24 celdas consecutivas (0-23):** Si el check reporta un número diferente, hay inconsistencia entre `ageb_clusters_v2.csv` y el grid.

4. **Sin celdas vacías:** Todas las 24 celdas deben tener agentes. Si alguna está vacía, la capacidad es demasiado baja o el placement tiene problemas.

Si todos los checks pasan con ✅, lanza las simulaciones con confianza.
