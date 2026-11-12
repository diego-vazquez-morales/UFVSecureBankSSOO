# UFV SecureBank

Multi-process banking system in C for Linux, built for the *Operating Systems* course in Computer Engineering at Universidad Francisco de Vitoria (2026).

Clients connect over TCP and each session runs in its own process. The processes share account data safely and report activity to a monitor that detects suspicious behaviour and can block accounts in real time.

**Stack:** C · POSIX (processes, pipes, message queues, named semaphores, threads, signals, sockets) · Linux

## Features

- **Accounts in three currencies** (EUR, USD, GBP): deposit, withdraw, transfer between accounts and convert between currencies.
- **Configurable limits** per currency for withdrawals and transfers, and exchange rates, read from `config.txt`.
- **Fraud detection**: a monitor process flags excessive withdrawals and repeated transfers and the bank blocks the affected account.
- **Transaction log** written to `transacciones.log`.
- **Remote access**: clients connect over TCP (port set in `config.txt`), log in with an account number or create a new account.

## Architecture

```
            TCP clients
                 │
        ┌────────▼────────┐  fork + exec   ┌──────────────┐
        │   banco (main)  │───────────────▶│   monitor    │
        │  accepts conns, │◀── alerts ─────│  detects     │
        │  writes the log │   (mq)         │  anomalies   │
        └──┬──────────▲───┘                └──────▲───────┘
  fork+exec│          │ log entries (mq)          │ operations (mq)
  per client          │                           │
        ┌──▼──────────┴───────────────────────────┴──┐
        │   usuario (one process per session)        │
        │   stdin/stdout redirected to the socket    │
        └────────────────────────────────────────────┘
```

- **Processes:** `banco` forks the `monitor` and one `usuario` process per client (`fork` + `execv`), and reaps them with `waitpid`.
- **IPC:** three POSIX message queues (operations → monitor, log → bank, alerts → bank), plus a pipe per child to push account blocks.
- **Synchronization:** named POSIX semaphores guard the
