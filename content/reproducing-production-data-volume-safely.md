---
title: Reproducing Production Data Volume Safely
tags:
  - infra
  - infra-fluency
  - system-design
  - performance
  - postgres
---

# Reproducing Production Data Volume Safely

Cuando un problema aparece únicamente con clientes que tienen mucho volumen de datos, reproducirlo fuera de producción no siempre es trivial.

El objetivo no debería ser necesariamente **copiar producción**, sino conseguir una reproducción suficientemente fiel de la característica que provoca el problema:

- volumen de filas;
- distribución de los datos;
- consultas ejecutadas;
- índices y planes de ejecución;
- concurrencia;
- latencia;
- comportamiento de la aplicación con datasets grandes.

Dependiendo de lo que se quiera comprobar, existen varias estrategias con distintos niveles de fidelidad, coste y riesgo.

## 1. Probar directamente en producción

La opción con mayor fidelidad es ejecutar el flujo real contra los datos reales.

Puede ser útil cuando:

- el problema depende fuertemente del tamaño real de los datos;
- necesitamos validar un comportamiento muy concreto;
- no existe una reproducción fiable fuera de producción.

### Ventajas

- Máxima fidelidad.
- Mismos datos, índices, estadísticas y configuración de PostgreSQL.
- Misma infraestructura.
- Permite comprobar el comportamiento que realmente experimenta el usuario.

### Problemas

- Riesgo de afectar a producción.
- Una consulta aparentemente inocua puede consumir muchos recursos.
- `EXPLAIN ANALYZE` ejecuta realmente la consulta.
- Las pruebas destructivas o exploratorias quedan descartadas.
- Es necesario limitar mucho qué se ejecuta y entender previamente su impacto.

Producción puede servir para **observar y medir**, pero no debería convertirse en un entorno de experimentación.

## 2. Analizar las consultas reales en producción

Muchas veces no necesitamos reproducir todo el sistema. Podemos reducir el problema hasta identificar las consultas SQL que ejecuta el flujo problemático.

El enfoque general es:

1. identificar el endpoint o caso de uso implicado;
2. seguir la ejecución hasta las consultas SQL relevantes;
3. medir tiempos;
4. comparar distintos volúmenes o ventanas de datos;
5. inspeccionar los planes de ejecución mediante `EXPLAIN`;
6. entender qué parte escala con el volumen.

Esta estrategia permite responder preguntas como:

- ¿qué consulta está consumiendo el tiempo?
- ¿el coste está en el listado, el `COUNT`, un `SUM`, un join o un cálculo posterior?
- ¿se utiliza un índice?
- ¿aparece un sequential scan?
- ¿el plan cambia al aumentar el rango de datos?
- ¿el problema sigue existiendo actualmente?

### Ventajas

- Trabajamos sobre el dataset real sin copiarlo.
- Permite aislar el cuello de botella.
- El riesgo puede mantenerse bajo si las consultas son conocidas y controladas.
- A menudo es suficiente para verificar si un workaround histórico sigue siendo necesario.

### Limitaciones

- No reproduce el flujo completo desde la interfaz.
- No permite experimentar libremente.
- Una consulta pesada sigue siendo una consulta pesada aunque sea de solo lectura.
- Los resultados pueden variar según carga, cachés, estadísticas o estado de PostgreSQL.

### Caso real: Luzo

En el caso investigado, una feature flag reducía el rango por defecto de la vista de transacciones de 30 días a 1 día para evitar timeouts en cuentas con mucho volumen.

Siguiendo el flujo desde el BFF hasta Core fue posible separar las diferentes partes del coste y medirlas directamente sobre una cuenta con volumen real.

La comparación mostró que:

| Operación | 1 día | 30 días |
| --- | ---: | ---: |
| Página 1 + balances | ~100 ms | ~100 ms |
| `total_count` | 15 ms | 564 ms |

El `COUNT` de 30 días era claramente más caro, pero el flujo completo seguía siendo sub-segundo y no reproducía el timeout original.

