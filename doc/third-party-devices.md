# Using Third-Party TCP Devices

PSU-EXT can communicate with SCPI instruments over TCP. This guide shows how
to add two third-party instruments and configure the existing dashboard
widgets for them:

- Siglent SPD4323X programmable power supply
- Rigol DM858E digital multimeter

The examples use channel 1 where a command requires a channel. Confirm all
limits and wiring against the instrument manual before controlling real
hardware.

## Before you begin

Start the Proxy and frontend and connect the frontend to the Proxy as described
in the [software quick-start guide](software-quick-start.md). The instrument
and the computer running the Proxy must be reachable over the same network.

Both instruments use TCP port `5025` for the SCPI socket examples in this
guide. If a connection fails, confirm the instrument's LAN configuration and
socket port on its front panel rather than assuming that its address is
correct.

## Add a TCP device

1. Open **Config** in the frontend and connect the **WebSocket** widget to the
   Proxy.
2. In **Device List**, select **Add Device**.
3. Set **Type** to `TCP`, enter a device name, the instrument's IP address, and
   port `5025`.
4. Add the device, then select **Connect** in its table row.
5. In **WebSocket Monitor**, select the connected device and send `*IDN?`.
   Check that the response identifies the expected manufacturer and model.

The examples below use these profile names and placeholder addresses:

| Instrument | Name | Type | IP address | Port |
| --- | --- | --- | --- | --- |
| Siglent SPD4323X | `SPD4323X` | `TCP` | `192.0.2.10` | `5025` |
| Rigol DM858E | `DM858E` | `TCP` | `192.0.2.20` | `5025` |

Replace the example addresses with the addresses shown by your instruments.
The `192.0.2.0/24` range is reserved for documentation and is not expected to
work on a real network.

## Add and configure a widget

1. Open **Dashboard** and enable edit mode.
2. Choose a widget from **Add widget** and add it to the dashboard.
3. Open the widget's settings.
4. Select the connected device and enter the SCPI fields shown below.
5. Set the card name, unit, color, polling frequency, and chart history options
   as desired, then select **Save**.

Start with a polling frequency of `1 Hz`. Faster polling increases traffic to
the instrument, and multiple widgets may compete for an instrument that can
process only one command at a time.

## Siglent SPD4323X power supply

The SPD4323X has four independently controlled channels. Replace `CH1` with
`CH2`, `CH3`, or `CH4` to configure widgets for another channel.

Observe the channel limits when setting voltage:

| Channel | Voltage range | Current range |
| --- | --- | --- |
| CH1 and CH4 | 0–6 V | 0–3.2 A |
| CH2 and CH3 | 0–32 V | 0–3.2 A |

### Single Value widgets

Add one **Single Value** widget for each value you want to display. Use device
`SPD4323X`, a frequency of `1 Hz`, and one of these query configurations:

| Card name | SCPI query | Unit | Meaning |
| --- | --- | --- | --- |
| CH1 Voltage | `MEAS:VOLT? CH1` | `V` | Measured output voltage |
| CH1 Current | `MEAS:CURR? CH1` | `A` | Measured output current |
| CH1 Power | `MEAS:POW? CH1` | `W` | Measured output power |
| CH1 Voltage Setpoint | `SOUR:VOLT? CH1` | `V` | Configured voltage |
| CH1 Current Setpoint | `SOUR:CURR? CH1` | `A` | Configured current limit |
| CH1 Mode | `MEAS:RUN:MODE? CH1` | leave empty | Regulation mode, such as `CV` or `CC` |

Single Value displays the response as text, so it is suitable for both numeric
values and the textual `CV`/`CC` response.

### Voltage, current, or power Chart

Add a **Chart** widget and configure its first series as follows:

| Setting | Voltage chart | Current chart | Power chart |
| --- | --- | --- | --- |
| Device | `SPD4323X` | `SPD4323X` | `SPD4323X` |
| SCPI query | `MEAS:VOLT? CH1` | `MEAS:CURR? CH1` | `MEAS:POW? CH1` |
| Frequency | `1 Hz` | `1 Hz` | `1 Hz` |
| Unit | `V` | `A` | `W` |
| Line label | `CH1 Voltage` | `CH1 Current` | `CH1 Power` |

Choose the chart size and history cap to suit the observation period. Chart
series require responses that can be parsed as numbers; do not chart the
`MEAS:RUN:MODE?` response.

### Output Single Toggle

Before enabling an output remotely, disconnect or protect the load, set a safe
voltage and current limit, verify the wiring, and confirm the selected channel.

Add a **Single Toggle** widget with these settings:

| Setting | Value |
| --- | --- |
| Device | `SPD4323X` |
| Card name | `CH1 Output` |
| Status command | `OUTP? CH1` |
| On command | `OUTP CH1,ON` |
| Off command | `OUTP CH1,OFF` |
| On response | `1` |
| Off response | `0` |
| Auto-refresh | enabled, `1 Hz` |

Response mappings are exact comparisons after surrounding whitespace is
removed. If the toggle reports an unknown state, send `OUTP? CH1` in the
WebSocket Monitor and use the instrument's actual responses as the mappings.

### OVP Protection Control

Add a **Protection Control** widget, select `SPD4323X`, and configure:

| Setting | Value |
| --- | --- |
| Card name | `OVP CH1` |
| Protection key | `OVP` |
| Unit | `V` |
| Value query | `OVP? CH1` |
| Value set template | `OVP CH1,{value}` |
| Trip query | `OVP:PROT:STAT? CH1` |

