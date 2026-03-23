[README.md](https://github.com/user-attachments/files/26160405/README.md)
# Dashboard Gerencial — Casa Luis Chemes S.A.

Panel gerencial mobile-first para gerentes de sucursal.
Accesible desde celular y PC, sin instalación, sin VPN.

---

## Arquitectura

```
SQL Server (10.10.10.99)
    │
    ▼ PowerShell (Export-DashboardData.ps1) — cada 1 hora
    │
    ▼ CSVs en Google Drive (carpeta "Dashboard_Chemes_Export")
    │
    ▼ Apps Script Web App (DashboardChemes.gs) — publica JSON
    │
    ▼ index.html en GitHub Pages — consume el JSON via fetch()
```

---

## Estructura de archivos

```
chemes-dashboard/
├── powershell/
│   ├── Export-DashboardData.ps1     ← Script principal de exportación
│   └── Ejecutar_Dashboard.bat       ← Lanzador para Task Scheduler
├── appscript/
│   ├── DashboardChemes.gs           ← Apps Script (Web App backend)
│   └── objetivos_template.csv       ← Plantilla CSV de objetivos
└── web/
    └── index.html                   ← Dashboard (subir a GitHub Pages)
```

---

## Paso 1 — Google Drive

1. Crear carpeta en Google Drive: **"Dashboard_Chemes_Export"**
2. Copiar el **ID** de la carpeta (aparece en la URL al abrirla)
3. Instalar Google Drive for Desktop en el servidor
4. Sincronizar `C:\Exports\Dashboard` con esa carpeta de Drive

---

## Paso 2 — PowerShell

1. Copiar `Export-DashboardData.ps1` y `Ejecutar_Dashboard.bat`
   a la carpeta `F:\Tarea\Dashboard\`
2. Crear carpeta de logs: `F:\Tarea\Dashboard\Logs\`
3. Ajustar parámetros en el .bat si la sucursal no es VERA:
   ```
   -Sucursal "RECONQUISTA"
   ```
4. Programar en Task Scheduler:
   - Acción: Ejecutar `Ejecutar_Dashboard.bat`
   - Desencadenador: Diario, repetir cada 60 min, de 07:00 a 21:00
   - Ejecutar con privilegios elevados: Sí

5. Probar manualmente:
   ```powershell
   cd F:\Tarea\Dashboard
   .\Export-DashboardData.ps1 -Sucursal "VERA"
   ```
   Verificar que aparezcan los CSV en `C:\Exports\Dashboard\`

---

## Paso 3 — Apps Script

1. Ir a https://script.google.com → Nuevo proyecto
2. Pegar el contenido de `DashboardChemes.gs`
3. Editar la línea:
   ```javascript
   var SOURCE_FOLDER_ID = "PEGAR_ID_DE_CARPETA_AQUI";
   ```
   con el ID de la carpeta del Paso 1

4. **Publicar como Web App:**
   - Menú: Implementar → Nueva implementación
   - Tipo: Aplicación web
   - Ejecutar como: **Yo** (tu cuenta Google)
   - Acceso: **Cualquier usuario**
   - Copiar la URL generada (formato: `https://script.google.com/macros/s/...XXXX.../exec`)

5. **Instalar trigger de cache:**
   - Menú: Activadores (reloj)
   - Agregar activador: función `invalidarCache`
   - Tipo: Intervalo de tiempo → Cada hora

6. Probar la URL en el navegador:
   ```
   https://script.google.com/macros/s/TU_URL/exec?modulo=ventas&sucursal=VERA
   ```
   Debe retornar JSON.

---

## Paso 4 — GitHub Pages

1. Crear repositorio en GitHub (puede ser el existente `bjcbaigo`)
2. Subir `web/index.html` a la raíz o carpeta `dashboardweb/`
3. En Settings → Pages → Source: **main branch / root**
4. Editar `index.html` — línea de configuración:
   ```javascript
   var CONFIG = {
     WEB_APP_URL: "https://script.google.com/macros/s/TU_URL_AQUI/exec",
     SUCURSAL:    "VERA",
     DEMO_MODE:   false   // ← cambiar a false cuando la URL esté cargada
   };
   ```
5. Hacer commit y push.

URL final: `https://bjcbaigo.github.io/dashboardweb/`
(o la ruta donde quede el index.html)

---

## CSV de Objetivos (manual)

Para cargar objetivos sin tocar la BD, subir el archivo `objetivos.csv`
a la carpeta de Google Drive con el formato del template:

```
sep=;
Anio;Mes;Sucursal;Categoria;ObjetivoVentas;ObjetivoUnidades;ObjetivoClientesNuevos;ObjetivoPolizas;ObjetivoGarantias
2026;4;VERA;GENERAL;11500000;140;85;32;42
```

El Apps Script prioriza `objetivos.csv` (manual) sobre `objetivos_db.csv` (BD).

---

## Multi-sucursal

Para agregar otra sucursal, duplicar el .bat cambiando `-Sucursal`:
```
-Sucursal "RECONQUISTA"
-OutputFolder "C:\Exports\Dashboard_RECONQUISTA"
```

Y en el HTML acceder con parámetro URL:
```
https://bjcbaigo.github.io/dashboardweb/?sucursal=RECONQUISTA
```

(agregar lectura de `?sucursal=` con `new URLSearchParams(location.search)` en el CONFIG)

---

## Tablas SQL utilizadas

| Archivo CSV        | Tablas / Vistas                          |
|--------------------|------------------------------------------|
| ventas.csv         | GVA12, GVA14, GVA51                      |
| ventas_articulos.csv | GVA13, STA11                           |
| stock.csv          | STA09, STA11                             |
| morosidad.csv      | GVA46, GVA14                             |
| seguros.csv        | VW_VentasSeguros (CENTRAL_CHEMES)        |
| garantias.csv      | GVA13 (filtro cod_art LIKE 'GAR%')       |
| objetivos_db.csv   | Dim_Objetivos (CENTRAL_CHEMES)           |
| meta.json          | generado por PowerShell                  |

---

## Notas importantes

- El índice de garantías usa `cod_art LIKE 'GAR%' OR 'EXT%' OR 'COB%'`.
  Ajustar los prefijos según los códigos reales en STA11 de la sucursal.

- La tabla `STA09` es el saldo de stock en Tango. Si el nombre difiere
  (ej. `STA08` en alguna versión), ajustar en el PowerShell.

- El campo `fec_ult_mov` en `STA11` puede no existir en todas las versiones
  de Tango Gestión. Si no existe, reemplazar por una subquery contra GVA13.

- El Web App de Apps Script tiene límite de 20.000 ejecuciones/día en
  cuentas gratuitas. Con cache de 10 min y ~30 usuarios simultáneos es más
  que suficiente.
