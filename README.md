# TDStrudelSync

TDStrudelSync es un componente para TouchDesigner que permite sincronizar visuales en tiempo real con eventos generados desde Strudel usando WebSockets.

A diferencia de enfoques tradicionales, no intenta reconstruir el reloj de audio, sino que utiliza directamente los timestamps futuros programados del evento, logrando una sincronización precisa y robusta.

## Características

* Recepción de eventos /dirt/play vía WebSocket
* Scheduler basado en tiempo absoluto
* Cola ordenada por timestamp_ms
* Soporte para eventos futuros (lookahead del audio)
* API programática (public_api)
* Sistema de diagnóstico en tiempo real
* Totalmente desacoplado de visuales

## Ejemplo

En la carpeta ejemplo puedes ver un proyecto configurado. 

<img width="906" height="618" alt="image" src="https://github.com/user-attachments/assets/53628297-a98b-4086-910d-b0c396c4b0cc" />

<img width="1693" height="1229" alt="image" src="https://github.com/user-attachments/assets/2ed98652-8bea-46f7-94a5-f25f8f482b3c" />



Puedes usar uno de estos dos ejemplos en Strudel.cc:

```js
setcps(0.5)

const pat = s("[bd*2 sd hh oh]").bank("tr909")

$: stack(
  pat.gain('1'),
  pat.osc()
)
```

```js
setcps(0.5)

let bassLine = note("<[c2 c3]*4 [bb1 bb2]*4 [f2 f3]*4 [eb2 eb3]*4>").sound("gm_synth_bass_1")
  .lpf(slider(1000, 0, 1000))

let leadLine = n(`<
[~ 0] 2 [0 2] [~ 2]
[~ 0] 1 [0 1] [~ 1]
[~ 0] 3 [0 3] [~ 3]
[~ 0] 2 [0 2] [~ 2]
>*4`).scale("C4:minor")
.sound("gm_synth_strings_1")

let drumLine = sound("bd*4, [~ <sd cp>]*2, [~ hh]*4")
.bank("tr909")

let hhLine = sound("hh*16").gain(sine)

$: stack(
  bassLine,
  bassLine.osc()
)

$: stack(
  leadLine,
  leadLine.osc()
)

$: stack(
  drumLine,
  drumLine.osc()
)

_$: stack(
  hhLine,
  hhLine.osc()
)
```

## Arquitectura

```
Strudel
   ↓ WebSocket
TDStrudelSync
   ↓ onStrudelEvent(ev)
Visual Adapter (usuario)
   ↓
Sistema visual (TOPs / CHOPs)
```

## Instalación

1. Copia TDStrudelSync.tox a tu proyecto
2. Arrástralo a la red de TouchDesigner
3. Abre el DAT `setup_component`
4. Ejecuta: `Right Click → Run Script`

## Configuración

### Network

| Parámetro          | Descripción                       |
| ------------------ | --------------------------------- |
| `Active`           | Activa/desactiva el servidor      |
| `Port`             | Puerto WebSocket                  |
| `Addressfilter`    | Filtro de mensajes (`/dirt/play`) |
| `Eventcallbackdat` | DAT que recibe los eventos        |
| `Autostart`        | Inicio automático                 |

### Timing

| Parámetro          | Descripción                       |
| ------------------ | --------------------------------- |
| `Active`           | Activa/desactiva el servidor      |
| `Port`             | Puerto WebSocket                  |
| `Addressfilter`    | Filtro de mensajes (`/dirt/play`) |
| `Eventcallbackdat` | DAT que recibe los eventos        |
| `Autostart`        | Inicio automático                 |

### Debug

| Parámetro         | Descripción     |
| ----------------- | --------------- |
| `Debug`           | Activa logs     |
| `Clearqueue`      | Vacía la cola   |
| `Resetall`        | Reset completo  |

### Uso básico

1. Crear callback

* Crea un Text DAT, llamado event_callbacks, en tu proyecto y configur el contenido del lenguaje en python:

<img width="396" height="206" alt="image" src="https://github.com/user-attachments/assets/f305b92d-d5f0-472c-be5d-78bf3af4cfae" />

```py
def onStrudelEvent(ev):
    print(ev['sound'], ev['delta'])
```

2. Conectar callback

En la página Network de los parámetros de TDStrudelSync configura el callback: `/project1/strudel_visual_adapter/event_callbacks`

3. Usar eventos

```py
def onStrudelEvent(ev):
    sound = ev['sound']
    delta = ev['delta']

    if sound == 'tr909bd':
        op('kick_timer').par.start.pulse()
```

## Estructura de un evento

```js
{
    'id': int,
    'address': str,
    'timestamp_ms': float,
    'timestamp_sec': float,
    'recv_ms': float,
    'lead_ms': float,

    'sound': str,
    'delta': float,
    'cycle': float,
    'cps': float,
    'bank': str,

    'params': dict
}
```

## Campos clave

`timestamp_ms`: momento exacto en que el evento debe ocurrir.
`lead_ms`: responde a la pregunta cuánto antes llegó el evento. Un valor mayor a cero es un valor normal.

`params`: contiene todos los datos originales:

```py
ev['params']['cps']
ev['params']['cycle']
ev['params']['delta']
```

## Modelo de sincronización

```py
disparar cuando:
local_now_ms() + Latencybiasms >= timestamp_ms
```

## Ajustes de sincronización

| Problema          | Solución                    |
| ----------------- | --------------------------- |
| Visual adelantada | `Latencybiasms = -10 → -40` |
| Visual atrasada   | `Latencybiasms = +5 → +30`  |

## API (Textport)

Esta API se puede invocar desde el Textport

```py
api = op('/project1/TDStrudelSync/public_api').module
```

Limpiar cola:

```py
api.clearQueue()
```

Reset:

```py
api.reset()
```

Diagnóstico:

```py
print(api.getDiagnostics())
```

## Buenas prácticas

Separar responsabilidades:

| Componente     | Rol            |
| -------------- | -------------- |
| TDStrudelSync  | sincronización |
| Visual Adapter | lógica visual  |

## Problemas comunes

No llegan eventos:

* Verifica puerto
* Verifica conexión WebSocket
* Verifica Addressfilter

Eventos desincronizados: 

* Ajusta Latencybiasms

La cola crece:

* Revisar Maxqueuelen
* Verificar scheduler activo