El plan de ejecución mostró además que el `COUNT` realizaba un `Parallel Seq Scan` sobre las transacciones de la ventana temporal y después comprobaba mediante índice cuáles pertenecían a la cuenta concreta.

Esto permitió concluir que el workaround histórico ya no parecía necesario para el síntoma que originalmente pretendía evitar.

La enseñanza importante no es el caso concreto, sino el método:

> Antes de intentar reproducir toda producción, reducir el problema hasta encontrar qué operación escala con el volumen.

## 3. Crear datos sintéticos con volumen similar a producción

Otra estrategia es generar artificialmente un dataset que tenga características parecidas al dataset problemático.

Por ejemplo:

- crear una compañía;
- crear cuentas;
- generar cientos de miles o millones de transacciones;
- mantener una distribución temporal similar;
- reproducir relaciones entre entidades;
- ejecutar después el mismo flujo de aplicación.

No necesitamos copiar los datos reales. Necesitamos copiar **sus propiedades relevantes**.

### Ventajas

- Sin datos sensibles.
- Sin riesgo para producción.
- Podemos modificar y destruir datos libremente.
- Podemos probar volúmenes incluso superiores a los actuales.
- Permite encontrar umbrales de degradación.
- Es fácilmente repetible.
- Puede convertirse en una herramienta reutilizable de testing.

### Limitaciones

El volumen por sí solo no garantiza una reproducción realista.

Un dataset sintético puede diferir de producción en:

- distribución temporal;
- cardinalidad;
- relaciones entre tablas;
- proporción de estados;
- distribución de valores;
- checkpoints;
- estadísticas del planner;
- fragmentación;
- concurrencia.

Por tanto, un buen generador no debería limitarse a crear `N` filas.

Debería intentar modelar la **forma de los datos** que afecta a las consultas.

## 4. Dataset sintético escalable

Una variante especialmente útil consiste en parametrizar el dataset:

```text
small   → 10k movimientos
medium  → 100k movimientos
large   → 1M movimientos
xlarge  → 10M movimientos
```

Esto permite observar cómo evoluciona el rendimiento.

En lugar de preguntar:

> ¿Esta consulta es lenta?

podemos preguntar:

> ¿Cómo escala esta consulta cuando el dataset crece 10x?

Esto permite detectar:

- crecimiento aproximadamente lineal;
- crecimiento superlineal;
- cambios en el query planner;
- aparición de sequential scans;
- puntos donde deja de utilizarse un índice;
- umbrales donde la latencia deja de ser aceptable.

Esta aproximación es especialmente útil para estudiar **scalability**, no únicamente para reproducir incidentes.

## 5. Restaurar un dump de producción en local

Otra posibilidad es restaurar una copia de producción en una base de datos local o aislada.

### Ventajas

- Dataset muy realista.
- Libertad total para experimentar.
- Permite modificar registros, flags e índices.
- No afecta a producción durante las pruebas.

### Problemas

En nuestro entorno esta opción se considera inadecuada por dos motivos.

Primero, generar el dump puede introducir carga significativa sobre la base de datos primaria.

Segundo, implica copiar datos reales de producción a otro entorno, lo que añade riesgos de:

- privacidad;
- seguridad;
- cumplimiento;
- gestión de credenciales;
- persistencia accidental de datos sensibles.

Aunque técnicamente sea una estrategia válida, no debería asumirse que es una opción disponible.

## 6. Restaurar un dump desde una réplica

Si existe una réplica de lectura, el dump puede obtenerse desde ella en lugar de hacerlo desde la primaria.

Esto reduce el impacto sobre la base de datos principal.

### Ventajas

- Dataset real.
- Menor riesgo operativo para la primaria.
- Permite posteriormente experimentar en un entorno aislado.

### Limitaciones

- Sigue existiendo el problema de copiar datos reales.
- La réplica también tiene capacidad limitada.
- Puede existir replication lag.
- Necesitamos acceso a esa infraestructura.

