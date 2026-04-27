# 🤖 Módulo: Automatización de Estadísticas (Automated Stats)

Guía paso a paso para entender, configurar y automatizar la recolección de estadísticas en Teradata usando Python y Shell.

---

## 🧠 ¿Qué son las Estadísticas en Teradata?

Imagina que Teradata es una biblioteca gigante y el "Optimizador" (el motor interno que decide cómo buscar la información) es el bibliotecario. Las **estadísticas** son el catálogo detallado de esa biblioteca. 

Le dicen al bibliotecario exactamente cuántos libros hay de cada tema, en qué estantes están distribuidos y si hay muchos libros de un mismo autor. Sin este catálogo actualizado, el bibliotecario tendría que caminar y revisar estante por estante. En bases de datos, esto se conoce como un "Full Table Scan", un proceso que consume mucho tiempo y recursos.

---

## 🎯 ¿Para qué sirven?

Mantener las estadísticas actualizadas sirve para garantizar que las consultas (queries) se ejecuten de la forma más rápida y eficiente posible. Específicamente ayudan a:

* **Elegir la mejor ruta:** El Optimizador toma decisiones inteligentes sobre si debe usar un índice, hacer un escaneo rápido o cómo cruzar la información de dos tablas distintas.
* **Ahorrar capacidad del sistema:** Las consultas eficientes consumen menos procesador (CPU) y menos lectura de discos (I/O).
* **Evitar cuellos de botella:** Previenen que una consulta mal planificada acapare los recursos y bloquee el sistema para el resto de los usuarios.

---

## ⚙️ Cómo automatizar la recolección (El método probado)

Recolectar estadísticas a mano tabla por tabla es insostenible en ambientes productivos. A continuación, se detallan los pasos exactos para crear un motor automatizado confiable usando Python, basado en ejecuciones exitosas en entornos reales.

### Paso 1: Identificar las tablas candidatas
El script debe consultar el diccionario de datos del sistema para saber qué tablas crecieron o cambiaron lo suficiente como para necesitar una actualización de su catálogo.

**Query base para el script:**
```sql
SELECT 
    DatabaseName, 
    TableName 
FROM DBC.StatsV 
WHERE LastCollectTimeStamp < CURRENT_DATE - 7;
