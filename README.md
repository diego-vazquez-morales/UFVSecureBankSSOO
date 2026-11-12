# UFV SecureBank

Sistema bancario multiproceso en C para Linux, desarrollado para la asignatura de *Sistemas Operativos* del Grado en Ingeniería Informática de la Universidad Francisco de Vitoria (2026).

Los clientes se conectan por TCP y cada sesión se ejecuta en su propio proceso. Los procesos comparten los datos de las cuentas de forma segura y envían su actividad a un monitor que detecta comportamientos sospechosos y puede bloquear cuentas en tiempo real.

**Tecnologías:** C · POSIX (procesos, pipes, colas de mensajes, semáforos con nombre, hilos, señales, sockets) · Linux

## Funcionalidades

- **Cuentas en tres divisas** (EUR, USD, GBP): ingresos, retiradas, transferencias entre cuentas y cambio de divisa.
- **Límites configurables** por divisa para retiradas y transferencias, y tipos de cambio, leídos desde `config.txt`.
- **Detección de fraude**: un proceso monitor detecta retiradas excesivas y transferencias repetitivas, y el banco bloquea la cuenta afectada.
- **Registro de transacciones** en `transacciones.log`.
- **Acceso remoto**: los clientes se conectan por TCP (puerto definido en `config.txt`) e inician sesión con su número de cuenta o crean una nueva.

## Arquitectura

```
            Clientes TCP
                 │
        ┌────────▼────────┐  fork + exec   ┌──────────────┐
        │ banco (principal)│──────────────▶│   monitor    │
        │ acepta conexiones│◀── alertas ───│  detecta     │
        │ escribe el log   │   (cola)      │  anomalías   │
        └──┬──────────▲───┘                └──────▲───────┘
  fork+exec│          │ entradas de log (cola)    │ operaciones (cola)
  por cliente         │                           │
        ┌──▼──────────┴───────────────────────────┴──┐
        │   usuario (un proceso por sesión)          │
        │   stdin/stdout redirigidos al socket       │
        └────────────────────────────────────────────┘
```

- **Procesos:** `banco` crea el `monitor` y un proceso `usuario` por cliente (`fork` + `execv`), y los recoge con `waitpid`.
- **Comunicación entre procesos:** tres colas de mensajes POSIX (operaciones → monitor, log → banco, alertas → banco) y un pipe por hijo para enviar bloqueos de cuenta.
- **Sincronización:** semáforos POSIX con nombre protegen el fichero de cuentas y la configuración.
- **Sockets:** la entrada y salida de cada sesión se redirige a su socket con `dup2`.
- **Señales:** `SIGINT`/`SIGTERM` provocan un cierre limpio que detiene los hijos y elimina las colas y semáforos.

## Compilar y ejecutar

```bash
gcc -o banco   banco.c   -lrt -pthread
gcc -o usuario usuario.c -lrt -pthread
gcc -o monitor monitor.c -lrt -pthread

./banco
```

Desde otra terminal, conéctate como cliente:

```bash
nc localhost 8080
```

## Ficheros

| Fichero | Función |
|---|---|
| `banco.c` | Proceso principal: servidor, gestión de procesos, log y alertas |
| `usuario.c` | Sesión del cliente: operaciones sobre la cuenta |
| `monitor.c` | Detección de anomalías |
| `banco_comun.h` | Constantes, estructuras y nombres de colas y semáforos compartidos |
| `config.txt` | Límites, umbrales, tipos de cambio y puerto |
