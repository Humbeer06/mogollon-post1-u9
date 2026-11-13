# Recordatorios Citas Service

## Descripción

Parte 1 del post-contenido de la Unidad 9 de Patrones de Diseño de
Software. Servicio Spring Boot que envía recordatorios de citas por el
canal preferido del paciente (correo o SMS), construido con Strategy
(`ReminderChannel`) y Factory Method (`ReminderChannelFactory`)
inyectados por constructor.

## Decisiones de diseño

**Mock, Stub o Fake para cada colaborador de `CitaReminderServiceImpl`.**

- `CitaRepository` → Mock. Hay que controlar lo que devuelve la consulta
  y además verificar que `save()` se invoca una vez con la cita marcada
  como enviada.
- `ReminderChannelFactory` y `ReminderChannel` → Mock. La selección de
  canal es el comportamiento bajo prueba: la factory se stubbea para
  entregar el canal de la cita y es el canal el que verifica haber
  recibido el mensaje.
- `Clock` → Fake. `Clock.fixed(...)` es una implementación completa del
  JDK que se comporta como la real con el instante congelado; mockearlo
  exigiría stubbear `instant()` y `getZone()` y aun así no garantizaría
  que `LocalDateTime.now(clock)` se comporte como en producción.

## Ejecutar las pruebas

```bash
mvn test
```
