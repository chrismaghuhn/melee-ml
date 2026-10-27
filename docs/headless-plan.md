# Headless / ML-Umbau – Analyse

Basis: melee-pc @ `0fb0af3`. Ziel: `env.step(pads) -> obs` mit weit über 60 FPS,
schneller Reset, mehrere Instanzen parallel.

## Was schon da ist

| Baustein | Wo | Nutzen fürs ML |
|---|---|---|
| Null-Renderer | `MELEE_BACKEND=null` (`src/pc/main.c:352`, Dawn `BACKEND_NULL`) | Kein GPU-Rendering nötig |
| Sim-Tick als Funktion | `gm_RunSimTick()` in `src/melee/gm/gmscene.c:430` | Das ist `step()` |
| Ganzer Spielzustand als Snapshot | `snapshot_take/restore` in `src/pc/net_snapshot.c:602/706` (Heaps + `.melee_data/.melee_bss` per Linker-Script `src/pc/melee_state.ld`, ~6 MB, ~0,6 ms) | Reset in < 1 ms, auch Branching/MCTS |
| Pad-Injektion vor dem Tick | `pad_head()` / `replay_feed()` in `src/pc/net.c:2519/3086` | Hier schreiben wir die Agent-Aktionen rein |
| Fighter-Zustand auslesen | `log_state_bits()` in `src/pc/net_snapshot.c:342` (pos, vel, kb, %, motion_id, stocks, facing) | Vorlage für die Observation |
| Determinismus-Tests | `MELEE_NET_RECORD/REPLAY`, `MELEE_NET_SYNCTEST` | Prüfen, dass Headless bit-identisch bleibt |
| Austauschbare Uhr | `aurora::time::internal::set_now_function` (`extern/aurora/lib/time_internal.hpp:20`) | Virtuelle Zeit statt Wanduhr |
| Auto-Exit | `MELEE_EXIT_AFTER_FRAMES` (`src/pc/vi.c:241`) | Smoke-Tests |

## Was blockiert (und der Fix)

1. **Wanduhr-Pacing.** `pc_frame_boundary()` (`src/pc/vi.c:83`) schläft bis 60 Hz,
   und der Pad-Alarm (`lb_0195.c:109`, `OSSetPeriodicAlarm`) feuert nach
   `OSGetTime()` = echte Zeit. Die Scene-Loop (`gmscene.c:397`) wartet auf diesen
   Alarm, bevor sie tickt.
   → Headless-Modus: `set_now_function` auf eine virtuelle Uhr, die pro Step um
   genau 1/60 s vorrückt; Pacing-Sleep in `vi.c` überspringen. Dann feuert der
   Alarm genau einmal pro Step, deterministisch.
2. **Render-Pfad pro Frame.** `gmscene.c:455–470` (`HSD_StartRender` … `HSD_VICopyXFBAsync`)
   plus `aurora_begin/end_frame` in `vi.c`. Mit Null-Backend läuft das, kostet aber
   CPU (GX-FIFO-Aufzeichnung, Szenengraph-Walk).
   → Flag, das den Block überspringt (optional jeden N-ten Frame rendern zum Zuschauen).
   Achtung: `ftdrawcommon.c:379` hat Logik, die davon abhängt, ob gezeichnet wurde –
   Rollback-Resims laufen schon ohne Render, also ist der Pfad erprobt.
3. **Audio-Thread & 1000-Hz-Input-Thread.** Unnötig im Training.
   → SDL-Audio auf Dummy-Treiber (`SDL_AUDIO_DRIVER=dummy`) bzw. Synth stummschalten
   wie bei Resims (`pc_net_resim()` in `lbaudio_ax.c:769`); Input-Poll-Thread nicht starten.
4. **Menüs.** Der Weg bis in ein Match geht durch Titel/CSS/SSS.
   → Direkt-Boot in ein VS-Match (Charaktere/Stage per Env/Config), danach ein
   Snapshot am Match-Start = Reset-Punkt.
5. **Ein Prozess pro Env.** Alles ist global (MEM1 fest bei `0x80000000`).
   → Mehrere Prozesse, Steuerung über Shared Memory + Semaphore (oder Unix-Socket).

## Vorgeschlagene Architektur

```
Python (gymnasium VectorEnv)
   │ shm: 4×PADStatus rein, Observation-Struct raus, step/reset-Kommando
   ▼
melee-pc --headless   (N Prozesse)
   src/pc/ml_env.c:   virtuelle Uhr, Kommando-Loop, pad_head()-Injektion,
                      Observation aus Player_GetEntity()/GET_FIGHTER(),
                      Reset = snapshot_restore(match_start)
```

Hook-Punkt für die Kommando-Loop: direkt vor `gm_RunSimTick()` in `gmscene.c`
(dort, wo heute `pc_net_sync()` sitzt).

## Meilensteine

1. **Build läuft** mit eigenem ISO (NTSC-U 1.02), `MELEE_BACKEND=null`.
2. **Virtuelle Uhr + kein Pacing**: FPS messen (`MELEE_FPS=1`), Replay-Determinismus prüfen.
3. **Render überspringen**: FPS erneut messen.
4. **Direkt-Boot ins Match** + Snapshot-Reset.
5. **shm-Bridge + Python-Env**, Observation-Struct.
6. **Parallelisierung** (N Prozesse), Throughput-Benchmark.

## Offene Risiken

- DVD/ARQ-Transfers während des Ticks: `snapshot_take` verweigert dann
  (`net_snapshot.c:613`). Für Reset am Match-Start unkritisch, für Mid-Match-Snapshots beachten.
- Die Musik-Streams halten DVD beschäftigt – im Headless-Modus Musik abschalten.
- Ob Rendering-Skip über viele Frames Nebenwirkungen hat (z. B. Animation-Updates,
  die am Draw hängen), muss per Replay-Checksumme verifiziert werden.
