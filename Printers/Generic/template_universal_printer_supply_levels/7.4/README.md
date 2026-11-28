# Universal Printer Supply Levels by SNMP
This template simply tracks your toner/ink levels via SNMP, and a simple ping to your device, with "low" and "critical" alerts. That's the whole scope of the template.

The ping item uses the key `icmpping[,3]`, so the template can be linked together with Zabbix's "ICMP Ping" or "Network Generic Device by SNMP" templates (both have a plain `icmpping` item). Its trigger "Printer is unreachable (asleep or switched off)" is informational: printers in deep sleep or switched off stop answering ping, which is usually expected.

As long as your MFP follows the Printer MIB standard (should be true for any mainstream printer), this template will discover all installed "Supplies" in your MFP via SNMP. "Marker Supplies" is the term used in the Printer MIB for toner, ink, and other consumable substances that make a mark on whatever you're printing on.

## What's provided
Item prototypes use the "current" and "max" values provided by SNMP to calculate the remaining amount in a percentage. Simple graph prototypes included.

Alert prototypes provide an info-level alert when the "low" threshold is crossed, and a warning-level alert when the "critical" threshold is crossed. These are 5% and 0% by default, and are adjustable with provided macros.

## Printer status (HOST-RESOURCES-MIB)
A second discovery rule finds the printer device(s) (hrPrinterTable, usually index 1) and adds:
- device status (hrDeviceStatus) and printer status (hrPrinterStatus: idle / printing / warmup);
- the detected error state (hrPrinterDetectedErrorState), decoded into one item per condition and an "Active error conditions" text item.

| Trigger | Severity |
|---|---|
| Paper jam, Printer offline, Service requested, Printer is down | Average |
| Out of paper, Door open, Input tray empty, Input/Output tray missing, Supply missing, Output tray full | Warning |
| Low paper, Output tray nearly full, Preventive maintenance overdue | Info |

Low/no toner are collected but have no trigger (the supply levels cover them). Interval: `{$PRINTER.STATUS.INTERVAL}` (1m).

## Covers (Printer-MIB prtCoverTable)
Discovers covers/interlocks by name and polls their status every minute. Trigger **Cover [name] is open** (Warning) when a cover or its interlock is open.

## Input trays (Printer-MIB prtInputTable)
Discovers paper trays with a capacity of 2 sheets or more (`{$PRINTER.TRAY.MAX.MATCHES}`; skips manual / single-sheet feeds; names to skip: `{$PRINTER.TRAY.NOT_MATCHES}`). Per tray: paper level and status (5m). Triggers: **Tray [name] is empty** (level 0) and **Tray [name] reports a critical alert** (status bit 16), both Warning. Many printers report only "some paper" (-3) rather than a sheet count.

## Page count and print engine (Printer-MIB prtMarkerTable)
Per marker (print engine, usually one): **Lifetime count** (prtMarkerLifeCount, 15m), **Pages per day** (dependent, change per second x 86400) and **Status** (5m). Trigger **print engine reports a critical alert** (status bit 16, Average).

## Printer alert table (Printer-MIB prtAlertTable)
A discovery rule (`{$PRINTER.ALERTS.INTERVAL}`, 1m) turns each alert the printer currently reports into an item with the printer's own text: trigger **Printer alert: <text>** (critical, Average) or **Printer warning: <text>** (Info). When the printer drops the alert, discovery deletes item and trigger at once and the problem resolves. Discovery (GETBULK) is used instead of `walk[]` because some printers answer `walk[]`'s initial GET with noSuchName when the table is empty.

## HP front panel message
On HP printers (gdStatusDisplay answers) a discovery rule adds **HP front panel message** (5m): the status line shown on the printer, e.g. "Sleep mode is on.". Other vendors: nothing is created. No trigger.

## Setup
1. Download `template_universal_printer_supply_levels.yaml`.
2. In Zabbix, go to `Data Collection` -> `Templates` and then `Import` the .yaml with the button in the upper-right.
3. Create a new Host using an SNMP interface and attach the template "**Universal Printer Supply Levels by SNMP**".
4. Adjust Host `Macros` to filter Discovery.

## Additional tweaking
Depending on how your printer uses the Supplies table, **you very well may end up with unused or duplicate items discovered**. To filter out unwanted entries, it's easiest to discover everything, and use the discovered items as reference. Every discovered item will have its SNMP index as part of the item key (e.g. supply.type.[5] for the supply at SNMP Index 5) - Any indices you wish to filter out of discovery, I've provided macros to do so.

You can learn more about the Printer MIB by searching for it on your favorite search engine. It's old, hasn't changed, and is widely used and documented.

## Credits
Author: Mick Swanson / @Treeefort
