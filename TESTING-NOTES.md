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

Valutazione onesta: **quella di syssi è migliore**. È esattamente la risposta
che chmutoff chiedeva (manda il secondo comando solo dopo la prima risposta) e
ottiene entrambi i frame a ogni `update_interval`, mentre la nostra li ottiene
a cicli alterni, dimezzando la freschezza. L'unico punto a nostro favore è
l'instradamento esplicito invece dell'euristica sulla dimensione — ma è un
vantaggio teorico finché non si trova un frame che la inganna.

## Cosa possiamo dare che manca

La PR è ferma da 14 mesi e tecnicamente è pronta. Quello che con ogni
probabilità manca è **conferma su hardware reale** — e noi abbiamo due pacchi
Seplos, di cui uno che in questo momento riporta warning e protection.

Da verificare flashando questo branch:

1. Le 7 binary sensor riflettono lo stato vero del pacco? In particolare:
   nessun **falso positivo** a pacco sano. syssi è già stato morso da questo
   ([#232](https://github.com/syssi/esphome-seplos-bms/pull/232) "Fix EIC
   false-positive problem detection") e per un maintainer è la ragione
   principale per non fare merge di una decodifica allarmi.
2. Il pacco che oggi dà `warning: on` + `protection: on` col nostro fork: con
   #155 quale delle 7 si accende? Se nessuna, c'è un disaccordo da segnalare.
3. `alarms` ed `errors` riportano testo sensato o stringhe vuote?
4. L'euristica sulla dimensione instrada correttamente entrambi i frame su un
   pacco a 16 celle? (I nostri sono 16S: telemetria 150 byte, allarme ~55.)

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
