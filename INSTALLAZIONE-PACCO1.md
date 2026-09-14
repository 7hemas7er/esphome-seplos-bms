# Installazione del test sul pacco 1 (`seplos-bms`)

File da flashare: **`seplos-bms-test-155.yaml`** (in questo branch).

Il nome del device resta `seplos-bms`, quindi **tutta la telemetria mantiene i
suoi entity_id**: Energy dashboard, template, utility meter e la card Sunsynk
continuano a funzionare senza toccare nulla. Cambia solo la parte allarmi.

## Passi

La config di test dichiara `name: seplos-bms`, lo stesso della config attuale.
**Non metterla come file separato**: la dashboard ESPHome mostrerebbe due
schede entrambe chiamate "seplos-bms" e non si capisce piu' quale si sta
installando. Va sostituito il contenuto del file esistente.

Da SSH sul .26:

```bash
# 1. backup della config attuale
sudo cp /homeassistant/esphome/esphome-web-e6554c.yaml \
        /homeassistant/esphome/esphome-web-e6554c.yaml.pre155

# 2. scarica la config di test al posto suo
sudo curl -fsSL -o /homeassistant/esphome/esphome-web-e6554c.yaml \
  https://raw.githubusercontent.com/7hemas7er/esphome-seplos-bms/refs/heads/test/alarm-frame-155/seplos-bms-test-155.yaml

# 3. verifica che il backup ci sia e che il nuovo file sia arrivato intero
ls -l /homeassistant/esphome/esphome-web-e6554c.yaml*
grep -c "alarm_event" /homeassistant/esphome/esphome-web-e6554c.yaml   # atteso: 9
```

Poi dalla dashboard ESPHome: **Install → Wirelessly (OTA)** sulla scheda
`seplos-bms`. La prima compilazione scarica i componenti da GitHub, quindi ci
mette qualche minuto in piu' del solito.

Il device riparte e si ripresenta con lo stesso nome.

Non serve toccare `secrets.yaml`: la config usa le stesse chiavi di prima
(`wifi_IOT`, `wifi_IOT_password`, `fallback_password`, `api_encryption_key`,
`ota_password`).

## Cosa cambia in Home Assistant

**Spariscono** (la #155 non ha gli aggregati):

- `binary_sensor.seplos_bms_seplos_bms_warning`
- `binary_sensor.seplos_bms_seplos_bms_protection`
- `binary_sensor.seplos_bms_seplos_bms_system_fault`
- `sensor.seplos_bms_seplos_bms_errors` — attenzione: la #155 ha una `errors`
  ma contiene altro. L'equivalente del vecchio contenuto è ora `alarms`.

**Compaiono**: `charging`, `discharging`, `balancing`, `voltage_protection`,
`temperature_protection`, `current_protection`, `soc_protection`, il text sensor
`alarms`, i bitmask grezzi `alarm event1..8` e `balancing bitmask`.

### Effetto sull'automazione allarmi

`automation.allarme_batteria_seplos_protezione_guasto` si appoggia a 4 trigger,
due dei quali sono del pacco 1 e spariranno. **Durante il test il pacco 1 non è
coperto dalle notifiche**; il pacco 2 continua normalmente e l'automazione resta
funzionante per lui. Non serve modificarla: HA ignora i trigger su entità
inesistenti.

Se il test dura più di un paio di giorni conviene aggiungere un trigger
temporaneo sui nuovi `*_protection` del pacco 1.

## Cosa guardare

La finestra utile è **il pomeriggio, a pacco pieno sotto carica PV**, quando il
BMS genera davvero allarmi. In quel momento il pacco 2 (firmware attuale)
riporta tipicamente:

```
Cell high voltage; Intermittent supply waiting
Cell overvoltage; Intermittent supply waiting
```

Sul pacco 1 con la #155 ci si aspetta, nelle stesse condizioni:

- `voltage_protection` → **on** (è alarm event 2, dove stanno cella alta e
  sovratensione)
- `alarms` → le stesse condizioni elencate
- `alarm event2 bitmask` → diverso da 0; il valore numerico è il byte grezzo e
  dice esattamente quali bit sono attivi

I due pacchi non sono identici (280 Ah e 300 Ah) e non arrivano a fine carica
nello stesso istante, quindi il confronto è sul **tipo** di allarme e sulla
coerenza, non sulla simultaneità.

Da segnalare come problema, se capita:

- una protezione che si accende con `alarm eventN bitmask` a 0 → decodifica
  incoerente;
- allarmi implausibili (cortocircuito, guasto sensore) a pacco tranquillo;
- `alarms` vuoto mentre una delle protezioni è on.

## Catturare il frame grezzo

Il logger è già a `DEBUG`. Nei log ESPHome del device compare:

```
Alarm frame (55 bytes) received
Alarm event 2 (voltage): 0x03
  Bit0 Monomer high voltage alarm: ON
  ...
```

Salva un blocco completo preso durante un allarme vero: è l'allegato che rende
utile il commento sulla PR.

## Tornare indietro

Reinstalla la config precedente:

```bash
sudo cp /homeassistant/esphome/esphome-web-e6554c.yaml.pre155 \
        /homeassistant/esphome/esphome-web-e6554c.yaml
```

poi Install → OTA. Gli entity_id della telemetria non si sono mai mossi; le
entità allarme tornano quelle di prima e l'automazione ritorna completa da sola.

Se in HA restano entità orfane marcate *restored*, si tolgono da
Impostazioni → Dispositivi e servizi → Entità, filtrando per non disponibili.
