---
# Sobres — Constructora Capital

Este proyecto surgio de la necesidad de hacer seguimiento a los sobres electronicos de firma digital que maneja la constructora. El reporte que exporta la plataforma llega bastante crudo, asi que hicimos un proceso de limpieza en Python para dejarlo listo y conectarlo a un dashboard interactivo en Power BI.

---

## ¿Que hay en este repositorio?
## Requisitos

- Python 3.8 en adelante
- pandas y numpy

```bash
pip install pandas numpy
```

---

## ¿Que hace el notebook?

El notebook `EDA_Sobres.ipynb` tiene todo el proceso paso a paso. Aca un resumen de que hace cada parte:

### Carga del archivo
Antes de leer el CSV detectamos si el separador es coma o punto y coma, porque dependiendo de como se exporta puede variar. Tambien usamos la codificacion `utf-8-sig` para evitar problemas con caracteres especiales que suelen aparecer en archivos exportados desde Excel.

### Renombrado de columnas
Los nombres originales venian en ingles y poco descriptivos, asi que los renombramos a español. Algo importante: cada fila del archivo representa a un destinatario dentro de un sobre, o sea, si un sobre tiene 3 firmantes aparece 3 veces.

### Correccion de fechas
Este fue uno de los problemas mas grandes que encontramos. Las fechas venian en formato colombiano (DD/MM/YYYY) pero Python por defecto las leia al reves (MM/DD/YYYY), lo que generaba tiempos negativos y sobres que supuestamente tardaban mas de un año. La solucion fue usar `dayfirst=True` para forzar la lectura correcta.

Tambien calculamos dos versiones de la fecha de completado:
- Una por firmante (cuando firmo cada persona)
- Una por sobre (cuando firmo el ultimo, que es la que realmente indica cuando se cerro)

Con eso calculamos `Tiempo_Dias`: cuantos dias tardo cada sobre desde que se envio hasta que lo firmaron todos.

### Clasificacion del tipo de documento
El campo `Asunto` tiene un codigo de 6 digitos al inicio que indica que tipo de contrato es. Construimos una funcion que lee ese codigo y lo convierte a un nombre legible (por ejemplo, `120101` → Cierre de venta). Tambien detecta si el sobre tiene errores de marcacion o si fue devuelto alguna vez.

| Codigo | Tipo |
|--------|------|
| 120101 | Cierre de venta |
| 120102 | Promesa de compraventa |
| 120103 | Otrosi |
| 120104 | Cesion de derechos |
| 120106 | Otrosi ajuste de salarios |
| 120107 | Contrato mera tenencia |
| 120112 | Contratos de vinculacion |
| 120115 | Exoneracion impuesto rentas y registro |

### Extraccion del proyecto, torre y unidad
El mismo campo `Asunto` tambien tiene el nombre del proyecto, la torre y el apartamento o parqueadero, pero en texto libre y con muchas variaciones. Usamos expresiones regulares para extraer esa informacion y luego la normalizamos con un diccionario que mapea todas las variantes al nombre oficial del proyecto.

### Flags de gestion
Creamos dos columnas para identificar sobres que necesitan atencion:
- `Requiere_Seguimiento`: sobres que siguen activos, sin completar ni anular.
- `Requiere_Calidad`: sobres que tienen error en la marcacion del asunto.

### Firmante pendiente
Para los sobres que todavia no estan completos, identificamos quien tiene el sobre en este momento sin haberlo firmado. Tambien lo categorizamos (cliente, sala de ventas, tramitador, archivo, etc.) para saber donde se estan generando los cuellos de botella.

### Semaforo
Clasificamos cada sobre en verde, amarillo o rojo segun cuanto tiempo lleva:
- Si ya esta completado usamos `Tiempo_Dias`.
- Si todavia esta en proceso calculamos cuantos dias han pasado desde que se envio hasta hoy.

Eso permite ver en tiempo real cuales sobres ya se estan demorando demasiado.

### Exportacion
Generamos dos archivos:
- `Sobres_Limpio.csv`: el dataset principal con todas las columnas limpias y listo para cargar en Power BI. Las fechas van en formato `YYYY-MM-DD` para que Power BI las reconozca sin problema.
- `Prediccion_Sobres.csv`: una serie de tiempo con el historico de sobres por mes y tipo de documento, mas una proyeccion de los proximos 6 meses usando suavizamiento exponencial de Holt.

---

## Archivos de salida

