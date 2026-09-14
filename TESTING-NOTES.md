# Note di studio — PR #155 "Decode alarm frame"

File di lavoro sul fork, **non destinato a upstream**. Cancellare prima di
proporre qualsiasi cosa a syssi.

## Il punto

La funzionalità che avevamo forkato a maggio 2026 (decodifica del frame 0x44)
esiste già come PR del maintainer:

- **[#6](https://github.com/syssi/esphome-seplos-bms/issues/6)** — richiesta originale, aperta dal 19/06/2022
- **[#10](https://github.com/syssi/esphome-seplos-bms/pull/10)** — primo tentativo di syssi, 2022, +22/-26, ora in conflitto
- **[#155](https://github.com/syssi/esphome-seplos-bms/pull/155)** — l'implementazione vera, di syssi, aperta il 25/07/2025, ultimo commit 23/05/2026

**#155 è più completa della nostra in ogni dimensione.** Aprire una PR
concorrente non aggiungerebbe niente.

Stato al 14/09/2026: 16/16 check CI verdi, `MERGEABLE` / `CLEAN`,
**zero commenti, zero review in 14 mesi**.

## Confronto delle due implementazioni

|  | PR #155 (syssi) | Il nostro fork |
|---|---|---|
| Entità allarme | 7 binary sensor granulari: `charging`, `discharging`, `voltage_protection`, `temperature_protection`, `current_protection`, `soc_protection`, `balancing` | 3 aggregati: `warning`, `protection`, `system_fault` |
| Text sensor | `errors` + `alarms` | `errors` |
| Altri componenti | copre anche `seplos_bms_ble` | solo `seplos_bms` |
| Documentazione | aggiunge il protocollo V2.0 in `docs/` | nessuna |
| Esempi | aggiorna 9 YAML | nessuno |
| Test | `tests/test_alarm_frame_decoder.cpp` (381 righe) | nessuno |

### Come ciascuna risolve il problema delle due richieste

Questo era il nodo tecnico: il protocollo risponde a un comando per volta, e
[chmutoff nel 2023](https://github.com/syssi/esphome-seplos-bms/issues/6)
segnalava che mandare 0x42 e 0x44 nello stesso `update()` rompe la prima
risposta.

**#155 — concatenata.** Quando arriva la risposta di telemetria, manda subito
0x44 dall'interno della callback:

```cpp
if (data.size() >= 44 && data[8] >= 8 && data[8] <= 16) {
  this->on_telemetry_data_(data);
  if (this->parent_ != nullptr)
    this->send(0x44, this->pack_);
  return;
}
```

Instrada in ricezione con un'euristica sulla **dimensione** del frame
(`data.size() >= 9 && data.size() < 60 && data[8] >= 8 && data[8] <= 16`).

**Nostra — alternata.** Un comando per ciclo, ricordando cosa si è chiesto
(`last_requested_function_`), e instradando su quello invece che sulla
dimensione.

Valutazione onesta: **quella di syssi è migliore**, e non di poco.

Avevo scritto qui che il nostro instradamento esplicito era un punto a favore
rispetto all'euristica sulla dimensione. **È il contrario, ed è dimostrato.**

### Il nostro instradamento è la causa dei falsi positivi

Misurato sull'impianto il 14/09/2026: `protection` del pacco 2 si accende per
un solo frame **~83 volte al giorno** (830 transizioni in 10 giorni), su un
pacco che non è in protezione. Ogni impulso dura esattamente ~10s, cioè un
ciclo di alternanza.

Il meccanismo:

1. `update()` manda 0x44 e imposta `last_requested_function_ = 0x44`.
2. La risposta tarda o si perde.
3. `update()` successivo manda 0x42 e imposta il flag a 0x42.
4. Arriva **in ritardo la risposta 0x42** del ciclo precedente... o peggio, la
   0x44: in entrambi i casi il flag non corrisponde più al frame che arriva, e
   il frame finisce nel decoder sbagliato.

Quando un frame di telemetria finisce in `on_telesignalization_data_`, il
nostro codice non ha **nessun** controllo che lo respinga: non valida la
dimensione, e legge `temperature_sensors` senza validarlo. Verificato sul frame
di telemetria reale dei test upstream (16 celle, `data[8]=0x10`): passa il
controllo celle, passa tutto, e i byte degli "alarm event" finiscono per essere
i **valori di temperatura** (0x0B 0xA6 = 25,1 °C) letti come bitfield.

Risultato pubblicato su quel frame:

```
protection = ON, warning = ON, system fault = ON
errors = "Temp sensor fault; Current sensor fault; Cell high voltage;
          Cell overvoltage; Cell undervoltage; Charge over-temp; ...;
          Output short circuit; Short-circuit lockout; ..."
```

17 allarmi simultanei, cortocircuito in uscita compreso.

**#155 è immune per costruzione**: instrada con
`data.size() >= 9 && data.size() < 60`, e un frame di telemetria a 16 celle ne
ha 81 (150 sul nostro hardware). Non può finire nel decoder allarmi. In più
valida `temp_count > 8` e richiede `alarm_events_offset + 14` byte disponibili,
due guardie che a noi mancano entrambe.

Questo è anche il motivo per cui la sua scelta di concatenare invece di
alternare è più robusta, non solo più fresca: il secondo comando parte *dopo*
la risposta al primo, quindi non c'è una finestra in cui il flag e il frame in
arrivo possono divergere.

## Cosa possiamo dare che manca

La PR è ferma da 14 mesi e tecnicamente è pronta. Quello che con ogni
probabilità manca è **conferma su hardware reale** — e noi abbiamo due pacchi
Seplos, di cui uno che in questo momento riporta warning e protection.

Da verificare flashando questo branch:

1. **Le 7 binary sensor restano stabili?** È la domanda principale. Il nostro
   fork produce ~83 impulsi spuri al giorno sullo stesso pacco: se #155 sta
   piatto per 24-48h, è il contro-esempio documentato che serve. syssi è già
   stato morso dai falsi positivi
   ([#232](https://github.com/syssi/esphome-seplos-bms/pull/232) "Fix EIC
   false-positive problem detection") ed è la ragione più probabile per cui una
   PR con la CI verde è ferma da 14 mesi.
2. `alarms` ed `errors` riportano testo sensato o stringhe vuote?
3. L'euristica sulla dimensione instrada correttamente entrambi i frame su un
   pacco a 16 celle? (Telemetria 150 byte, allarme ~55: margine ampio rispetto
   alla soglia di 60, ma va confermato sul nostro firmware.)
4. Catturare un frame 0x44 grezzo con `logger: level: DEBUG` (`Alarm frame
   (N bytes)`) da allegare al commento sulla PR.

## Osservazioni sulla PR, verificate

Tre cose concrete, utili come commento a prescindere dal test:

1. **Il test non gira in CI.** `tests/test_alarm_frame_decoder.cpp` sta nella
   root di `tests/`, ma sia `run-cpp-tests.sh` sia `ci.yaml` sincronizzano solo
   `tests/components/*/`. Il check verde "Run C++ unit tests" **non include
   questo file**. Verificato: `grep -rn test_alarm_frame_decoder
   .github/workflows/ci.yaml run-cpp-tests.sh` non trova nulla.

2. **Il test verifica una copia della logica, non la logica.** È standalone
   (`assert` + `main()`), mentre gli altri 5 test del repo usano gtest e il
   componente vero. Le tabelle dei nomi allarme sono duplicate nel test con il
   commento "mirrors seplos_bms.cpp": se il componente cambia, il test resta
   verde. Il pattern giusto esiste già in
   `tests/components/seplos_bms/seplos_bms_test.cpp`.

3. **`errors` non è esposto in nessun esempio.** È definito in
   `text_sensor.py` ma non compare in nessuno dei 9 YAML aggiornati, mentre
   `alarms` e `balancing` sì.

## Config di prova

`seplos-test-alarm-frame.yaml` in questo branch: la nostra config viva del
pacco 2, con i componenti di #155 e tutte e 7 le binary sensor esposte, con
prefisso `test-` per non collidere con le entità esistenti in HA.

Flashare **su un solo pacco** e tenere l'altro sul firmware attuale come
riferimento, così si confrontano le due decodifiche sullo stesso impianto
in parallelo.

Per catturare il frame grezzo da allegare a un eventuale commento:
`logger: level: DEBUG` e cercare `Alarm frame (N bytes)`.
