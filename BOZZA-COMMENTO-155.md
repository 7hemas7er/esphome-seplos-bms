Bozza del commento per https://github.com/syssi/esphome-seplos-bms/pull/155
Da incollare come commento sulla PR. Questo file non va mai a upstream.

---

Tested this PR on real hardware — two 16S Seplos V2 packs on RS485, one running
this branch, the other left on a different decoder so I could compare.

**The frame decoding checks out.** Both read the same bits from the same frames
all day, no disagreement.

**One issue: `soc_protection` fires on a benign state.**

Yesterday afternoon, pack sitting at 98–99.6% SOC:

| | |
|---|---|
| `alarm_event6_bitmask` | `2` (bit 1 only) |
| all other event bitmasks | `0` |
| `alarms` | `CHG: Intermittent recharge waiting` |
| `soc_protection` | **on for 4h00m**, 15:40 → 19:41 |

When bit 1 cleared, the flag cleared in the same update — so the plumbing is
fine, it's the classification.

`*_protection = (alarm_eventN != 0)` treats every bit in the byte as a
protection, but each byte mixes alarm bits with protection bits. From your own
bit labels:

```
voltage_protection      alarm_event2 & 0xAA
temperature_protection  alarm_event3 & 0xAA
current_protection      alarm_event5 & 0xFA
soc_protection          alarm_event6 & 0x39
```

(event6 bit 6 "Output connection fault" is a judgement call — masked out above.)

As it stands, `soc_protection` sits on for hours every sunny afternoon on any
pack that does intermittent recharge.

Three smaller things:

- `tests/test_alarm_frame_decoder.cpp` isn't picked up by CI — `run-cpp-tests.sh`
  and `ci.yaml` only sync `tests/components/*/`, so the green "Run C++ unit
  tests" doesn't cover it.
- It also duplicates the alarm name tables ("mirrors seplos_bms.cpp") instead of
  exercising the component, unlike the gtest files under `tests/components/`.
- `errors` isn't exposed in any of the updated example YAMLs.

Happy to run anything else against the hardware — it's a live installation with
both packs, so alarm conditions show up on their own most afternoons.
