# Actividad-8
REFLEXIONES Y PROCESO DE CADA RETO 
##  Resumen General del Proyecto
En esta actividad construimos la base de datos de una empresa agrícola colombiana con fincas en Cundinamarca, Caldas y Boyacá. La solución integra:
1. **DDL (Data Definition Language):** Creación de la estructura relacional con restricciones de integridad (`PRIMARY KEY`, `FOREIGN KEY`, `CHECK`, `UNIQUE`, `ENUM`).
2. **DML (Data Manipulation Language):** Consultas analíticas complejas con `JOIN`, `GROUP BY`, `HAVING`, subconsultas y funciones de fecha.
3. **Normalización (3FN):** Migración y refactorización de planillas de campo sin estructurar para eliminar redundancias y anomalías.
4. **Visión Artificial (YOLO):** Modelado y almacenamiento de detecciones geoespaciales para el control de plagas y soporte a la toma de decisiones agrícolas.

---

##  Desarrollo de Puntos y Retos Resueltos

### Punto 1 · Modelado y Creación de Tablas (DDL)
Se extendió el modelo relacional incorporando el control de métodos de riego y la gestión de operarios encargados de la aplicación de insumos.

* **Solución SQL Reto 1:**
```sql
sql('''
ALTER TABLE lote 
ADD COLUMN tipo_riego ENUM('goteo', 'aspersión', 'gravedad', 'secano') NOT NULL DEFAULT 'secano';

CREATE TABLE operario (
    id_operario         INT AUTO_INCREMENT PRIMARY KEY,
    documento           VARCHAR(15) NOT NULL UNIQUE,
    nombre_completo     VARCHAR(80) NOT NULL,
    telefono            VARCHAR(15),
    certificado_aplicador BOOLEAN NOT NULL DEFAULT FALSE
);

INSERT INTO operario (documento, nombre_completo, telefono, certificado_aplicador) VALUES
  ('1012', 'Jorge Patiño', '3101112233', TRUE),
  ('2045', 'Luz Marina Ortiz', '3125556677', TRUE),
  ('3090', 'Ramiro Suárez', '3009998877', FALSE);

SELECT * FROM operario;
''')
```

####  Preguntas de Reflexión 1
* **¿Por qué `area_ha` es `DECIMAL(6,2)` y no `FLOAT`?**  
  `DECIMAL` almacena números de precisión exacta fija. Se evita el problema de imprecisión en punto flotante que tiene `FLOAT` al realizar redondeos binarios, garantizando exactitud en los cálculos de áreas y compras de insumos.
* **¿Diferencia entre validar el área en HTML vs `CHECK` en MySQL?**  
  La validación HTML brinda retroalimentación rápida en la interfaz de usuario, pero es fácilmente vulnerada o anulada. La restricción `CHECK` en MySQL actúa directamente en el motor de la base de datos como la última línea de defensa técnica, asegurando que ningún dato inválido ingrese sin importar el cliente que realice la transacción.
* **Desventaja de `ENUM` para el cultivo y alternativa:**  
  La desventaja es la rigidez: agregar un nuevo cultivo exige ejecutar un `ALTER TABLE MODIFY COLUMN`. La mejor alternativa es crear una tabla catálogo `cultivo` y relacionarla mediante una llave foránea (`FOREIGN KEY`).

---

### Punto 2 · Integridad Referencial e Insumos con Ficha Técnica
Se registró el inventario de productos respetando las restricciones del ICA y se automatizó el cálculo de dosis y periodos de seguridad.

