# LCMF Documentation Draft

`LCMF` (`Loosely Coupled Modular Frontend`) is an architectural approach to
building a frontend system from many small modules with explicit boundaries.

In `LCMF`, inter-module cooperation stays explicit in two main ways:

- through messages on the `bus`;
- through narrow public synchronous reads via `registry`.

This repository is a working documentation space for `LCMF`.
Its purpose is not to add a decorative theory layer on top of code, but to make
concrete a practical way of building frontend systems that can stay readable as
they grow.

`LCMF` is developed alongside [`lcmm-docs`](https://github.com/algebrain/lcmm-docs),
which describes the related `LCMM` backend architecture.
The two documentation sets are not identical, but they are meant to stay aligned
where frontend and backend contracts meet.

The main value of `LCMF` is not novelty for its own sake.
It is an attempt to keep important architectural decisions visible:

- who owns state;
- how modules coordinate with each other;
- where synchronous reads are allowed;
- how HTTP and realtime flows are handled;
- how the application is assembled as a whole without hidden coupling.

What is here:

- documentation drafts in [`docs/`](./docs);
- working critique and proposed documents in `.answers/`.

## Where To Start

It is better to begin with a reading path for your role than to jump randomly
between files.

- for architects: [docs/READING_ORDER_ARCH.md](./docs/READING_ORDER_ARCH.md)
- for application-level developers: [docs/READING_ORDER_APP.md](./docs/READING_ORDER_APP.md)
- for module-level developers: [docs/READING_ORDER_MODULE.md](./docs/READING_ORDER_MODULE.md)

If you want the shortest possible summary of `LCMF`, it is this:

- the frontend is built from many small modules;
- each module owns its own state and behavior;
- inter-module coordination happens primarily through the `bus`;
- public synchronous reads through `registry` stay a narrow supporting channel;
- app-level assembly should remain understandable as a whole and should not hide
  architectural decisions in scattered glue code.
