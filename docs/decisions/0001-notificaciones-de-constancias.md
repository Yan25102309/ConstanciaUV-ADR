
---
status: "proposed"
date: 2026-11-03
---

# Separación del procesamiento de notificaciones de constancias

## Context and Problem Statement

ConstanciaUV necesita comunicar la generación de constancias digitales con un proveedor externo de correo electrónico para notificar a los estudiantes cuando sus documentos estén disponibles. El problema consiste en decidir cómo organizar esta comunicación para que la generación y el almacenamiento de constancias no dependan de la disponibilidad del proveedor externo.

Además, las notificaciones pendientes deben conservarse para futuros intentos de envío y el sistema debe prevenir solicitudes duplicadas para una misma constancia. Por lo tanto, se necesita una solución que permita mantener la continuidad del servicio y controlar de manera independiente el procesamiento de las notificaciones.

## Decision Drivers

* **Continuidad del servicio:** la generación y el almacenamiento de constancias deben funcionar aunque el proveedor externo de correo no esté disponible, de acuerdo con RT2.

* **Confiabilidad de las notificaciones:** las notificaciones pendientes deben conservarse para intentar enviarlas posteriormente y se deben prevenir solicitudes duplicadas para una misma constancia, de acuerdo con RT3 y RT4.

## Considered Options

* Alternativa A: Comunicación directa con recuperación de fallos.
* Alternativa B: Procesamiento asíncrono mediante eventos persistidos.

## Decision Outcome

Chosen option: "Alternativa B: Procesamiento asíncrono mediante eventos persistidos", porque permite separar la generación de constancias del envío de notificaciones. Después de almacenar una constancia, la aplicación registra una notificación pendiente y un componente independiente se encarga de procesarla y comunicarse con el proveedor externo.

Esta alternativa permite que la generación y el almacenamiento de documentos continúen funcionando aunque el proveedor de correo no esté disponible. También facilita conservar las notificaciones pendientes, administrar los reintentos y prevenir duplicados mediante identificadores únicos y mecanismos de idempotencia.

### Consequences

* Good, because la generación y el almacenamiento de constancias pueden continuar sin depender de la disponibilidad del proveedor externo, cumpliendo RT2.
* Good, because las notificaciones pendientes permanecen almacenadas para intentar enviarlas posteriormente, cumpliendo RT3.
* Good, because permite implementar restricciones de unicidad y mecanismos de idempotencia para prevenir solicitudes duplicadas, cumpliendo RT4.
* Good, because los cambios en la comunicación con el proveedor pueden aislarse en el componente encargado de la integración externa.
* Bad, because se necesita desarrollar y mantener un componente independiente para procesar las notificaciones.
* Bad, because el envío no es inmediato y puede existir un retraso entre la generación de la constancia y la recepción del correo.
* Bad, because se requiere administrar los estados de las notificaciones, los reintentos y los posibles errores de procesamiento.
* Trade-off, se acepta una mayor complejidad de implementación a cambio de reducir la dependencia entre la generación de constancias y el proveedor externo.

### Confirmation

En una implementación futura se realizarían las siguientes pruebas y revisiones para comprobar el cumplimiento de esta decisión:

* **Prueba de disponibilidad:** simular que el proveedor de correo no responde y solicitar una constancia. Se comprobaría que el documento se genere, almacene y aparezca en el historial sin interrupciones.
* **Prueba de persistencia:** verificar que la notificación permanezca registrada con estado pendiente cuando no sea posible enviarla.
* **Prueba de recuperación:** restablecer la disponibilidad del proveedor y comprobar que el componente independiente procese las notificaciones pendientes.
* **Prueba de duplicados:** registrar varios intentos de notificación para una misma constancia, incluso de manera simultánea, y comprobar que exista un único registro lógico mediante una restricción de unicidad sobre el identificador de la constancia y el tipo de notificación.
* **Prueba de idempotencia:** ejecutar varias veces el procesamiento de un mismo registro y comprobar que no se creen solicitudes de notificación adicionales. También se deberá revisar el comportamiento cuando ocurra un tiempo de espera y no se conozca si el proveedor aceptó el mensaje.
* **Revisión de arquitectura:** comprobar que el componente de generación de constancias no dependa directamente de la interfaz del proveedor y que la comunicación externa esté encapsulada en el componente de integración.

## Pros and Cons of the Options

### Alternativa A: Comunicación directa con recuperación de fallos

* Good, porque permite intentar enviar la notificación inmediatamente después de almacenar la constancia.
* Good, porque no requiere necesariamente un componente independiente para iniciar el procesamiento.
* Bad, porque mantiene una dependencia directa entre la aplicación y el proveedor externo.
* Bad, porque la lógica de recuperación de fallos y administración de pendientes debe incorporarse al flujo de comunicación directa.
* Bad, porque los cambios en la interfaz del proveedor pueden afectar directamente la integración de la aplicación.

### Alternativa B: Procesamiento asíncrono mediante eventos persistidos

* Good, porque separa las responsabilidades de generación de constancias y envío de notificaciones.
* Good, porque permite conservar los registros pendientes y procesarlos cuando el proveedor vuelva a estar disponible.
* Good, porque facilita aislar la comunicación externa mediante un componente de integración.
* Good, porque permite administrar los reintentos sin interrumpir el proceso principal de generación de documentos.
* Bad, porque requiere desarrollar y mantener un componente adicional de procesamiento.
* Bad, porque introduce complejidad relacionada con los reintentos, los estados de los eventos y la prevención de duplicados.
* Bad, porque las notificaciones pueden llegar con cierto retraso debido al procesamiento asíncrono.

## More Information

La decisión se mantiene como propuesta académica y podrá revisarse si cambian las condiciones de integración con el proveedor externo o los requisitos de notificación.

### Revisión 1. Cambio en el proveedor de notificaciones

El proveedor externo de notificaciones modificó su interfaz de comunicación y la forma en que reporta los errores de envío.

Este cambio afecta principalmente al componente independiente encargado de procesar las notificaciones y al componente de integración que se comunica con el proveedor, ya que deben adaptarse para interpretar correctamente la nueva interfaz y sus respuestas de error.

La alternativa B permite localizar el cambio en el componente de integración externa, evitando modificar directamente el proceso de generación y almacenamiento de constancias. Por esta razón, la decisión original se mantiene y no es necesario seleccionar una alternativa diferente.

Sin embargo, se identifica el riesgo de interpretar incorrectamente los nuevos errores, provocar reintentos innecesarios o dejar notificaciones pendientes sin un procesamiento adecuado.

Como consecuencia, será necesario actualizar el mecanismo de interpretación de errores y revisar las condiciones bajo las cuales se realizan los reintentos.

Se agrega una nueva comprobación: realizar pruebas de integración con los diferentes tipos de respuesta y error de la nueva interfaz del proveedor, verificando que los estados de las notificaciones se actualicen correctamente y que los reintentos no generen solicitudes duplicadas.

La decisión podrá evaluarse nuevamente si los cambios del proveedor introducen restricciones que afecten la viabilidad de esta integración.