* **Solución SQL Reto 2A (Productos Faltantes):**
```sql
sql('''
INSERT INTO producto (nombre_comercial, tipo, ingrediente_activo, categoria_tox, registro_ica,
                      dosis_ha, unidad, carencia_dias, reingreso_horas, precio_unidad) VALUES
 ('BioLarva WP',     'bioinsumo', 'Bacillus thuringiensis var. kurstaki', 'IV', 'ICA-DEMO-2001', 0.5, 'kg', 0, 4, 95000),
 ('RoyaStop 250 EC', 'fungicida', 'Triazol 250 g/L',                      'III', 'ICA-DEMO-3001', 0.4, 'L', 30, 24, 160000),
 ('TizónFin 80 WP',  'fungicida', 'Ditiocarbamato 800 g/kg',              'III', 'ICA-DEMO-3002', 2.5, 'kg', 14, 24, 38000);
''')
```

* **Solución SQL Reto 2B (Subconsultas y Tiempos de Cosecha):**
```sql
sql('''
INSERT INTO aplicacion (id_lote, id_producto, fecha, cantidad, responsable)
SELECT l.id_lote, p.id_producto, '2026-10-14', p.dosis_ha * l.area_ha, 'Jorge Patiño'
FROM lote l, producto p 
WHERE l.codigo = 'POR-L01' AND p.nombre_comercial = 'RoyaStop 250 EC';

SELECT a.id_aplicacion, l.codigo AS lote, p.nombre_comercial, a.fecha AS fecha_aplicacion,
       p.carencia_dias, DATE_ADD(a.fecha, INTERVAL p.carencia_dias DAY) AS cosecha_desde
FROM aplicacion a
JOIN lote l     ON l.id_lote = a.id_lote
JOIN producto p ON p.id_producto = a.id_producto;
''')
```

####  Preguntas de Reflexión 2
* **Error de `id_lote = 99`:** Violación de llave foránea (`Cannot add or update a child row`). Evita que existan registros huérfanos asociados a lotes inexistentes.
* **Imposibilidad de borrar la finca 1:** La regla `ON DELETE RESTRICT` impidió el borrado para proteger los lotes asociados. Se usaría `ON DELETE CASCADE` únicamente cuando la eliminación del registro principal requiera eliminar automáticamente sus dependientes (ej. al borrar un vuelo de dron y sus detecciones).
* **`registro_ica` como `UNIQUE`:** Garantiza la trazabilidad única del producto comercial. Evita la duplicidad o adulteración en registros oficiales.
* **Control de reingreso (10h vs 24h necesarias):** El dato `reingreso_horas` lo impidió. En la web se muestra mediante un bloqueo visual o alerta en el tablero del lote notificando "Lote en periodo de reingreso restringido hasta [Fecha y Hora]".

---

### Punto 3 · Consultas para Tomar Decisiones (DML)
Se generaron consultas para optimizar costos, identificar áreas sembradas y evaluar la gestión de insumos en finca.

* **Solución SQL Reto 3:**
```sql
sql('''
-- 1) Área total sembrada por cultivo
SELECT cultivo, SUM(area_ha) AS area_total_ha
FROM lote
GROUP BY cultivo
ORDER BY area_total_ha DESC;

-- 2) Fincas con área total > 5 ha
SELECT f.nombre AS finca, SUM(l.area_ha) AS area_total
FROM finca f
JOIN lote l ON f.id_finca = l.id_finca
GROUP BY f.id_finca, f.nombre
HAVING area_total > 5;

-- 3) Productos nunca aplicados
SELECT p.id_producto, p.nombre_comercial
FROM producto p
LEFT JOIN aplicacion a ON p.id_producto = a.id_producto
WHERE a.id_aplicacion IS NULL;

-- 4) Responsable con mayor fertilizante aplicado
SELECT a.responsable, SUM(a.cantidad) AS total_kg_aplicado
FROM aplicacion a
JOIN producto p ON a.id_producto = p.id_producto
WHERE p.tipo = 'fertilizante'
GROUP BY a.responsable
ORDER BY total_kg_aplicado DESC
LIMIT 1;
''')
```