### `Sobres_Limpio.csv`

| Columna | Que es |
|---------|--------|
| `ID_Sobre` | Identificador unico del sobre |
| `Proyecto` | Nombre del proyecto |
| `Torre` / `Unidad` | Torre y apartamento o parqueadero |
| `Tipo_Documento` | Tipo de contrato |
| `Estado` | Completado / Enviado / Anulado |
| `Tiempo_Dias` | Dias que tardo en completarse |
| `Semaforo` | Verde, amarillo o rojo |
| `Firmante_Pendiente` | Quien tiene el sobre pendiente |
| `Categoria_Firmante` | A que area pertenece ese firmante |
| `Requiere_Seguimiento` | Si el sobre sigue activo |
| `Requiere_Calidad` | Si tiene error de marcacion |
| `Año_Mes` | Mes de envio en formato YYYY-MM |

### `Prediccion_Sobres.csv`

| Columna | Que es |
|---------|--------|
| `Año_Mes` | El mes |
| `Codigo` | Codigo del tipo de documento |
| `Total_Sobres` | Sobres reales ese mes |
| `Proyeccion` | Valor historico o proyectado |
| `Limite_Inf` | Limite inferior del intervalo de confianza |
| `Limite_Sup` | Limite superior del intervalo de confianza |
| `Tipo` | Historico o Proyeccion |
| `Orden_Mes` | Numero de orden para graficar en Power BI |

---

## Dashboard Power BI

El dashboard tiene 7 paginas, cada una enfocada en un aspecto distinto del proceso.

### 1. Resumen General
Es la pagina de entrada. Muestra un panorama global con tarjetas de indicadores clave: total de sobres, cuantos estan completados, enviados, anulados y declinados, y el porcentaje de completados. Ademas tiene una grafica de dona con la distribucion por estado, un grafico de barras con el volumen por proyecto y una linea de tiempo que muestra como ha evolucionado el flujo de sobres mes a mes.

### 2. Seguimiento de Firmas
Aca esta el detalle operativo. Hay una tabla con todos los sobres activos que muestra el asunto, proyecto, tipo de documento, destinatario, estado, fecha de envio, cuantos dias lleva y en que accion esta cada firmante. Se puede filtrar por proyecto, estado y tipo de documento. Tambien hay una tarjeta con el promedio de dias de finalizacion para tener una referencia rapida.

### 3. Tiempos y Devoluciones
Esta pagina se enfoca en la eficiencia del proceso. Tiene tres graficos de barras: uno con el promedio de dias que tarda cada proyecto en completar sus sobres, otro con el total de devoluciones por proyecto, y otro que muestra las devoluciones segun el tipo de error. Tambien incluye una tarjeta con la fecha de corte del reporte.

### 4. Gestion de Seguimiento
Muestra los sobres que todavia no han sido completados ni anulados y necesitan atencion. Incluye una tarjeta con el total de esos sobres, una tabla con el detalle (asunto, proyecto, estado y quien tiene el sobre pendiente), un grafico de barras por proyecto y una tarjeta con los dias promedio sin completar. Se puede filtrar por proyecto, tipo de documento y categoria del firmante.

### 5. Gestion de Calidad
Similar a la anterior, pero enfocada en los sobres con errores de marcacion. Muestra cuantos sobres tienen ese problema, el detalle en una tabla y el total de devoluciones asociadas. Se puede filtrar por proyecto, tipo de documento y estado.

### 6. Semaforo
Esta pagina permite dar un vistazo de  como estan los tiempos de gestion. Hay tres tarjetas que muestran cuantos sobres estan en verde, amarillo y rojo, y dos graficos de barras apiladas: uno por tipo de documento y otro por proyecto. Se puede filtrar por proyecto, tipo de documento y mes.

### 7. Prediccion
Muestra una linea de tiempo que combina el historico real de sobres por mes con la proyeccion de los proximos 6 meses, calculada con suavizamiento exponencial de Holt desde el notebook de Python. La banda sombreada entre `Limite_Inf` y `Limite_Sup` representa el intervalo de confianza al 90%, comunicando visualmente la incertidumbre de la proyeccion. Sin embargo, al tener tan poca información y tener una muestra de datos tan reducida, no es posible ser certero y realizar una buena proyección de generación de sobres mes a mes.

---

## Autores

Diana Marcela Tuberquia Lozano
Juan Esteban Marín Mejía
Carolina Tobon Betancur


