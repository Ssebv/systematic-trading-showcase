# Systematic Trading Research Platform

*🇬🇧 [English version](README.md)*

> Infraestructura de trading algorítmico multi-estrategia construida para **saber la verdad
> sobre un edge antes de arriesgar capital** — no para presumir un P&L inventado.


---

## TL;DR

Diseñé y operé de punta a punta un sistema sistemático de trading (12 estrategias sobre
cripto + prediction markets, 20 procesos en un VPS) que ejecuta contra venues reales en modo
paper para **validar edge ejecutable antes de comprometer dinero**. El énfasis no está en
"un bot que gana", sino en la **infraestructura de rigor**: backtesting multi-fase, simulación
de ejecución con slippage real, gestión de riesgo en capas, y disciplina estadística para
**matar con datos las estrategias que no sobreviven a los costes**.

**El punchline:** la mayoría de edges retail se compiten a escala minorista. Construí el sistema
que me permitió *demostrarlo con datos* y actuar en consecuencia — cosechar el único edge robusto,
y parar de invertir en los que no pagan. Disciplina intelectual sobre P&L de vanidad.

---

## Arquitectura

```
                         ┌───────────────────────────────┐
                         │        VPS (Docker, hardened)  │
                         │   read-only · non-root · CI    │
                         └───────────────┬───────────────┘
                                         │ supervisord (20 procesos)
        ┌────────────────────┬───────────┼────────────────────┬────────────────────┐
        ▼                    ▼            ▼                    ▼                    ▼
 ┌────────────┐      ┌────────────┐ ┌──────────┐      ┌──────────────┐    ┌──────────────┐
 │ Estrategias│      │ Prediction │ │  Macro   │      │  Capa de      │    │ Observabilidad│
 │  cripto    │      │  markets   │ │  context │      │  RIESGO       │    │  + alerting   │
 │ (spot/fut) │      │ (copy-trade│ │ (radar)  │      │               │    │               │
 └─────┬──────┘      │  wallets)  │ └──────────┘      │ circuit break.│    │ dashboard RT  │
       │             └─────┬──────┘                   │ forward-guard │    │ 31 watchers   │
       │                   │                          │ (auto-retiro) │    │ → Telegram    │
       │                   │                          │ sizing/gates  │    │ auto-auditor  │
       ▼                   ▼                          │ daily CB      │    │ ledgers SQL   │
 ┌──────────────────────────────┐                    └───────┬───────┘    └──────┬────────┘
 │  Motor de decisión + ML       │                           │                   │
 │  (calibración, winrate-opt)   │◄──────────────────────────┴───────────────────┘
 └──────────────┬───────────────┘         feedback loop: cada decisión medida,
                │                          auditada y comparada vs forward-data
                ▼
 ┌──────────────────────────────┐
 │  Backtester multi-fase        │
 │  filter-replay · Monte Carlo  │
 │  param-sweep · motor OHLCV     │
 └──────────────────────────────┘
```

<!-- screenshot pendiente: dashboard en tiempo real — pega aquí una captura del panel de estado -->

---

## Highlights de ingeniería

- **Forward-guard con auto-retiro** — las estrategias que no superan su deadline + profit-factor
  se jubilan solas (peso → 0) sin intervención manual. Diseño de riesgo que se aplica a sí mismo.
- **Simulador live-shadow** — mide el edge *ejecutable* simulando ejecución real (slippage contra
  el order book, fees, gas) sobre señales en vivo, en vez del edge teórico que engaña.
- **Análisis contrafactual** — cuando los gates de riesgo bloqueaban operaciones, simulé las
  señales bloqueadas contra precios de mercado reales y **demostré que ejecutarlas habría perdido
  −339% en 5 días** (186 de 204 trades perdedores). *Cuantifiqué que "no operar" era lo correcto.*
- **Debugging de sistemas en producción** — (1) un OOM-loop donde el kernel SIGKILL-eaba en bucle
  a un worker con un modelo de 946 MB → env-gate a un scorer ligero, RAM 87 %→63 %; (2) un proceso
  que el orquestador daba por *"healthy"* pero llevaba 7 días colgado tras una excepción no
  capturada, cazado por inspección de `/proc` (0 s CPU, sin heartbeat).
- **Disciplina de research** — maté candidatos con datos: market-making (spreads de 0.1¢, venue ya
  eficiente), mean-reversion (edge era cota superior, no ejecutable), CLV (muestra contaminada).