####  Preguntas de Reflexión 3
* **Cambiar `LEFT JOIN` por `JOIN` en 3.3:** Devolvería un resultado vacío, porque un `INNER JOIN` exige intersección en ambas tablas y justamente buscábamos productos con conteo nulo.
* **`GROUP BY` con `l.codigo` y `l.cultivo`:** Se agregan para cumplir con la regla estricta de SQL donde todo campo proyectado en el `SELECT` que no sea función de agregado debe pertenecer al agrupamiento.
* **¿Dónde calcular el costo por hectárea (SQL vs JS)?** Es conveniente calcularlo en **SQL**. Reduce el volumen de datos enviados a través de la red (ancho de banda) y centraliza la lógica del negocio en el backend.

---

### Punto 4 · Normalización de Planilla de Campo (3FN)
Se solucionaron los problemas de redundancia y anomalías de una planilla sin estructurar, migrándola a Tercera Forma Normal.

* **Solución SQL Reto 4:**
```sql
sql('''
-- 1) Crear tabla aplicacion_n en 3FN
DROP TABLE IF EXISTS aplicacion_n;
CREATE TABLE aplicacion_n (
    id                 INT AUTO_INCREMENT PRIMARY KEY,
    fecha              DATE NOT NULL,
    id_lote            INT NOT NULL,
    id_producto        INT NOT NULL,
    cantidad           DECIMAL(8,2) NOT NULL,
    documento_operario VARCHAR(15) NOT NULL,
    observaciones      VARCHAR(200),
    CONSTRAINT fk_apln_lote     FOREIGN KEY (id_lote)     REFERENCES lote(id_lote),
    CONSTRAINT fk_apln_prod     FOREIGN KEY (id_producto) REFERENCES producto(id_producto),
    CONSTRAINT fk_apln_oper     FOREIGN KEY (documento_operario) REFERENCES operario_n(documento)
);

-- 2) Migración mediante INSERT ... SELECT
INSERT INTO aplicacion_n (fecha, id_lote, id_producto, cantidad, documento_operario, observaciones)
SELECT pc.fecha, l.id_lote, p.id_producto, pc.cantidad, pc.documento_operario, pc.observaciones
FROM planilla_campo pc
JOIN lote l     ON l.codigo = pc.codigo_lote
JOIN producto p ON p.nombre_comercial = pc.producto;

-- 3) Reconstrucción de la planilla original
SELECT a.fecha, l.codigo AS codigo_lote, l.cultivo, p.nombre_comercial AS producto,
       p.categoria_tox, a.cantidad, o.documento AS documento_operario,
       o.nombre_completo AS nombre_operario, o.telefono AS telefono_operario, a.observaciones
FROM aplicacion_n a
JOIN lote l        ON l.id_lote = a.id_lote
JOIN producto p    ON p.id_producto = a.id_producto
JOIN operario_n o  ON o.documento = a.documento_operario;
''')
```

####  Preguntas de Reflexión 4
* **Ejemplo de anomalías en la planilla desnormalizada:**
  * *Actualización:* El operario `1012` figuraba con dos teléfonos distintos. Corregir uno implicaba cambiarlo en múltiples filas.
  * *Inserción:* Imposible registrar a un operario nuevo en la base de datos sin que hubiese ejecutado una aplicación previa.
  * *Borrado:* Al borrar la única aplicación de un producto se perdía la información sobre ese producto.
* **Violación de `categoria_tox`:** Violaba la **3FN** por una dependencia transitiva (`id_aplicacion -> producto -> categoria_tox`).
* **¿Cumple `observaciones` la 1FN?** No la cumple si se guardan múltiples datos no atómicos en la misma celda. Se debe rediseñar creando una tabla intermedia `hallazgo_monitoreo` con campos atómicos (`plaga_id`, `porcentaje_afectacion`).
* **Aceptación de Desnormalización:** Es aceptable únicamente para optimizar lecturas masivas en sistemas OLAP o tableros con alto tráfico donde el costo de calcular `JOINs` complejos constantemente penalice el rendimiento.

---

### Punto 5 · Visión Artificial con YOLO y MySQL
Integración de un modelo de detección de objetos en tiempo real (YOLO) para auditar vuelos de dron y disparar alertas automáticas mediante `TRIGGERS`.

