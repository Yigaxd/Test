-- SCRIPT SQL: NORMALIZACIÓN DE PLANILLA DESNORMALIZADA - TALLER MECÁNICO
--
-- 1. PRIMERA FORMA NORMAL (1FN)
--
-- ANOMALÍAS DETECTADAS EN LA PLANILLA ORIGINAL:
--
-- GRUPOS REPETITIVOS / FILAS DUPLICADAS:
--     La orden ID_ORDEN=5001 aparece repetida en múltiples filas, una por cada
--     repuesto utilizado. Los datos del cliente (RUT_CLIENTE, NOM_CLIENTE,
--     FONO_CLIENTE), del vehículo (PATENTE_AUTO, MARCA_AUTO) y del mecánico
--     (MECANICO_RUT, MECANICO_NOM) se repiten en cada fila, lo cual viola 1FN.
--
-- AUSENCIA DE CLAVE PRIMARIA SIMPLE:
--     Ninguna columna sola identifica unívocamente cada fila. Se requiere la
--     combinación (ID_ORDEN, ID_REPUESTO) como clave compuesta para identificar
--     cada detalle de la orden.
--
-- CORRECCIÓN APLICADA:
--     Se define la clave primaria compuesta (ID_ORDEN, ID_REPUESTO) para la
--     tabla de detalles.
--
-- SEGUNDA FORMA NORMAL (2FN)
-- 
-- ANOMALÍAS DETECTADAS (dependencias parciales):
--
--   Clave compuesta: (ID_ORDEN, ID_REPUESTO)
--
--   Dependencias parciales encontradas:
--   → ID_ORDEN  → FECHA, RUT_CLIENTE, NOM_CLIENTE, FONO_CLIENTE,
--                 PATENTE_AUTO, MARCA_AUTO, MECANICO_RUT, MECANICO_NOM
--     (estos datos pertenecen a la orden, NO al par orden+repuesto)
--
--   ANOMALÍAS DE MODIFICACIÓN: Si el precio de un repuesto cambia, habría
--   que actualizar múltiples filas arriesgando inconsistencias.
--
--   ANOMALÍAS DE INSERCIÓN: No se puede registrar un repuesto sin asociarlo
--   a una orden existente.
--
--   ANOMALÍAS DE ELIMINACIÓN: Si se elimina la única orden que usa un repuesto,
--   se pierde la información del repuesto.
--
-- CORRECCIÓN APLICADA:
--   Se separan las entidades independientes:
--   - REPUESTO(ID_REPUESTO, NOM_REPUESTO, PRECIO_UNITARIO)
--     → NOM_REPUESTO y PRECIO_UNITARIO dependen solo de ID_REPUESTO.
--   - ORDEN(ID_ORDEN, FECHA, RUT_CLIENTE, PATENTE_AUTO, MECANICO_RUT)
--     → Los datos de la orden dependen solo de ID_ORDEN.
--   - DETALLE_ORDEN(ID_ORDEN, ID_REPUESTO, CANTIDAD)
--     → CANTIDAD sí depende completamente de (ID_ORDEN, ID_REPUESTO).
--
-- TERCERA FORMA NORMAL (3FN)
--
-- ANOMALÍAS DETECTADAS (dependencias transitivas):
--
-- En la tabla ORDEN (tras aplicar 2FN) aún persisten:
--   Si un cliente cambia su teléfono, habría que actualizar todas
--   sus órdenes. Si se elimina la única orden de un cliente, se pierde su info.
--
-- CORRECCIÓN APLICADA:
--   Se extraen las entidades transitivas a tablas propias:
--   - CLIENTE(RUT_CLIENTE, NOM_CLIENTE, FONO_CLIENTE)
--   - VEHICULO(PATENTE_AUTO, MARCA_AUTO, RUT_CLIENTE [FK])
--   - MECANICO(MECANICO_RUT, MECANICO_NOM)
--   - ORDEN(ID_ORDEN, FECHA, RUT_CLIENTE [FK], PATENTE_AUTO [FK], MECANICO_RUT [FK])
--
-- CREACIÓN DE TABLAS NORMALIZADAS (3FN)
--
-- TABLA: CLIENTE
-- Entidad independiente extraída para eliminar dependencias transitivas.
-- RUT_CLIENTE es la clave primaria natural del cliente.
-- -----------------------------------------------------------------------------
CREATE TABLE CLIENTE (
    RUT_CLIENTE   VARCHAR(15)  NOT NULL,
    NOM_CLIENTE   VARCHAR(100) NOT NULL,
    FONO_CLIENTE  VARCHAR(20),
    CONSTRAINT PK_CLIENTE PRIMARY KEY (RUT_CLIENTE)
);