Por tanto, elimina parte del riesgo operativo pero no los riesgos asociados a datos productivos.

## 7. Conectar una aplicación local a la base de datos de producción

Técnicamente podría ejecutarse Core o Control Panel en local apuntando a una base de datos productiva o a una réplica.

Esto permite utilizar el código local sobre datos reales.

Sin embargo, conectar una aplicación local a producción es una práctica especialmente peligrosa.

### Riesgos

- ejecutar accidentalmente escrituras;
- jobs o callbacks inesperados;
- código local diferente del desplegado;
- migraciones;
- scripts o comandos ejecutados contra el entorno equivocado;
- consultas experimentales demasiado caras.

Incluso con acceso read-only, sigue existiendo riesgo de generar carga.

Por estos motivos, en nuestro entorno esta alternativa se descarta.

## 8. Utilizar una réplica de lectura para análisis

Una réplica puede ser útil para:

- `EXPLAIN`;
- análisis exploratorio;
- consultas de volumen;
- validación de índices;
- medición de consultas costosas.

Tiene la ventaja de separar parte de la carga de la primaria.

Pero una réplica tampoco debería tratarse como un sandbox infinito: sigue siendo infraestructura compartida y puede afectar a replicación, reporting u otros consumidores.

## Comparativa

| Estrategia | Fidelidad | Riesgo para producción | Libertad para experimentar | Datos reales |
| --- | --- | --- | --- | --- |
| Flujo completo en producción | Muy alta | Alto | Muy baja | Sí |
| SQL controlado en producción | Alta | Bajo–medio | Baja | Sí |
| SQL sobre réplica | Alta | Bajo | Media | Sí |
| Aplicación local → producción | Alta | Alto | Media | Sí |
| Dump producción → local | Alta | Medio durante el dump | Muy alta | Sí |
| Dump réplica → local | Alta | Bajo–medio | Muy alta | Sí |
| Datos sintéticos | Media–alta | Ninguno | Muy alta | No |

No existe una estrategia universalmente mejor.

La elección depende de **qué propiedad del problema necesitamos reproducir**.

## Estrategia práctica

Una secuencia razonable de investigación es:

### 1. Reducir el problema

Antes de mover grandes cantidades de datos, averiguar:

- qué endpoint falla;
- qué caso de uso ejecuta;
- qué queries genera;
- qué operaciones dependen del volumen.

### 2. Medir con datos reales de forma controlada

Si es seguro, obtener métricas de las consultas relevantes usando producción o una réplica.

El objetivo es identificar el cuello de botella, no hacer pruebas arbitrarias.

### 3. Entender el plan de ejecución

`EXPLAIN` puede revelar:

- sequential scans;
- joins costosos;
- estimaciones erróneas;
- índices utilizados;
- número de filas esperado;
- coste relativo de cada operación.

### 4. Construir un dataset sintético

Una vez entendida la forma del problema, reproducirla localmente:

```text
datos sintéticos
        ↓
volumen equivalente o superior a producción
        ↓
mismo flujo de Core
        ↓
mismo flujo de Control Panel
```

### 5. Buscar el límite

No limitarse a reproducir el volumen actual.

Probar también:

```text
1x volumen actual
2x
5x
10x
```

Así podemos comprobar si estamos corrigiendo el problema actual o diseñando algo capaz de escalar.

## Idea clave

Reproducir producción no significa necesariamente **copiar producción**.

En problemas de rendimiento interesa identificar qué dimensiones del entorno productivo son relevantes y reproducirlas de la forma más segura posible.

A veces basta con analizar las consultas reales.

Otras veces necesitamos reproducir la distribución y volumen mediante datos sintéticos.

Y solo en último término necesitamos ejecutar el flujo real sobre infraestructura productiva.

La pregunta útil no es:

> ¿Cómo puedo tener producción en local?

sino:

> ¿Qué característica de producción necesito reproducir para que aparezca el mismo problema?
