# 🛰️ Guía de Trabajo para el Equipo: GEE, Google Colab y Entorno Local
**Iniciativa:** Brecha catastral y asentamientos informales en el Municipio del Distrito Central  
**Ecosistema:** Comunidad Urbana Copernicus Centroamérica  
**Proyecto GEE:** `Prueba 2-Distrito Central` | **Cloud Project ID:** `ee-mariomr16`  
**Equipo responsable:** Maritza Irias, Jonathán Rámirez, Mario Baide, Mario Matamoros  
**Rama activa de trabajo:** `mejora/actualizacion-metodologia`  

---

## 📌 1. Resumen Ejecutivo y Enlaces Clave

| Recurso | Tipo | Ubicación / Enlace |
| :--- | :--- | :--- |
| **Cuaderno Google Colab** | Notebook interactivo | [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/mariomr16-sudo/urbano-hn-distrito-central/blob/mejora/actualizacion-metodologia/notebooks/analisis_gee_distrito_central.ipynb) |
| **GEE Code Editor** | Script JavaScript en navegador | [https://code.earthengine.google.com/?project=ee-mariomr16](https://code.earthengine.google.com/?project=ee-mariomr16) |
| **Script Colab (código plano)** | Python puro | [`src/colab_runner.py`](../src/colab_runner.py) |
| **Generador de Figuras Locales** | Python / Matplotlib Forense | [`src/gee_analysis_visualizer.py`](../src/gee_analysis_visualizer.py) |
| **Figuras Analíticas (300 DPI)** | Gráficos PNG | [`docs/figuras/`](./figuras/) |
| **Portal Web Local** | Interfaz Forensic Dark | `http://localhost:8000/` |
| **Pull Request en GitHub** | Revisión de cambios | [Ver PR en GitHub](https://github.com/mariomr16-sudo/urbano-hn-distrito-central/pull/new/mejora/actualizacion-metodologia) |

---

## 💻 2. Flujo de Ejecución en Google Colab

### A. Abrir directamente desde GitHub
Cualquier integrante del equipo puede hacer clic en el botón **Open in Colab** arriba o ingresar a:
> **https://colab.research.google.com/github/mariomr16-sudo/urbano-hn-distrito-central/blob/mejora/actualizacion-metodologia/notebooks/analisis_gee_distrito_central.ipynb**

### B. Código Celda por Celda para Colab

#### Celda 1: Dependencias
```python
!pip install -q earthengine-api geemap matplotlib pandas numpy
import ee
import geemap
import matplotlib.pyplot as plt
import numpy as np
import pandas as pd
print("✓ Dependencias cargadas.")
```

#### Celda 2: Autenticación y Conexión con GEE
```python
ee.Authenticate()
PROJECT_ID = "ee-mariomr16"
ee.Initialize(project=PROJECT_ID)
print(f"✓ Conectado a Earth Engine con proyecto: {PROJECT_ID}")
```

#### Celda 3: Delimitación del Área de Estudio (Tegucigalpa y Comayagüela)
```python
aoi = ee.Geometry.Polygon([
    [[-87.275, 14.020],
     [-87.135, 14.020],
     [-87.135, 14.165],
     [-87.275, 14.165],
     [-87.275, 14.020]]
])
centroide = aoi.centroid().coordinates().getInfo()
print(f"✓ AOI configurada. Centroide: Lon {centroide[0]:.4f}, Lat {centroide[1]:.4f}")
```

#### Celda 4: Copernicus DEM y Laderas Críticas (Pendiente ≥ 30%)
```python
dem = ee.ImageCollection("COPERNICUS/DEM/GLO30").select("DEM").mosaic().clip(aoi)
slope_deg = ee.Terrain.slope(dem)
slope_pct = slope_deg.divide(180).multiply(np.pi).tan().multiply(100)
laderas_criticas = slope_pct.gte(30).selfMask()
print("✓ Laderas críticas procesadas.")
```

#### Celda 5: Compuesto Óptico Sentinel-2 y Cálculo de Índices
```python
s2 = (
    ee.ImageCollection("COPERNICUS/S2_SR_HARMONIZED")
    .filterBounds(aoi)
    .filterDate("2025-01-01", "2026-03-31")
    .filter(ee.Filter.lt("CLOUDY_PIXEL_PERCENTAGE", 20))
    .median()
    .clip(aoi)
)
# NDBI = (SWIR - NIR) / (SWIR + NIR)
ndbi = s2.normalizedDifference(["B11", "B8"]).rename("NDBI")
# NDVI = (NIR - RED) / (NIR + RED)
ndvi = s2.normalizedDifference(["B8", "B4"]).rename("NDVI")
print("✓ Índices NDBI y NDVI calculados.")
```

#### Celda 6: Dynamic World y Asentamientos en Riesgo
```python
dw = (
    ee.ImageCollection("GOOGLE/DYNAMICWORLD/V1")
    .filterBounds(aoi)
    .filterDate("2025-01-01", "2026-03-31")
    .select("built")
    .mean()
    .clip(aoi)
)
ciudad_construida = ndbi.gt(0.08).And(dw.gte(0.45)).selfMask()
asentamientos_riesgo = ciudad_construida.And(laderas_criticas).selfMask()
print("✓ Máscaras operativas listas.")
```

#### Celda 7: Mapa Interactivo con `geemap`
```python
Map = geemap.Map(center=[centroide[1], centroide[0]], zoom=12)
Map.add_basemap("HYBRID")

dem_vis = {"min": 900, "max": 1600, "palette": ["#0d0887", "#6a00a8", "#b12a90", "#fca636", "#f0f921"]}
slope_vis = {"min": 0, "max": 45, "palette": ["#10b981", "#f59e0b", "#ef4444"]}
ndbi_vis = {"min": -0.2, "max": 0.4, "palette": ["#1e3a8a", "#94a3b8", "#f59e0b", "#b91c1c"]}

Map.addLayer(dem, dem_vis, "Copernicus DEM (Elevación)", False)
Map.addLayer(slope_pct, slope_vis, "Pendiente (%)", False)
Map.addLayer(laderas_criticas, {"palette": ["#dc2626"]}, "Laderas Críticas (>= 30%)", True, opacity=0.7)
Map.addLayer(ndbi, ndbi_vis, "NDBI (Construido)", False)
Map.addLayer(ciudad_construida, {"palette": ["#fbbf24"]}, "Ciudad Construida Observada (2026)", True, opacity=0.8)
Map.addLayer(asentamientos_riesgo, {"palette": ["#ef4444"]}, "Asentamientos en Ladera Crítica", True, opacity=0.9)

Map.addLayerControl()
Map
```

#### Celda 8: Gráficos de Síntesis en Colab (Estética Forense)
```python
plt.style.use("dark_background")
fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(14, 5), dpi=150)
fig.patch.set_facecolor("#030305")
ax1.set_facecolor("#07070B")
ax2.set_facecolor("#07070B")

# Huella según pendiente
rangos = ["Plano (0-10%)", "Moderado (10-20%)", "Fuerte (20-30%)", "Crítico (>= 30%)"]
areas = [68.5, 42.1, 28.3, 19.8]
colores = ["#38BDF8", "#34D399", "#FFC800", "#EF4444"]
bars = ax1.bar(rangos, areas, color=colores, edgecolor="#1E222B", width=0.55)
ax1.set_title("HUELLA CONSTRUIDA POR RANGO DE PENDIENTE (KM²)", fontsize=11, color="#FFFFFF", pad=10, loc="left")
ax1.set_ylabel("Superficie Estimada (km²)", color="#CBD5E1")
ax1.grid(color="#181A24", linestyle="--", linewidth=0.5, axis="y")
for b in bars:
    h = b.get_height()
    ax1.text(b.get_x() + b.get_width()/2, h + 1, f"{h:.1f}", ha="center", color="#FFFFFF", fontsize=9, fontweight="bold")

# Distribución de brecha
ax2.pie([129.5, 48.5], labels=["Catastro Vigente (AMDC)", "Brecha / Rezago"], autopct="%1.1f%%",
        colors=["#64748B", "#FFC800"], startangle=140, explode=[0, 0.08],
        textprops={"color": "#FFFFFF", "fontsize": 10}, wedgeprops={"edgecolor": "#030305", "linewidth": 2})
ax2.set_title("PROPORCIÓN DE BRECHA CATASTRAL ESTIMADA", fontsize=11, color="#FFFFFF", pad=10, loc="left")

plt.tight_layout()
plt.show()
```

---

## 🎨 3. Generador de Figuras de Alta Resolución en Local

Para generar las 4 láminas analíticas listas para informes y presentaciones a `300 DPI`, ejecuta en tu terminal:

```bash
python3 src/gee_analysis_visualizer.py
```

### Contenido de las figuras generadas en `docs/figuras/`:
1. **`figura_1_serie_temporal_brecha.png`**: Evolución multitemporal (2016–2026) entre la ciudad construida satelital vs. catastro municipal registrado.
2. **`figura_2_matriz_tipologias_asentamientos.png`**: Matriz de tipologías A a E cruzando densidad de edificaciones (*Open Buildings*) frente al umbral crítico de pendiente (**≥ 30%** en *Copernicus DEM*).
3. **`figura_3_perfil_espectral_indices.png`**: Análisis de separabilidad espectral de índices (*NDBI*, *BSI*, *NDVI*, *Dynamic World*).
4. **`figura_4_brecha_por_colonia.png`**: Ranking analítico de colonias prioritarias (Nueva Capital, Flor del Campo, Los Pinos, Canaán, etc.).

---

## 🛠️ 4. Historial de Prompts y Comandos Ejecutados en la Sesión

### Paso 1: Verificación e instalación de herramientas
* Verificación de `git`, `python3`, `pip3`, `jsonschema` y GitHub CLI (`gh`).
* Instalación de `jsonschema` (4.25.1).
* Descarga e instalación directa de `gh` (v2.103.3 para macOS ARM64).
* Autenticación con GitHub por navegador (`gh auth login --web -p https`).

### Paso 2: Configuración del repositorio local y ramas
* Fork y clonación del repositorio institucional:
  ```bash
  gh repo fork copernicuscentroamerica/urbano-hn-distrito-central --clone=true
  ```
* Verificación de remotos: `origin` (personal) y `upstream` (canónico).
* Sincronización con `upstream/main` y creación de rama:
  ```bash
  git checkout -b mejora/actualizacion-metodologia
  ```

### Paso 3: Ediciones editoriales y gobernanza
* **[`content/metodologia.md`](../content/metodologia.md)**: Se añadió la precisión metodológica del cruce espacial (tolerancia de 3 a 5 m, distinción entre vacío de datos vs. rezago catastral, y filtrado multitemporal).
* **[`project.json`](../project.json)**: Se actualizó `governance.content_updated_at` a la fecha ISO `2026-11-10`.
* **[`scripts/build_site.py`](../scripts/build_site.py)**: Se corrigió la compatibilidad de `Path.write_text` para Python 3.9+ y se insertaron los nombres del equipo.

### Paso 4: Rediseño visual (Forensic Architecture Dark Minimalist)
* **[`web/assets/css/styles.css`](../web/assets/css/styles.css)**:
  * Fondo negro mate espacial (`#020204` / `#030305`) con retícula técnica de 38px.
  * Animación de iluminación ambiental intermitente en ámbar y cian (`light-intermittent-amber`, `light-intermittent-cyan`).
  * Baliza de estado parpadeante (`pulse-beacon`) y barrido de radar lineal.
  * Inclusión de nombres de integrantes en cabecera y Hero: **Maritza Irias, Jonathán Rámirez, Mario Baide y Mario Matamoros**.

### Paso 5: Despliegue del servidor local y GitHub
* Servidor web en segundo plano:
  ```bash
  python3 -m http.server 8000 --directory site
  ```
* Confirmación y subida de commits a GitHub:
  ```bash
  git push -u origin mejora/actualizacion-metodologia
  ```

---

## 🔒 5. Nota Metodológica sobre Fuentes de Datos

* **Datos satelitales:** Públicos, abiertos y procesados en tiempo real mediante Copernicus (*Sentinel-2 L2A*, *Copernicus DEM 30m*) y Google (*Dynamic World*, *Open Buildings*).
* **Datos catastrales:** La información vectorial a nivel de lote/predio es de custodia institucional protegida (**AMDC** e **Instituto de la Propiedad**). Los datos utilizados en las figuras analíticas corresponden a una **calibración metodológica preliminar (mockup representativo)** basada en la ficha oficial `S24` y el censo de TECHO 2018.
* **Integración de datos reales del catastro:**
  Cuando la AMDC entregue el archivo vectorial autorizado, se sube a:
  `projects/ee-mariomr16/assets/catastro_amdc`
  Y se carga en Colab con:
  ```python
  catastro_real = ee.FeatureCollection("projects/ee-mariomr16/assets/catastro_amdc")
  ```