-- -----------------------------------------------------------------------------
-- TABLA: VEHICULO
-- Entidad independiente. PATENTE_AUTO identifica unívocamente al vehículo.
-- Se asocia al cliente propietario mediante FK.
-- -----------------------------------------------------------------------------
CREATE TABLE VEHICULO (
    PATENTE_AUTO  VARCHAR(10)  NOT NULL,
    MARCA_AUTO    VARCHAR(50)  NOT NULL,
    RUT_CLIENTE   VARCHAR(15)  NOT NULL,
    CONSTRAINT PK_VEHICULO PRIMARY KEY (PATENTE_AUTO),
    CONSTRAINT FK_VEHICULO_CLIENTE FOREIGN KEY (RUT_CLIENTE)
        REFERENCES CLIENTE (RUT_CLIENTE)
);

-- -----------------------------------------------------------------------------
-- TABLA: MECANICO
-- Entidad independiente extraída para eliminar la dependencia transitiva
-- ID_ORDEN → MECANICO_RUT → MECANICO_NOM.
-- -----------------------------------------------------------------------------
CREATE TABLE MECANICO (
    MECANICO_RUT  VARCHAR(15)  NOT NULL,
    MECANICO_NOM  VARCHAR(100) NOT NULL,
    CONSTRAINT PK_MECANICO PRIMARY KEY (MECANICO_RUT)
);

-- -----------------------------------------------------------------------------
-- TABLA: REPUESTO
-- Entidad independiente extraída para eliminar la dependencia parcial
-- ID_REPUESTO → {NOM_REPUESTO, PRECIO_UNITARIO}.
-- -----------------------------------------------------------------------------
CREATE TABLE REPUESTO (
    ID_REPUESTO      VARCHAR(10)    NOT NULL,
    NOM_REPUESTO     VARCHAR(100)   NOT NULL,
    PRECIO_UNITARIO  DECIMAL(10, 2) NOT NULL,
    CONSTRAINT PK_REPUESTO PRIMARY KEY (ID_REPUESTO)
);

-- -----------------------------------------------------------------------------
-- TABLA: ORDEN
-- Representa la orden de servicio. Todos sus atributos dependen directa y
-- completamente de ID_ORDEN (clave primaria simple). Las referencias a cliente,
-- vehículo y mecánico se resuelven mediante claves foráneas.
-- -----------------------------------------------------------------------------
CREATE TABLE ORDEN (
    ID_ORDEN      INT          NOT NULL,
    FECHA         DATE         NOT NULL,
    RUT_CLIENTE   VARCHAR(15)  NOT NULL,
    PATENTE_AUTO  VARCHAR(10)  NOT NULL,
    MECANICO_RUT  VARCHAR(15)  NOT NULL,
    CONSTRAINT PK_ORDEN PRIMARY KEY (ID_ORDEN),
    CONSTRAINT FK_ORDEN_CLIENTE FOREIGN KEY (RUT_CLIENTE)
        REFERENCES CLIENTE (RUT_CLIENTE),
    CONSTRAINT FK_ORDEN_VEHICULO FOREIGN KEY (PATENTE_AUTO)
        REFERENCES VEHICULO (PATENTE_AUTO),
    CONSTRAINT FK_ORDEN_MECANICO FOREIGN KEY (MECANICO_RUT)
        REFERENCES MECANICO (MECANICO_RUT)
);

