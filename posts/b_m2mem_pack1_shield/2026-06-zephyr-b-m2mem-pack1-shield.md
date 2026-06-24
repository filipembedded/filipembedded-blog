# Adding B_M2MEM_PACK1 shield support to Zephyr

*June 2026 · Zephyr RTOS · [PR #109004](https://github.com/zephyrproject-rtos/zephyr/pull/109004)*

After landing upstream board and SoC support for the **NUCLEO-U3C5ZI-Q**, I
wanted a follow-up that pushed me a bit further. The board exposes an **M.2
Key-A** slot intended for external memory cards, and STMicroelectronics ships a
matching add-on: the **B_M2MEM_PACK1** serial-memory pack. It wasn't supported
in Zephyr yet, so I picked it up.

What looked like "just add a shield" turned into a tour through Zephyr's
devicetree modeling, a brand-new memory subsystem, and even a build break that
had nothing to do with my hardware. This post is the story of the interesting
parts.

## The hardware

The M.2 Key-A connector on the NUCLEO-U3C5ZI-Q is wired to the SoC's **XSPI**
interface. It sits on the underside of the board:

<p align="center">
  <img src="nucleo_u3c5zi_q_M.2.jpeg" alt="NUCLEO-U3C5ZI-Q top and bottom view; the M.2 Key-A connector is highlighted on the underside of the board." width="760">
  <br>
  <em>The M.2 Key-A connector (highlighted) on the underside of the NUCLEO-U3C5ZI-Q, routed to the SoC's XSPI bus.</em>
</p>

The B_M2MEM_PACK1 isn't a single board — it's a family of memory daughter-cards
(different Macronix OctaFlash parts, an ST page-EEPROM, etc.), each on its own
carrier:

<p align="center">
  <img src="b_m2mem_pack1.webp" alt="The five B_M2MEM_PACK1 M.2 serial-memory cards (MB1927C and MB1928C variants) fanned out." width="560">
  <br>
  <em>The B_M2MEM_PACK1 pack: a family of M.2 serial-memory cards (MB1927C / MB1928C) — each variant became its own shield overlay.</em>
</p>

So the real goal was to support **all variants** as distinct Zephyr *shield*
overlays sharing one connector definition.

In Zephyr terms, that breaks down into three pieces:

1. A **connector** definition on the board side (which SoC peripherals the M.2
   pins map to).
2. One **overlay per memory variant** describing the flash device on the card.
3. A **`shield.yml`** declaring the shield and what it provides.

## The shield connector
First thing first, I needed to add a support for this new M.2 connector. 
So how to do it? By working on a ZephyrRTOS contributions for a while, I noticed
that how well you are observing patterns, is one of the most valuable skills you
can have. I looked around, and I saw that almost every board has some shield 
exposed, i.e **arduino_r3_connector** - one of the most famous shields, and I
tried to mimic the pattern I saw there, and I made a **m2_memory_connector.dtsi** -
board overlay file that defines the m2mem-connector devicetree node and connects 
the pins on the exact board to the M.2 memory connector pins - which should be 
defined globaly in some file - *zephyr/dt-bindings/gpio/m2-memory-connector.h*.
This file defines constants for non-power signals exposed on the M.2 serial memory
connector, values match the physical M.2 Key A connector pin numbers from the 
technical note [TN1618](https://www.st.com/resource/en/technical_note/tn1618-m2serialmemory-interface-specification-stmicroelectronics.pdf)
```dts
/* zephyr/include/zephyr/dt-bindings/gpio/m2-memory-connector.h */
...
#define M2MEM_CONNECTOR_GPIO_SPI_SCK  16 /**< Optional auxiliary SPI clock */
#define M2MEM_CONNECTOR_GPIO_SPI_MOSI 17 /**< Optional auxiliary SPI controller-out */
#define M2MEM_CONNECTOR_GPIO_LDO_EN   18 /**< Mandatory open-drain LDO enable */
#define M2MEM_CONNECTOR_GPIO_SPI_MISO 19 /**< Optional auxiliary SPI controller-in */
#define M2MEM_CONNECTOR_XSPI_NCS2     20 /**< XSPI chip select 2 */
#define M2MEM_CONNECTOR_GPIO_SPI_NSS  21 /**< Optional auxiliary SPI chip select */
#define M2MEM_CONNECTOR_XSPI_NCS1     22 /**< XSPI chip select 1 */

#define M2MEM_CONNECTOR_XSPI_DQS0 25 /**< XSPI data strobe 0 */
#define M2MEM_CONNECTOR_XSPI_D6   26 /**< XSPI data 6 */
#define M2MEM_CONNECTOR_XSPI_D7   27 /**< XSPI data 7 */
#define M2MEM_CONNECTOR_XSPI_D5   28 /**< XSPI data 5 */
#define M2MEM_CONNECTOR_XSPI_D4   31 /**< XSPI data 4 */
#define M2MEM_CONNECTOR_XSPI_D3   32 /**< XSPI data 3 */
#define M2MEM_CONNECTOR_XSPI_D2   34 /**< XSPI data 2 */
#define M2MEM_CONNECTOR_XSPI_D1   35 /**< XSPI data 1 */
#define M2MEM_CONNECTOR_XSPI_D0   37 /**< XSPI data 0 */

#define M2MEM_CONNECTOR_LED2       38 /**< Red LED (LD2) */
#define M2MEM_CONNECTOR_INT1       40 /**< Optional GPIO, error, or interrupt 1 */
#define M2MEM_CONNECTOR_XSPI_CLK1  41 /**< XSPI clock 1 */
#define M2MEM_CONNECTOR_LED1       42 /**< Green LED (LD1) */
#define M2MEM_CONNECTOR_XSPI_NCLK1 43 /**< XSPI differential clock complement 1 */
...
```


The board side defines stable labels for the peripherals routed to the
connector, so a shield overlay never has to know the SoC's instance names:

```dts
/* boards/st/nucleo_u3c5zi_q/m2_memory_connector.dtsi */
m2mem_i2c: &i2c1 {};
m2mem_xspi: &xspi1 {};
```

A shield variant then only references those labels:

```dts
/* boards/shields/st_b_m2mem_pack1/b_m2mem_pack1_mb1927_18ba.overlay */
&m2mem_xspi {
    status = "okay";

    mx25uw25645: xspi-nor-flash@0 {
        compatible = "st,stm32-xspi-nor";
        /* ... */
    };
};
```

One early review note from a maintainer was simply: *document these connector
labels in the shield-interfaces page.* Small, but it's exactly the kind of thing
that makes a contribution reusable instead of a one-off — so the connector got
proper documentation alongside the code.

## Challenge 1 — Jumping straight to the MSPI API

My first implementation used the STM32-specific `st,stm32-xspi-nor` driver,
which is the established way to drive these flashes today. During review,
@erwango suggested something better:

> Even if the current flash/ext_nor driver is not deprecated yet, it probably
> makes sense to jump directly to MSPI which will make these cards more widely
> compatible upfront.

**MSPI** is Zephyr's newer, vendor-agnostic multi-bit SPI subsystem. Instead of
a vendor NOR driver bound to a vendor controller, you model:

- the SoC peripheral as a generic **MSPI controller** (`st,stm32-xspi-controller`), and
- each flash as a generic **`jedec,mspi-nor`** device.

I hadn't worked with MSPI before, so I read through the subsystem and the
`samples/drivers/mspi/mspi_flash` reference overlays, then reworked the whole
shield onto it. The flash node went from a vendor binding:

```dts
mx25uw25645: xspi-nor-flash@0 {
    compatible = "st,stm32-xspi-nor";
    /* ST-specific properties */
};
```

to the generic MSPI one:

```dts
mx25uw25645: mspi-nor-flash@0 {
    compatible = "jedec,mspi-nor";
    /* portable, controller-agnostic description */
};
```

The result is cleaner and, more importantly, makes these cards usable on any SoC
with an MSPI controller — not just this STM32.

## Challenge 2 — Killing a hardcoded memory address

In the original overlay I had the memory-mapped base address of the external NOR
written directly into the shield:

```dts
ranges = <0x0 0x90000000 DT_SIZE_M(64)>;
```

@FRASTM flagged it:

> The base address of the external NOR depends on the stm32 series; this is
> better defined in the board dtsi.

He was right — `0x90000000` is an SoC/board property, not a shield one. Every
board exposing this connector would otherwise have to restate the same mapping.

The fix fell out of the MSPI migration almost for free. The STM32 MSPI driver
reads the memory-map window straight from the controller's `reg[1]` at SoC
level, so once the shield used `jedec,mspi-nor`, the hardcoded address in the
overlay simply disappeared — a single source of truth, defined where the
hardware actually defines it. One review comment ended up resolving two separate
problems.

## Challenge 3 — A build break that wasn't mine

When @FRASTM asked whether I'd checked the `jesd216` and `spi_flash` samples
against the shield, I built all five variants. `jesd216` was fine. `spi_flash`
failed to compile:

```
error: 'struct flash_stm32_xspi_data' has no member named 'jedec_id'
```

This wasn't caused by my shield — it was a latent mismatch in
`flash_stm32_xspi.c`. The `XSPI_NOR_STM32_INIT` macro initializes `.jedec_id`
whenever the devicetree node has a `jedec-id` property:

```c
.jedec_id = DT_INST_PROP(n, jedec_id),
```

…but the field itself only exists when `CONFIG_FLASH_JESD216_API` is enabled:

```c
#ifdef CONFIG_FLASH_JESD216_API
    uint8_t jedec_id[...];
#endif
```

`spi_flash` doesn't enable that option, so the initializer referenced a field
that wasn't compiled in. The fix is to guard the initializer with the same
condition as the field:

```c
#ifdef CONFIG_FLASH_JESD216_API
    .jedec_id = DT_INST_PROP(n, jedec_id),
#endif
```

After that, both samples built across all five variants. It's a small fix, but a
satisfying one — it helps anyone who hits that configuration, not just this
shield.

## Challenge 4 — Catching my own earlier mistake

Reviewing the board's I2C wiring, @erwango asked whether the connector's I2C
mapping was compatible with the Arduino header mapping. Checking the board user
manual (UM3599, Table 14), I realized I had mapped I2C1 to the **wrong pins** in
my *original* board-support PR:

```dts
/* before — incorrect */
&i2c1 {
    pinctrl-0 = <&i2c1_scl_pb8 &i2c1_sda_pb9>;

/* after — correct Arduino I2C pins */
&i2c1 {
    pinctrl-0 = <&i2c1_scl_pb6 &i2c1_sda_pb7>;
```

A good reminder that review on a new contribution often surfaces bugs in the
*previous* one too. Better caught now than later.

## Making the coverage visible in CI

Rather than relying on people remembering to run manual `SHIELD=...` builds, I
added explicit **Twister** scenarios for every shield variant in the sample's
`sample.yaml`, and moved the shared console-harness config to a common section
so the original and new scenarios validate the same way. The shield's coverage
is now continuously checked upstream instead of being a manual step.

## Threads left open (on purpose)

Two things I deliberately deferred to follow-ups rather than rush:

- **The right datasheet.** I'd initially grabbed the wrong Macronix part
  (MX25U / MX25UM, the quad-only / older OctaFlash) instead of the **MX25UW**
  actually fitted on MB1927-18BA. The correct datasheet is gated on Macronix's
  site, so a couple of values (read dummy cycles, JEDEC ID) carry `TODO` markers
  with provisional values until I get the real document.

- **The M95P32.** This variant is a **page EEPROM**, not a standard NOR. It has
  512-byte pages and a non-standard page-program opcode (0x02 is an auto-erase
  "page write"). It *does* expose a JEDEC ID, SFDP, standard erase commands and a
  1-1-4 quad read, so `jedec,mspi-nor` might be able to drive it — but whether
  modeling an EEPROM through the NOR binding is *right* (vs. a dedicated
  binding/driver) is a design question worth asking maintainers before adding it.

Sometimes the most senior-looking move in a PR is knowing what *not* to cram
into it.

## What I took away

- **Listen for the "you could go one level deeper" hint in review.** The MSPI
  suggestion was optional and non-blocking — taking it turned a vendor-specific
  shield into a portable one and dissolved an unrelated problem along the way.
- **Build the samples, not just your own overlay.** That's how I found the
  upstream `jedec_id` break.
- **A new contribution is a chance to fix the previous one.** The I2C pin
  correction came for free.
- **Deferring is a feature.** Open questions documented clearly are better than
  guesses merged quietly.

None of this would have been possible without the patient guidance of the Zephyr
maintainers at STMicroelectronics — @kartben, @erwango, and @FRASTM. Thank you.

---

*PR: [zephyrproject-rtos/zephyr#109004](https://github.com/zephyrproject-rtos/zephyr/pull/109004)*