- **Bug de asignación de capital en producción** — dos estrategias auto-retiradas retenían en
  silencio el **62.7% del reparto de capital**: seguían en el universo del rebalanceador y una
  arrastraba un override manual que el persist honraba pese al bloqueo de la capa de riesgo.
  Detectado cruzando allocation vs riesgo — dos fuentes de verdad que nadie comparaba. El fix
  devolvió su parte a la superviviente (26%→69%). *Auditar el camino del dinero, no solo el del código.*
- **Monitor rediseñado como máquina de estados bidireccional** — un watcher alertaba "la fuente de
  alpha se apagó" y se auto-deshabilitaba (anti-spam). Correcto — hasta que la fuente **volvió** y
  la señal accionable pasó desapercibida 4 días. Rediseñado para vigilar ambas transiciones.
  *Un monitor de una condición recuperable no puede ser one-shot.*
- **Testear el path de fallo antes de cablear la alerta** — el primer disparo de un cron de
  reconciliación on-chain reveló que "aún sin configurar" (esperado) compartía path de alerta con
  "RPC roto" (fallo real) → 4 pings/día de ruido. Arreglado el mismo día. *Simular el estado actual
  del sistema contra el script antes de programarlo.*
- **Code review multiagente adversarial** — monté un harness de revisión con 23 agentes (4 lentes
  independientes + verificadores escépticos que deben *fracasar unánimemente en refutar* un hallazgo
  antes de reportarlo) sobre un lote de cambios ya verificado a mano: 16 hallazgos crudos → 9 reales
  confirmados (6 falsos positivos filtrados antes de llegar a un humano). Entre los confirmados: un
  patrón de deploy cuyo `git stash pop` incondicional podía regresionar datos de producción en
  silencio, y un panel de salud incapaz de reportar "muerto" porque `pgrep -f` se encontraba a su
  propio wrapper de shell. *La redundancia encuentra bugs; la verificación adversarial los mantiene
  honestos.*
- **Los health-probes deben medir éxito, no actividad** — el probe del builder del dashboard usaba
  el mtime de su log; como el cron volcaba stderr al mismo log, un builder crasheando en bucle
  refrescaba su propio heartbeat cada minuto y seguía "healthy" con la salida congelada. Probe movido
  al artefacto construido, que solo se actualiza en éxito. *Un componente que puede escribir su
  propio heartbeat mientras falla, lo hará.*
- **Los contrafactuales mienten si no son ejecutables** — el audit nocturno de "señales perdidas"
  puntuaba los rechazos con fills en la mecha intradía, coste cero y estrategias retiradas incluidas:
  una cota superior estructural que se leía como arrepentimiento. Reconstruí los defaults con fills
  ejecutables (cierre de vela), costes medidos y solo estrategias vivas — el número honesto *siguió*
  dando la razón a los gates (0% de aciertos en los rechazos recientes), pero ahora es un número
  accionable. *Un contrafactual optimista es una invitación permanente a aflojar el gate equivocado.*

<!-- screenshot pendiente: panel de research / registro de estrategias — opcional -->

---

## Stack

`Python` · `Docker (multi-stage, hardened)` · `supervisord` · `Linux/VPS` · `Tailscale` ·
`launchd/cron` · `CI` · `pandas/numpy` · `ML (calibración, backtesting)` · `SQLite/JSON ledgers` ·
`dashboard web (Chart.js, design tokens)` · `Telegram alerting`

---

## Qué demuestra (mapa a roles)

| Capacidad | Rol que la valora |
|---|---|
| Python de producción + arquitectura de sistemas | Backend / Platform Engineering |
| Microestructura de mercado, slippage, sizing, riesgo | Quant Dev / Trading Systems |
| ML aplicado + calibración + backtesting | Quant Research / ML Engineering |
| Docker / infra / observabilidad / CI | DevOps / SRE / Platform |
| **Rigor estadístico + honestidad intelectual** | **Todos** — es lo más escaso |

---

## Lo que aprendí

- Los edges retail se compiten (spreads de centésimas, fuentes de alpha que se agotan).
- **Actividad ≠ edge**: un sistema que opera "siempre" pierde; los buenos están quietos sin ventaja.
- Validar-antes-de-construir ahorra semanas; matar tu propia idea con datos es una habilidad.
- **Saber cuándo parar**: cuando el cuello de botella deja de ser técnico, seguir tocando destruye
  valor. Reconocerlo es criterio de ingeniería.

---

*El código fuente es privado (maneja credenciales y estrategias propias). Este writeup y las
capturas son la evidencia pública. Feliz de hacer un walkthrough técnico en una conversación.*
