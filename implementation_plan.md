# Plan: Exponer Variables Internas del Modelo de Verhulst

## Objetivo

Modificar el modelo de Verhulst para que las variables internas necesarias para las 4 visualizaciones 2D queden disponibles como salidas opcionales, **sin romper el comportamiento existente**.

## Mapeo: Visualización → Variables Necesarias

| Visualización | Variables internas a exponer | Archivo fuente |
|---|---|---|
| **Mapa de calor coclear** | `SheraP[t]` (polo instantáneo por sección) | `cochlear_model2018.py` |
| **Panel de vesículas** | `qt[t]`, `wt[t]`, `available[t]` (por tipo de fibra) | `auditory_nerve2018.py` |
| **Corrientes de la IHC** | `mt[t]`, `Imet[t]`, `Ikf[t]`, `Iks[t]` | `inner_hair_cell2018.py` |
| **Balance exc/inh del tronco** | Componentes excitatoria e inhibitoria separadas de CN e IC | `ic_cn2018.py` |

## Estrategia de Retrocompatibilidad

Cada función modificada recibe un parámetro **`store_internals=False`** (valor por defecto). Cuando es `False`, la función retorna exactamente lo mismo que antes. Cuando es `True`, retorna una tupla con las variables adicionales.

**Archivos que llaman a estas funciones y que NO debemos romper:**
- [run_model2018.py](file:///c:/Users/camix/Desktop/metodologiaDeLaInvestigacion/regelaboForked/backend/src/simulation/Verhulst/src/run_model2018.py) — usa las funciones con el `return` original → no se toca, sigue funcionando porque `store_internals` es `False` por defecto
- [model2018.py](file:///c:/Users/camix/Desktop/metodologiaDeLaInvestigacion/regelaboForked/backend/src/simulation/Verhulst/src/model2018.py) — se actualiza para usar `store_internals=True` cuando el nuevo storeflag `'d'` esté presente
- [ExampleSimulation.py](file:///c:/Users/camix/Desktop/metodologiaDeLaInvestigacion/regelaboForked/backend/src/simulation/Verhulst/examples/ExampleSimulation.py), [ExampleAnalysis.py](file:///c:/Users/camix/Desktop/metodologiaDeLaInvestigacion/regelaboForked/backend/src/simulation/Verhulst/examples/ExampleAnalysis.py), [ParallelRAMSimulationsEFR.py](file:///c:/Users/camix/Desktop/metodologiaDeLaInvestigacion/regelaboForked/backend/src/simulation/Verhulst/examples/ParallelRAMSimulationsEFR.py) — llaman a `model2018()` con storeflags existentes → no se tocan, siguen funcionando

## Proposed Changes

### Componente 1: Célula Ciliada Interna

#### [MODIFY] [inner_hair_cell2018.py](file:///c:/Users/camix/Desktop/metodologiaDeLaInvestigacion/regelaboForked/backend/src/simulation/Verhulst/src/core/inner_hair_cell2018.py)

**Cambios:**

1. Agregar `store_internals=False` a la firma de `inner_hair_cell_potential()` (línea 130)
2. Crear 4 arrays de almacenamiento antes del bucle principal, solo si `store_internals=True` (después de línea 176)
3. Almacenar `mt`, `Imet`, `Ikf`, `Iks` en cada iteración del bucle principal (después de línea 239)
4. Cambiar el `return` (línea 241) para retornar tupla cuando `store_internals=True`

**Comportamiento:**
- `store_internals=False` → `return Vsol` (idéntico al original)
- `store_internals=True` → `return (Vsol, mt_sol, Imet_sol, Ikf_sol, Iks_sol)`

---

### Componente 2: Nervio Auditivo

#### [MODIFY] [auditory_nerve2018.py](file:///c:/Users/camix/Desktop/metodologiaDeLaInvestigacion/regelaboForked/backend/src/simulation/Verhulst/src/core/auditory_nerve2018.py)

**Cambios:**

1. Agregar `store_internals=False` a la firma de `auditory_nerve_fiber()` (línea 89)
2. Crear 3 arrays de almacenamiento antes del bucle principal, solo si `store_internals=True` (después de línea 213)
3. Almacenar `qt`, `wt`, `available` en cada iteración del bucle principal (después de línea 251)
4. Cambiar el `return` (línea 252) para retornar tupla cuando `store_internals=True`

**Comportamiento:**
- `store_internals=False` → `return solution` (idéntico al original)
- `store_internals=True` → `return (solution, qt_sol, wt_sol, avail_sol)`

---

### Componente 3: Núcleos del Tronco Encefálico

#### [MODIFY] [ic_cn2018.py](file:///c:/Users/camix/Desktop/metodologiaDeLaInvestigacion/regelaboForked/backend/src/simulation/Verhulst/src/core/ic_cn2018.py)

**Cambios en `cochlearNuclei()`:**

1. Agregar `store_internals=False` a la firma (línea 34)
2. Separar la línea 108 en dos cálculos (`cn_exc` y `cn_inh`) y luego restarlos
3. Cambiar el `return` (línea 109) para incluir las componentes cuando `store_internals=True`

**Comportamiento:**
- `store_internals=False` → `return cn, summedAN` (idéntico al original)
- `store_internals=True` → `return (cn, summedAN, cn_exc, cn_inh)`

**Cambios en `inferiorColliculus()`:**

1. Agregar `store_internals=False` a la firma (línea 111)
2. Separar la línea 164 en dos cálculos (`ic_exc` y `ic_inh`)
3. Cambiar el `return` (línea 165) para incluir las componentes cuando `store_internals=True`

**Comportamiento:**
- `store_internals=False` → `return ic` (idéntico al original)
- `store_internals=True` → `return (ic, ic_exc, ic_inh)`

---

### Componente 4: Mecánica Coclear

#### [MODIFY] [cochlear_model2018.py](file:///c:/Users/camix/Desktop/metodologiaDeLaInvestigacion/regelaboForked/backend/src/simulation/Verhulst/src/core/cochlear_model2018.py)

**Cambios en `solve()`:**

1. Agregar parámetro `store_internals=False` a la firma de `solve()` (línea 682)
2. Crear `self.SheraPsolution` array antes del bucle (después de línea 695), solo si `store_internals=True`
3. Almacenar `self.SheraP` en los probe points dentro del bucle principal (después de la sección que guarda Vsolution/Ysolution, ~línea 748)

> [!NOTE]
> Este cambio es diferente a los otros porque `SheraPsolution` es un atributo del objeto `cochlea_model`, no un valor retornado por una función. No hay riesgo de romper ningún `return`.

---

### Componente 5: Orquestador

#### [MODIFY] [model2018.py](file:///c:/Users/camix/Desktop/metodologiaDeLaInvestigacion/regelaboForked/backend/src/simulation/Verhulst/src/model2018.py)

**Cambios en `ModelOutput`:**

Agregar 12 campos nuevos al `__init__` (después de línea 62):

```python
# ---- Variables internas para visualizaciones detalladas (storeflag 'd') ----
self.sheraPt = None       # Polo de Shera instantáneo [tiempo_bm x secciones]
self.mt_ihc = None        # Probabilidad apertura canal MET [tiempo_bm x secciones]
self.Imet = None          # Corriente de transducción MET [tiempo_bm x secciones]
self.Ikf = None           # Corriente K⁺ rápida [tiempo_bm x secciones]
self.Iks = None           # Corriente K⁺ lenta [tiempo_bm x secciones]
self.qt_H = None          # Vesículas RRP - fibras HSR [tiempo_an x secciones]
self.wt_H = None          # Pool reserva - fibras HSR [tiempo_an x secciones]
self.avail_H = None       # Fibras no refractarias HSR [tiempo_an x secciones]
self.cn_exc = None        # Componente excitatoria CN [tiempo_an x secciones]
self.cn_inh = None        # Componente inhibitoria CN [tiempo_an x secciones]
self.ic_exc = None        # Componente excitatoria IC [tiempo_an x secciones]
self.ic_inh = None        # Componente inhibitoria IC [tiempo_an x secciones]
```

**Cambios en `solve_one_cochlea()`:**

Agregar un nuevo storeflag `'d'` (de "detailed/diagnóstico"). Cuando `'d'` está presente:

1. Llamar a `coch.solve(store_internals=True)` y guardar `coch.SheraPsolution`
2. Llamar a `ihc.inner_hair_cell_potential(..., store_internals=True)` y desempaquetar la tupla
3. Llamar a `anf.auditory_nerve_fiber(..., store_internals=True)` y desempaquetar (solo HSR para no triplicar memoria)
4. Llamar a `nuclei.cochlearNuclei(..., store_internals=True)` y desempaquetar
5. Llamar a `nuclei.inferiorColliculus(..., store_internals=True)` y desempaquetar

**Cuando `'d'` NO está presente:** todas las llamadas usan `store_internals=False` (default) → comportamiento idéntico al original.

---

## Verificación

### Prueba de retrocompatibilidad
Ejecutar `ExampleSimulation.py` sin cambios y verificar que produce los mismos resultados.

### Prueba de las variables nuevas
Ejecutar `model2018()` con `storeflag='evihmlbwd'` (agregando `'d'`) y verificar que los nuevos campos de `ModelOutput` no son `None`.

---

## Resumen de Impacto

| Aspecto | Detalle |
|---|---|
| Archivos modificados | 5 (4 módulos core + orquestador) |
| Archivos NO tocados | `run_model2018.py`, `ExampleSimulation.py`, `ExampleAnalysis.py`, `ParallelRAMSimulationsEFR.py` |
| Líneas nuevas estimadas | ~80 |
| Ecuaciones alteradas | 0 |
| Riesgo de romper código existente | **Nulo** (parámetro por defecto `False` preserva comportamiento original) |
| Impacto en rendimiento (sin flag 'd') | **Cero** (el código nuevo no se ejecuta) |
| Impacto en rendimiento (con flag 'd') | ~2-3× más memoria para almacenar arrays adicionales; tiempo de cómputo igual |