* **Solución SQL Reto 5:**
```sql
sql('''
-- 1) Tendencia de detecciones válidas por semana
SELECT l.codigo AS lote, YEARWEEK(i.fecha_hora, 1) AS anio_semana, COUNT(*) AS detecciones_plaga
FROM deteccion d
JOIN imagen i         ON i.id_imagen = d.id_imagen
JOIN lote l           ON l.id_lote = i.id_lote
JOIN clase_objetivo c ON c.clase = d.clase
WHERE d.clase <> 'planta_sana' AND d.confianza >= c.umbral_accion
GROUP BY l.codigo, anio_semana
ORDER BY l.codigo, anio_semana;

-- 2) Densidad de plagas por hectárea
SELECT l.codigo AS lote, l.area_ha, COUNT(*) AS total_detecciones,
       ROUND(COUNT(*) / l.area_ha, 2) AS detecciones_por_ha
FROM deteccion d
JOIN imagen i         ON i.id_imagen = d.id_imagen
JOIN lote l           ON l.id_lote = i.id_lote
JOIN clase_objetivo c ON c.clase = d.clase
WHERE d.clase <> 'planta_sana' AND d.confianza >= c.umbral_accion
GROUP BY l.id_lote, l.codigo, l.area_ha
ORDER BY detecciones_por_ha DESC;

-- 3) Sensibilidad según umbral de confianza para maleza
SELECT 
    SUM(confianza >= 0.30) AS umbral_0_30,
    SUM(confianza >= 0.45) AS umbral_0_45,
    SUM(confianza >= 0.60) AS umbral_0_60,
    SUM(confianza >= 0.75) AS umbral_0_75
FROM deteccion
WHERE clase = 'maleza';

-- 4) Rediseño de estructura para soportar nuevas versiones de modelo
ALTER TABLE imagen ADD COLUMN version_modelo VARCHAR(40) DEFAULT 'yolo11n-agro-v1';
''')
```

####  Preguntas de Reflexión 5
* **Clasificación vs Detección de Objetos:** La clasificación indica únicamente *qué* hay en la imagen global; la detección especifica *qué* y *dónde* está ubicado (`x, y, w, h`), siendo indispensable para cuantificar y contar la cantidad de individuos por hoja o planta.
* **Efecto de modificar el umbral de confianza (0.25 a 0.80):** Al subirlo se reducen los **falsos positivos**, pero aumentan los **falsos negativos**. Para un agricultor es más costoso un falso negativo (no detectar el gusano cogollero a tiempo), ya que la plaga se extiende destruyendo el cultivo.
* **IoU y NMS:** IoU (*Intersection over Union*) calcula el traslape entre dos cajas candidatas. NMS (*Non-Maximum Suppression*) usa el IoU para descartar cajas redundantes que apuntan al mismo objeto conservando la de mayor confianza.
* **Coordenadas normalizadas (0 a 1):** Garantiza que las coordenadas se mantengan válidas e independientes de la resolución del sensor o del reescalado de las imágenes.
* **Sentencias parametrizadas (`%s`):** Evitan la **Inyección SQL**, separando la instrucción lógica de los datos de entrada en las peticiones enviadas desde el backend web.
* **Detección errónea "broccoli" en papa:** Demuestra el sesgo de los modelos preentrenados genéricos (como COCO) al ser aplicados en dominios específicos sin un entrenamiento o *fine-tuning* previo con imágenes del ámbito agrícola.

---

## 📸 Archivos y Evidencias Adjuntas en este Repositorio
* `https://colab.research.google.com/drive/1dGscCks5P4LRih2rRBPfZcbBDeTI5kSl?usp=sharing`: Cuaderno completo resuelto y ejecutado en Google Colab.
* <img src="Captura de pantalla 2026-10-30 084119.png">
* <img src="Captura de pantalla 2026-10-30 084148.png">
