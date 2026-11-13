# Recordatorios Citas Service

![CI](https://github.com/Humbeer06/mogollon-post1-u9/actions/workflows/ci.yml/badge.svg)

## Descripción

Repositorio del post-contenido de la Unidad 9 de Patrones de Diseño de
Software — Sexto Semestre. Un único proyecto Spring Boot que envía
recordatorios de citas por el canal preferido del paciente (correo o
SMS), construido desde el inicio con Strategy (`ReminderChannel`) y
Factory Method (`ReminderChannelFactory`) inyectados por constructor.

## Parte 1 — Pruebas unitarias con JUnit 5 y Mockito

`CitaReminderServiceImplTest` aísla `CitaReminderServiceImpl` de sus
colaboradores. La sección "Decisiones de diseño" justifica por qué cada
uno es un mock, un stub o un fake.

## Parte 2 — Refactor DIP, integración y CI/CD

`enviarRecordatorioUrgente` nació con un `new` hardcodeado que impedía
probarlo; el refactor movió esa construcción a
`config/CanalUrgenteConfig.java` e inyectó la abstracción por
constructor. `CitaRepositoryTest` (`@DataJpaTest`) y
`CitaReminderServiceIntegrationTest` (`@SpringBootTest`) prueban la
persistencia y el flujo completo con H2 y un reloj fijo. El pipeline
`.github/workflows/ci.yml` ejecuta `mvn verify` en cada push y pull
request a `main` y publica el reporte de cobertura de JaCoCo como
artefacto.

## Decisiones de diseño

### Qué patrón facilitó qué prueba

| Patrón o práctica | Prueba que lo aprovecha | Por qué sin él sería más difícil |
|---|---|---|
| Strategy (`ReminderChannel`) | `enviarRecordatoriosPendientes_citaEnVentana_usaElCanalPreferido` | Un solo `@Mock ReminderChannel` basta para verificar el envío de cualquier canal. Sin la interfaz, la lógica SMTP o HTTP estaría en línea dentro del servicio y habría que simular bibliotecas de bajo nivel. |
| Factory Method (`ReminderChannelFactory`) | la misma prueba, y `enviarRecordatoriosPendientes_sinCitasPendientes_noConsultaLaFactory` | La elección del canal queda detrás de una interfaz. En la primera prueba el stub `obtenerCanal(EMAIL)` entrega el canal mockeado, y si el servicio pidiera otro tipo el mock devolvería `null` y la prueba fallaría. En la segunda, `verifyNoInteractions` demuestra que sin citas no se selecciona ningún canal. Con un `switch` embebido en el servicio, esa decisión solo se observaría de forma indirecta. |
| `Clock` inyectado | `enviarRecordatoriosPendientes_ventanaSegunClockFijo_consultaLosLimitesExactos` | Con `Clock.systemDefaultZone()` la ventana de 24 horas cambiaría en cada ejecución y la aserción sobre los límites no sería reproducible. |
| DIP en `canalUrgente` | `enviarRecordatorioUrgente_canalFalla_noMarcaComoEnviado` | Con el `new` dentro del método no había dónde sustituir el canal para simular un fallo. |
| H2 en memoria | `CitaRepositoryTest` y `CitaReminderServiceIntegrationTest` | Permite probar la consulta derivada y el flujo completo sin infraestructura externa. |

### Mock, stub o fake para cada colaborador (Parte 1)

- **`CitaRepository` → Mock.** Hay que controlar lo que devuelve la
  consulta y también verificar que `save()` se invoca una vez con la
  cita marcada como enviada. Cuando una prueba necesita las dos cosas,
  un simple stub no alcanza.
- **`ReminderChannelFactory` y `ReminderChannel` → Mock.** La selección
  de estrategia es el comportamiento bajo prueba. La factory se stubbea
  para entregar el canal de la cita, y es el canal quien verifica haber
  recibido el mensaje con `verify(...).enviar(...)`. En la prueba de
  "sin citas" la factory se usa además con `verifyNoInteractions`.
- **`Clock` → Fake, no Mock.** `Clock.fixed(...)` es una implementación
  completa del JDK que se comporta como la real con el instante
  congelado. Mockearlo exigiría stubbear `instant()` y `getZone()` y no
  garantizaría que `LocalDateTime.now(clock)` se comporte como en
  producción.

### Diagnóstico antes del refactor (Parte 2)

La primera versión de `enviarRecordatorioUrgente` (commit
`feat(service): agregar enviarRecordatorioUrgente con new hardcodeado`)
no se podía probar por esta línea:

```java
ReminderChannel canalUrgente = new SmsReminderChannel(new TwilioNotificadorSmsGateway());
```

El método construía él mismo el canal y el gateway, así que ninguna
prueba podía sustituirlos por un test double. `enviarRecordatoriosPendientes()`,
en cambio, nunca construye sus colaboradores: recibe el repositorio, la
factory y el reloj por constructor, y por eso la Parte 1 pudo aislarlo
por completo.

### Antes y después del refactor

Antes, dentro de `CitaReminderServiceImpl`:

```java
ReminderChannel canalUrgente = new SmsReminderChannel(new TwilioNotificadorSmsGateway());
canalUrgente.enviar(cita, MENSAJE_URGENTE);
```

Después, el único `new` vive en el composition root
(`CanalUrgenteConfig`) y el servicio recibe la abstracción:

```java
@Bean
public ReminderChannel canalUrgente(NotificadorSmsGateway gateway) {
    return new SmsReminderChannel(gateway);
}
```

```java
public CitaReminderServiceImpl(CitaRepository citaRepository,
                               ReminderChannelFactory reminderChannelFactory,
                               @Qualifier("canalUrgente") ReminderChannel canalUrgente,
                               Clock clock)
```

La prueba `enviarRecordatorioUrgente_canalFalla_noMarcaComoEnviado`
simula el fallo en el canal inyectado (no en el gateway, que no se
ejercita aquí) y comprueba que la cita no se guarda ni queda marcada como
enviada. Antes del refactor esa prueba no se podía escribir.

### H2 real pero gateway mockeado

En `CitaReminderServiceIntegrationTest` conviven una base de datos real
en memoria y un `@MockBean` sobre `NotificadorSmsGateway`. H2 es un fake
aceptable porque su contrato SQL/JPA es suficientemente cercano al de una
base real y no necesita infraestructura. Un proveedor de SMS externo no
tiene un equivalente gratuito y determinista, y simularlo evita depender
de credenciales, de red y de su disponibilidad en cada ejecución del
pipeline.

### CI sin contenedor de base de datos

El workflow no declara servicios de base de datos porque las pruebas de
integración usan H2 en memoria. Probar contra PostgreSQL con
Testcontainers exigiría Docker en el runner y más tiempo de arranque.

## Ajustes deliberados sobre la guía

La guía incluye código de referencia que, seguido literalmente, no
compila o no se comporta como describe. Se corrigió así:

- **Bean `clock` duplicado.** El `@TestConfiguration` de la prueba de
  integración definía un bean llamado `clock`, igual que
  `ClockConfig.clock()`. Spring Boot desactiva por defecto la
  sobrescritura de beans y el contexto no arrancaría. El bean de prueba
  se llama `clockFijo` y gana por `@Primary`.
- **Import faltante.** Las pruebas de `enviarRecordatorioUrgente`
  usan `Optional` sin el import en la lista de imports de la guía.
- **Commit que no compilaba.** El primer commit del Paso 12 de la guía
  agregaba el método a la interfaz sin tocar la implementación. Aquí ese
  commit incluye también la implementación con `new` hardcodeado, que es
  lo que su mensaje describe. `CitaNoEncontradaException` se introduce en
  ese mismo commit de la Parte 2, que es donde se usa por primera vez.
- **Afirmación del README sin respaldo.** El README de ejemplo decía que
  la selección de canal se comprueba con
  `verify(reminderChannelFactory).obtenerCanal(EMAIL)`, pero ninguna
  prueba lo hace. La tabla de arriba describe lo que las pruebas hacen de
  verdad.
- **Cobertura.** La rúbrica pide el reporte de cobertura y la guía no
  configura JaCoCo. Se agregó el plugin (reporte en
  `target/site/jacoco/index.html`, sin umbral mínimo) y el workflow lo
  publica como artefacto.
- **Comentarios.** Se eliminaron los comentarios explicativos del código
  de ejemplo; la explicación vive en este README.

## Ejecutar las pruebas

```bash
mvn test      # pruebas unitarias
mvn verify    # unitarias + integración + reporte JaCoCo
```

Reporte de cobertura: `target/site/jacoco/index.html`.

## Evidencia

Ocho pruebas automatizadas: seis unitarias en
`CitaReminderServiceImplTest`, una `@DataJpaTest` en
`CitaRepositoryTest` y una `@SpringBootTest` en
`CitaReminderServiceIntegrationTest`. Las capturas están en `capturas/`.

## Herramientas utilizadas

- Java 21, Spring Boot 3.3, Spring Data JPA, H2, Lombok
- JUnit 5, Mockito, JaCoCo
- Apache Maven, GitHub Actions, Git, GitHub

## Conclusiones

La testabilidad del servicio no vino de Mockito sino de las decisiones
de diseño previas: Strategy, Factory e inyección por constructor dejaron
puntos de sustitución donde cada prueba pudo colocar su test double. El
caso de `enviarRecordatorioUrgente` lo mostró de forma directa: la misma
lógica era imposible de probar con un `new` dentro del método y trivial
de probar después de moverlo al composition root. Elegir entre mock,
stub y fake también obligó a preguntarse qué se quiere verificar de cada
colaborador, y no solo cómo reemplazarlo. Por último, revisar el código
de ejemplo de la guía antes de usarlo evitó errores que solo habrían
aparecido al ejecutar el contexto de Spring.