Leave the optional enable query and enable/disable command fields empty. The
SPD4323X SCPI interface exposes the OVP threshold and trip state through these
commands but does not provide the equivalent OVP enable-switch command used by
the widget's optional controls.

### OCP Protection Control

Add another **Protection Control** widget with:

| Setting | Value |
| --- | --- |
| Device | `SPD4323X` |
| Card name | `OCP CH1` |
| Protection key | `OCP` |
| Unit | `A` |
| Value query | `OCP? CH1` |
| Value set template | `OCP CH1,{value}` |
| Trip query | `OCP:PROT:STAT? CH1` |
| Enable query | `OCP:STAT? CH1` |
| Enable on command | `OCP:STAT CH1,1` |
| Enable off command | `OCP:STAT CH1,0` |
| Enabled response | `1` |
| Disabled response | `0` |

Set the widget's minimum, maximum, decimal precision, and refresh frequency to
values appropriate for the selected channel. Do not use a range merely because
the UI accepts it; the instrument remains the authority on valid limits.

### Sending setpoints

Single Value and Chart widgets read values but do not provide a generic numeric
setpoint control. Send arbitrary write commands through **WebSocket Monitor**
or a Script Runner script. For example, after checking the CH1 limits:

```text
SOUR:VOLT CH1,3.3
SOUR:CURR CH1,0.5
```

Setting a voltage or current does not by itself make it safe to enable an
output. Confirm both settings and the load before using the output toggle.

## Rigol DM858E multimeter

The DM858E is a measuring instrument, not a power source. Single Value and
Chart widgets are appropriate; output, OVP, and OCP controls do not apply.

### DC voltage Single Value

| Setting | Value |
| --- | --- |
| Device | `DM858E` |
| Card name | `DC Voltage` |
| SCPI query | `MEAS:VOLT:DC?` |
| Frequency | `1 Hz` |
| Unit | `V` |

### DC voltage Chart

| Setting | Value |
| --- | --- |
| Device | `DM858E` |
| Card name | `DC Voltage History` |
| SCPI query | `MEAS:VOLT:DC?` |
| Frequency | `1 Hz` |
| Unit | `V` |
| Line label | `DC Voltage` |

### DC current Single Value or Chart

Use the same settings, replacing the query with `MEAS:CURR:DC?` and the unit
with `A`. Use a card or line name such as `DC Current`.

### Select a range and read in one query

The `MEASure` query can select the function and range, perform a measurement,
and return the numeric reading in one command. These forms work as the SCPI
query for either a Single Value or Chart widget:

| Measurement | SCPI query | Effect |
| --- | --- | --- |
| DC voltage, automatic range | `MEAS:VOLT:DC? AUTO` | Selects DC voltage and chooses the range automatically |
| DC voltage, 10 V range | `MEAS:VOLT:DC? 10` | Selects DC voltage and uses the 10 V range |
| DC voltage, 10 V range and 1 mV resolution | `MEAS:VOLT:DC? 10,0.001` | Sets both range and resolution before reading |
| DC current, 1 A range | `MEAS:CURR:DC? 1` | Selects DC current and uses the 1 A range |
| DC current, 3 A range | `MEAS:CURR:DC? 3` | Uses the DM858E-specific 3 A range |

The range is the meter's full-scale measurement range, not the expected input
value. The DM858E supports a 3 A current range; the 10 A range belongs to the
DM858 model. Always use the correct current input terminal and protection
before selecting a current function.

The `MEASure` command selects and configures a measurement function before
taking a reading. Do not continuously poll voltage and current widgets against
the same DM858E at the same time: the commands would repeatedly switch the
meter function. Stop the old widget before changing functions, move the leads
to the correct input terminals, and confirm the expected range before making a
current measurement. A widget also reapplies an explicit range and resolution
on every poll.

The DM858E can return values in scientific notation. Single Value displays
that response, while Chart converts a valid numeric response into a plotted
sample.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| TCP connection times out | Confirm the instrument IP address, subnet, LAN cable, socket service, and port `5025`. Test reachability from the Proxy computer. |
| `*IDN?` has no response | Confirm that the device profile is connected and selected in WebSocket Monitor. Check whether another application already owns the instrument connection. |
| Widget remains unconfigured or disconnected | Select the exact connected device profile in widget settings and save the configuration. |
| Toggle shows an unknown state | Send its status query in WebSocket Monitor and update the on/off mappings to exactly match the trimmed responses. |
| Chart shows no samples | Run the query in WebSocket Monitor. A chart needs a numeric response; status strings such as `CV` cannot be plotted. |
| Values change function unexpectedly on the DM858E | Do not run measurement widgets for different functions simultaneously. Keep one active measurement function and verify the input terminals. |
| Siglent protection control reports an error | Check the channel name, threshold range, command fields, and returned trip or enable state against the instrument firmware and manual. |

## References

- [Siglent SPD4000X product page and manual downloads](https://www.siglent.com/int/products-overview/spd4000x/)
- [Rigol DM858 Series Programming Guide](https://download.rigol.com/en/Manual/Digital%20Multimeters/DM858/DM858_ProgrammingGuide_EN.pdf)

Instrument firmware can affect supported commands and response formatting.
Consult the manual matching the installed firmware when its behavior differs
from these examples.
