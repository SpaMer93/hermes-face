# Hermes Face

Mi agente tenía voz, pero no cara. Quería que la tuviera, y tenía un teléfono Android guardado en un cajón. Este proyecto es eso: un dispositivo dedicado que se queda encendido mostrando una cara que escucha, piensa y contesta en voz alta.

La cara no es un video ni un GIF en bucle. Son óvalos y una boca dibujados por código, con una capa de física encima que les da inercia. Parpadea, respira y se mueve sola, y cuando cambia de emoción no salta: se estira hasta llegar.

![Catálogo de expresiones](media/catalogo-emociones.png)

> Proyecto en pausa. El camino funciona y está medido; la voz no quedó como quería. Al final cuento por qué.

## Qué hace hoy

Le hablo y me escucha. El audio viaja al host, se transcribe, el modelo contesta, se sintetiza la voz y regresa al teléfono mientras la cara reacciona. Un turno completo, medido en un moto g20 real:

| Etapa | Tiempo |
|---|---|
| Escucha y transcripción | 3,088 ms |
| Modelo (camino directo) | 136 ms |
| Síntesis de voz | 1,517 ms |
| **Turno total** | **4,768 ms** |

## La medición que cambió el diseño

Al principio cada turno tardaba veinte segundos. El problema no era el teléfono ni la red: el *core* arrancaba un agente completo por cada frase, con su prompt, sus herramientas y su planeación. Para contestar "hola" no hace falta todo eso.

Lo cambié por una llamada directa al modelo, dejando el agente como respaldo, y medí los dos caminos con tres turnos cada uno:

| Camino | Turno 1 | Turno 2 | Turno 3 | Media |
|---|---|---|---|---|
| Agente completo | 18 898 ms | 20 160 ms | 22 778 ms | **20 612 ms** |
| Directo al modelo | 177 ms | 130 ms | 159 ms | **155 ms** |

Son 133 veces más rápido en el core. El respaldo se probó en vivo forzando una llave inválida: 17,6 s y el audio salió igual, así que si el camino rápido falla la cara no se queda muda.

De paso apareció un detalle que costó una tarde: un proveedor rechazaba las llamadas directas con 403 porque Cloudflare bloqueaba el `User-Agent` de `Python-urllib`. La solución fue ponerle un User-Agent propio a la cara.

## Dónde corre esto, y por qué importa

Los tiempos de arriba no salieron del monstruo. Salieron de **renascor**, una laptop vieja sin tarjeta gráfica dedicada, que es donde vive el agente ahora mismo porque puede quedarse encendida todo el día.

Al principio todo se probaba en mi PC, con una RTX 4070 Ti de 12 GB. La diferencia es enorme: la PC no puede estar prendida todo el tiempo, y renascor no tiene GPU, así que el trabajo de modelos ahí se hace en el procesador. Los números de un lado y del otro no se comparan, y no porque algo esté mal hecho: son dos máquinas distintas haciendo el mismo trabajo.

Es la misma regla que persiguen todos mis proyectos, y aquí se nota más que en ninguno: **hacer más con menos**. Sin suscripciones de quince dólares al mes que se acumulan, buscando modelos locales o de bajo costo, aceptando que a veces eso significa esperar un poco más.

## Arquitectura

```mermaid
graph LR
  A["Teléfono Android<br/>Kotlin + Compose"] -->|"frames y audio<br/>TLS + pairing"| B["Bridge en el host<br/>servicio autenticado"]
  B --> C{"Core"}
  C -->|"camino rápido"| D["Modelo directo<br/>chat/completions"]
  C -->|"respaldo"| E["Agente completo<br/>con herramientas"]
  D --> B
  E --> B
  B -->|"voz sintetizada"| A
```

## Tres decisiones que explican el código

**La lógica de la cara vive fuera de Android.** Geometría, física, máquina de estados y comportamiento están en un módulo sin dependencias de plataforma, así que se prueba en el host sin teléfono conectado. Ajustar una expresión no requiere instalar nada en el dispositivo, y la mayoría de las pruebas corren en segundos.

**Las emociones no las inventa el modelo.** El texto que contesta el agente pasa por un reductor que decide qué expresión corresponde según señales explícitas. Si el modelo dice algo raro, la cara no se pone a improvisar gestos.

**Los contratos son la fuente de verdad.** Los esquemas de mensajes, configuración y ejemplos viven en su propia carpeta y el código consume copias de ahí. Cuando el teléfono y el host hablan, hablan de eso, no de un acuerdo implícito que se rompe en el siguiente cambio.

## Estado: pausado, no terminado

Voy a decirlo sin adornos: **este proyecto está inconcluso y en pausa**.

La idea era que la cara hablara con la voz de Cortana. Ese detalle era un capricho geek, y lo sostengo: sin esa voz la temática se cae un poco. Probamos clonación de voz local y el resultado no fue fiel a la muestra. Para acercarme de verdad necesitaba la 4070 Ti encendida o pagar ElevenLabs, y ninguna de las dos cosas encaja con la regla de hacer más con menos. Así que preferí dejarlo descansar antes que hipotecar el proyecto a una suscripción.

Tampoco llegamos a las pruebas que de verdad importan: trabajo real, solicitudes, conversaciones largas y tendidas, de esas en las que el agente se equivoca y hay que ver cómo se recupera. Lo que sí está verificado es el camino: el pipeline completo corre de punta a punta, con voz sintetizada, sobre un teléfono real.

Con mis herramientas actuales no conseguí el resultado que quería, y no pasa nada. A veces las cosas no funcionan y lo que queda es lo aprendido: cómo se mide una latencia por etapas, dónde se va el tiempo, y que un agente completo por frase es un lujo innecesario cuando solo quieres que alguien te conteste.

No me rendí con esto. Sigo buscando alternativas para que la voz refleje la personalidad completa de Hermes, sin gastar un dineral en el proceso. Cuando aparezca una que no me obligue a elegir entre calidad y libertad, este proyecto vuelve.

## Sobre el código

Repositorio de exhibición: documenta el diseño, las mediciones y las decisiones. El código de la app, del bridge y los APK no se distribuyen.

© 2026 Javier Cuevas. Todos los derechos reservados.