-- -----------------------------------------------------------------------------
-- TABLA: DETALLE_ORDEN
-- Tabla de intersección que resuelve la relación M:N entre ORDEN y REPUESTO.
-- La clave primaria compuesta (ID_ORDEN, ID_REPUESTO) garantiza que CANTIDAD
-- depende completamente de ambos atributos clave, cumpliendo 2FN y 3FN.
-- -----------------------------------------------------------------------------
CREATE TABLE DETALLE_ORDEN (
    ID_ORDEN     INT            NOT NULL,
    ID_REPUESTO  VARCHAR(10)    NOT NULL,
    CANTIDAD     INT            NOT NULL,
    CONSTRAINT PK_DETALLE_ORDEN PRIMARY KEY (ID_ORDEN, ID_REPUESTO),
    CONSTRAINT FK_DETALLE_ORDEN FOREIGN KEY (ID_ORDEN)
        REFERENCES ORDEN (ID_ORDEN),
    CONSTRAINT FK_DETALLE_REPUESTO FOREIGN KEY (ID_REPUESTO)
        REFERENCES REPUESTO (ID_REPUESTO)
);


-- =============================================================================
-- DATOS DE PRUEBA (extraídos de la planilla original)
-- =============================================================================

INSERT INTO CLIENTE VALUES ('15.432.111-K', 'Carlos Tapia', '56911112222');
INSERT INTO CLIENTE VALUES ('18.765.432-1', 'Maria Pinto',  '56933334444');

INSERT INTO VEHICULO VALUES ('DL-PP-40', 'Toyota',  '15.432.111-K');
INSERT INTO VEHICULO VALUES ('GK-ST-12', 'Hyundai', '18.765.432-1');

INSERT INTO MECANICO VALUES ('10.888.777-6', 'Juan Gomez');
INSERT INTO MECANICO VALUES ('12.555.444-K', 'Raul Soto');

INSERT INTO REPUESTO VALUES ('R01', 'Filtro de Aceite',      12000.00);
INSERT INTO REPUESTO VALUES ('R02', 'Aceite Sintético 5W30', 38000.00);
INSERT INTO REPUESTO VALUES ('R03', 'Pastillas de Freno',    25000.00);

-- Fecha 46174 en Excel serial = 2026-05-21 (aprox); se usa fecha representativa
INSERT INTO ORDEN VALUES (5001, '2026-05-21', '15.432.111-K', 'DL-PP-40', '10.888.777-6');
INSERT INTO ORDEN VALUES (5002, '2026-05-22', '18.765.432-1', 'GK-ST-12', '12.555.444-K');
INSERT INTO ORDEN VALUES (5003, '2026-05-23', '15.432.111-K', 'DL-PP-40', '10.888.777-6');

INSERT INTO DETALLE_ORDEN VALUES (5001, 'R01', 1);
INSERT INTO DETALLE_ORDEN VALUES (5001, 'R02', 1);
INSERT INTO DETALLE_ORDEN VALUES (5002, 'R02', 1);
INSERT INTO DETALLE_ORDEN VALUES (5002, 'R03', 2);
INSERT INTO DETALLE_ORDEN VALUES (5003, 'R03', 1);


-- =============================================================================
-- CONSULTA DE VERIFICACIÓN: reconstruye la vista desnormalizada original
-- =============================================================================
SELECT
    o.ID_ORDEN,
    o.FECHA,
    c.RUT_CLIENTE,
    c.NOM_CLIENTE,
    c.FONO_CLIENTE,
    v.PATENTE_AUTO,
    v.MARCA_AUTO,
    r.ID_REPUESTO,
    r.NOM_REPUESTO,
    r.PRECIO_UNITARIO,
    d.CANTIDAD,
    m.MECANICO_RUT,
    m.MECANICO_NOM
FROM ORDEN          o
JOIN CLIENTE        c ON o.RUT_CLIENTE   = c.RUT_CLIENTE
JOIN VEHICULO       v ON o.PATENTE_AUTO  = v.PATENTE_AUTO
JOIN MECANICO       m ON o.MECANICO_RUT  = m.MECANICO_RUT
JOIN DETALLE_ORDEN  d ON o.ID_ORDEN      = d.ID_ORDEN
JOIN REPUESTO       r ON d.ID_REPUESTO   = r.ID_REPUESTO
ORDER BY o.ID_ORDEN, r.ID_REPUESTO;
