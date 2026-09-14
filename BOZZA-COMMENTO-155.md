Bozza del commento per https://github.com/syssi/esphome-seplos-bms/pull/155
Da incollare come commento sulla PR. Questo file non va mai a upstream.

---

Ran this branch for a day on a 16S Seplos V2 pack, with a second pack next to it
still on my old decoder so I had something to compare against. The two agreed on
every frame all day, so the decoding itself looks right.

One thing though. Yesterday afternoon the pack sat at 98-99% SOC and
`soc_protection` stayed on from 15:40 to 19:41. The only bit set anywhere in the
frame was alarm_event6 bit 1:

```
alarm_event6_bitmask       2
every other event bitmask  0
alarms                     "CHG: Intermittent recharge waiting"
```

It cleared the moment that bit cleared, so the wiring is fine, it's just that
`(alarm_event6 != 0)` counts a recharge wait as a protection. The other three
have the same shape, since each event byte interleaves alarm bits and protection
bits. Going by your own bit labels I think the masks want to be:

```
voltage_protection      alarm_event2 & 0xAA
temperature_protection  alarm_event3 & 0xAA
current_protection      alarm_event5 & 0xFA
soc_protection          alarm_event6 & 0x39
```

I left bit 6 of event6 out (Output connection fault), though that one's arguable.

Otherwise any pack that does intermittent recharge will show soc_protection on
for most of a sunny afternoon.

Couple of other things I ran into. test_alarm_frame_decoder.cpp doesn't actually
run in CI: run-cpp-tests.sh and ci.yaml both only sync tests/components/*/, so it
sits outside and the green "Run C++ unit tests" doesn't cover it. It also keeps
its own copy of the alarm name tables instead of going through the component, so
it would stay green if the two drifted apart. And `errors` isn't exposed in any
of the example yamls.

If you want anything else tried against real hardware, say so. Both packs are in
daily use and throw alarms on their own most afternoons.
