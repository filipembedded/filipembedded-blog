# Adding B_M2MEM_PACK1 shield support to Zephyr
- [PR #109004](https://github.com/zephyrproject-rtos/zephyr/pull/109004), merged **24.06.2026**.

## Intro
After I added upstream support for [STM32U3C5 SoC and NUCLEO-U3C5ZI-Q board](https://github.com/zephyrproject-rtos/zephyr/pull/105503), I observed the board and I got the idea to make one more contribution for it. I noticed that this board has a M.2 Key-A slot at the bottom side, and it sparked my curiosity.
After some research, I found out the intention behind this slot - the [B_M2MEM_PACK1](https://www.st.com/en/evaluation-tools/b-m2mem-pack1.html) shield.
<p align="center">
  <img src=".docs/nucleo_u3c5zi_q_M.2.jpeg" alt="NUCLEO-U3C5ZI-Q top and bottom view; the M.2 Key-A connector is highlighted on the underside of the board." width="760">
  <br>
  <em>The M.2 Key-A connector (highlighted) on the underside of the NUCLEO-U3C5ZI-Q, routed to the SoC's XSPI bus.</em>
</p>

## What is a B_M2MEM_PACK1 shield?
This shield is a pack of five different non-volatile serial memory cards by STMicroelectronics. These cards contain serial flash memories from different memory vendors and offer multiple serial bus schemes and memory interfaces.

<p align="center">
  <img src=".docs/b_m2mem_pack1.webp" alt="The five B_M2MEM_PACK1 M.2 serial-memory cards (MB1927C and MB1928C variants) fanned out." width="560">
  <br>
  <em>The B_M2MEM_PACK1 pack: a family of M.2 serial-memory cards (MB1927C / MB1928C) - each variant became its own shield overlay.</em>
</p>

## Zephyr shields - short recap
In Zephyr, a shield is defined by two key characteristics:
1. **Physical connectors** - the mechanical interface
2. **Electrical signals** - what each pin actually does

The connection between the host boards and a shield it carries is being done by Devicetree files on the board and shield sides:
1. **Board side** - The boards side must have dedicated devicetree file (e.g. **arduino_r3_connector.dtsi**) which uses [GPIO Nexus nodes](https://docs.zephyrproject.org/latest/hardware/porting/shields.html#gpio-nexus-node). It is like an interface between SoC GPIO pins and pins on the shield itself. 
Example: Every board that supports the arduino_r3_connector, must have a file `arduino_r3_connector.dtsi` which has the `arduino_header` GPIO Nexus node which maps pins exposed on the board's SoC to the common pin-names on the interface itself. It looks like this:
```dts
arduino_header: connector {
        compatible = "arduino-header-r3";
        #gpio-cells = <2>;
        gpio-map-mask = <0xffffffff 0xffffffc0>;
        gpio-map-pass-thru = <0 0x3f>;
        gpio-map = <0 0 &gpioa 0 0>,    /* A0 */
                   <1 0 &gpioa 1 0>,    /* A1 */
                   <2 0 &gpioa 4 0>,    /* A2 */
                   <3 0 &gpiob 0 0>,    /* A3 */
                   <4 0 &gpioc 1 0>,    /* A4 */
                   <5 0 &gpioc 0 0>,    /* A5 */
                   <6 0 &gpioa 3 0>,    /* D0 */
                   <7 0 &gpioa 2 0>,    /* D1 */
                   <8 0 &gpioa 10 0>,   /* D2 */
                   <9 0 &gpiob 3 0>,    /* D3 */
                   <10 0 &gpiob 5 0>,   /* D4 */
                   <11 0 &gpiob 4 0>,   /* D5 */
                   <12 0 &gpiob 10 0>,  /* D6 */
                   <13 0 &gpioa 8 0>,   /* D7 */
                   <14 0 &gpioa 9 0>,   /* D8 */
                   <15 0 &gpioc 7 0>,   /* D9 */
                   <16 0 &gpiob 6 0>,   /* D10 */
                   <17 0 &gpioa 7 0>,   /* D11 */
                   <18 0 &gpioa 6 0>,   /* D12 */
                   <19 0 &gpioa 5 0>,   /* D13 */
                   <20 0 &gpiob 9 0>,   /* D14 */
                   <21 0 &gpiob 8 0>;   /* D15 */
};
```
It also defines the labels for buses exposed on the connector:
```dts
arduino_i2c: &i2c1 {};

arduino_spi: &spi1 {};

arduino_serial: &lpuart1 {};
```


2. **Shield side** - Consists of shield devicetree files, more precise:
- **<shield_name>.overlay** - This file includes the connector gpio bindings, i.e:
```dts
#include <zephyr/dt-bindings/gpio/arduino-header-r3.h>
```
and it can optionally define some root devicetree node which is used to define a new node, and probably define something unique to that shield (i.e. some LED's,...) which cannot be referenced by some other, existing node.
```dts
/ {
	my_shield {
		compatible = "vendor,my-shield";
		status = "okay";
	};
};
```
More importantly, this file references existing nodes (defined in connector devicetree files) and adapts them to the peripherals exposed on the shield, i.e.:
```dts
&arduino_spi {
	status = "okay";
	cs-gpios = <&arduino_header ARDUINO_HEADER_R3_D9 GPIO_ACTIVE_LOW>;
};
```

- **Kconfig.shield** - This file defines the Kconfig symbol for the shield
```kconfig
config SHIELD_<SHIELD_NAME>
	def_bool $(shields_list_contains,shield_<shield_name>)
```
- **shield.yml** - This file defines what peripherals does the shield support
```yaml
shield:
  name: <shield_name>
  full_name: <full_shield_name>
  vendor: <shield_vendor>
  supported_features:
    - <supported_features>
```


## The PR workflow
This PR has been a tough one, and I did it through eight commits, where every commit chronologically represents my flow of thinking and adaptations I made by respecting reviewers' feedback. 

### 1. XSPI controller support 
Commit: [*dts: arm: st: stm32u3: add XSPI controller support*](https://github.com/zephyrproject-rtos/zephyr/pull/109004/changes/74fba4a29f02178d029c263f6429cdf48aadae86) 

Firstly, by reading the official B_M2MEM_PACK1 and NUCLEO_U3C5ZI_Q docs, I noticed that the M.2 connector is tied to **XSPI** peripheral, so I jumped to the docs and [AN5050](https://www.st.com/resource/en/application_note/an5050-getting-started-with-octospi-hexadecaspi-and-xspi-interface-on-stm32-mcus-stmicroelectronics.pdf) had all answers I needed. Long story short, the **XSPI** - e**X**tended **S**erial **P**eripheral **I**nterface, is the new interface that responds to the growing demand for faster, bigger external memories, needed by richer graphics, wider range of multimedia and data-intensive applications in embedded systems, which require higher data throughputs and extra demands on the limited on-chip MCU memory. It is a flexible interface which can be configured as:

 1. **SPI** for 1-line data transmission
 2. **Dual-SPI** for 2-line data transmission
 3. **Quad-SPI** for 4-line data transmission
 4. **Octo-SPI** for 8-line data transmission
 5. **Hexadeca-SPI** for 16-line data transmission

The STM32U3 implements it as OCTOSPI1, so on this SoC the XSPI goes up to octal mode (8 data lines).

I checked out the SoC devicetree files, and there wasn't XSPI node at all, I didn't include it in my SoC and board PR previously. I modified the `dts/arm/st/u3/stm32u3.dtsi` and added the node:
```dts
		xspi1: spi@420d1400 {
			compatible = "st,stm32-xspi";
			reg = <0x420d1400 0x400>, <0x90000000 DT_SIZE_M(256)>;
			interrupts = <76 0>;
			clock-names = "xspix", "xspi-ker";
			clocks = <&rcc STM32_CLOCK(AHB2_2, 4)>,
				 <&rcc STM32_SRC_SYSCLK OCTOSPI_SEL(0)>;
			#address-cells = <1>;
			#size-cells = <0>;
			status = "disabled";
		};
```
where:
- **xspi1** is the node label, and `spi@420d1400` is the node name, where `0x420d1400` is the **unit address** - the address in the SoC memory map where the XSPI1 peripheral registers begin
- **compatible** property describes which bindings will be used for this node - in short, which driver will be used for this node
- **reg** property has two address windows, first one is the **register block** - `<0x420d1400 0x400>` where first address matches unit address, and second one represents size of the register space; Second address window represents predefined address space reserved for **memory-mapped** mode, start address of the memory-mapped region is `0x90000000` and size of that region can be maximally 256MB (This is the maximal size of the external memory we can do memory-mapping with, limitation of address space) 
- **interrupts** property has two values, where first one describes which IRQ line is used for this node - IRQ line 76 is the _OCTOSPI1 global interrupt_ in the STM32U3 vector table [RM0487](https://www.st.com/resource/en/reference_manual/rm0487-stm32u3-series-armbased-32bit-mcus-stmicroelectronics.pdf); Second value represents the Zephyr interrupt priority.
- **clock-names/clocks** properties have two clocks, and this can be a little bit confusing. These two clocks actually have two different jobs: 

	- **xspix**	-> `STM32_CLOCK(AHB2_2, 4)` is the bus clock where bit 4 (`OCTOSPI1EN`) of `RCC_AHB2ENR2` (_2 suffix means second AHB2 enable register) enables the peripheral clock. This is the gate that makes the peripheral's AHB ports alive, and with this being closed, access to peripheral registers triggers bus fault.
	- **xspi-ker** -> `<&rcc STM32_SRC_SYSCLK OCTOSPI_SEL(0)>` is the kernel clock, the one that actually clocks the peripheral. The XSPI derives the memory interface clock (the CLK line towards the flash) from it, through its internal prescaler.
- **#address-cells / #size-cells** - describes how **child** nodes are addressed: 
```dts
	#address-cells = <1>; -> One address cell
	#size-cells = <0>; -> No size cell
``` 
means the child is written as a xspi-nor-flash@0 where 0 is the chip-select index, not a memory address. _! This will be a problem later on...!_
- **status = "disabled"** - The node initializes in a disabled state.

### 2. M.2 connector support
Commit: [*dts: bindings: gpio: add ST M2 memory connector binding*](https://github.com/zephyrproject-rtos/zephyr/pull/109004/changes/698ceeda5a40cd3b830d420fb0a5c07661269e47)

As mentioned earlier, the B-M2MEM-PACK1 attaches to Nucleo boards through an M.2 Key A connector. A shield is only portable if the connector it plugs into is described once, centrally, and then reused by every board that exposes it. Therefore, before implementing the shield itself, I first had to add support for the connector.

In Zephyr, connectors are described using two main components - **binding** and **header for pin macros**. This is what I implemented for M.2 memory connector:
1. **Binding** -  `dts/bindings/gpio/st,m2-memory-connector.yaml` - Defines the connector type and its devicetree representation as a GPIO nexus, and defines how GPIO specifiers `pin`, `flags` are interpreted.
2. **Header for pin macros** - `include/zephyr/dt-bindings/gpio/m2-memory-connector.h` - Defines connector pins, like:
```c
...
#define M2MEM_CONNECTOR_GPIO_SPI_SCK  16 /**< Optional auxiliary SPI clock */
#define M2MEM_CONNECTOR_GPIO_SPI_MOSI 17 /**< Optional auxiliary SPI controller-out */
#define M2MEM_CONNECTOR_GPIO_LDO_EN   18 /**< Mandatory open-drain LDO enable */
#define M2MEM_CONNECTOR_GPIO_SPI_MISO 19 /**< Optional auxiliary SPI controller-in */
...
```
,where some shield that uses this connector can include this file in shield overlay file and reference this pins to a physical pins exposed on the shield itself. (**b_m2mem_pack1** shield in this case).

### 3. Board side connector definition
Commit: [*boards: st: nucleo_u3c5zi_q: add M2 memory connector support*](https://github.com/zephyrproject-rtos/zephyr/pull/109004/changes/1db9c2a6be13467cccc808e19204cbe75853e1ec)

This commit introduced the `m2_memory_connector.dtsi` which is used to describe the M.2 connector on the **nucleo_u3c5zi_q** board - it defines the connections between GPIO pins on the board and pins on actual M.2 connector. It is adapted to the **nucleo_u3c5zi_q** pinout and respects the **m2-memory-connector binding**.

### 4. B_M2MEM_PACK1 shield
Commit: [*boards: shields: add ST B-M2MEM-PACK1 support*](https://github.com/zephyrproject-rtos/zephyr/pull/109004/changes/a6fcde70be1f27ab8f5d4b68b3d75ca0d5a2a66d)

Here the actual shield-related files are added: 
  - doc files (a picture and `index.rst` file describing shield features)
  - common `b_m2mem_pack1_common.dtsi` which defines the features populated on all shield variants (`m2mem-leds` node which describe the 2 LED's and `m2mem_id_eeprom` node which is child node to `m2mem_i2c` parent, and describes the I2C identification EEPROM memory which is used to identify the populated memories on the shield variants)
  - variant-specific overlay files - `b_m2mem_pack1_*.overlay` - One overlay per variant. Includes common .dtsi file and defines the child devicetree node under `m2mem_xspi` parent node.
    At the time of creating this commit, I only knew about legacy XSPI drivers provided in Zephyr. A snippet below describes the dereferenced `m2mem_xspi` node and child node describing the **Macronix MX25UW25645** Octal SPI memory, and specifying the legacy XSPI driver by setting `compatible = "st,stm32-xspi-nor"`

    *boards/shields/st_b_m2mem_pack1/b_m2mem_pack1_mb1927_18ba.overlay* 
    ```dts

      &m2mem_xspi {
    	  status = "okay";

      	mx25uw25645: xspi-nor-flash@0 {
      		compatible = "st,stm32-xspi-nor";
      		reg = <0>;
      		size = <DT_SIZE_M(256)>; /* 256 Mbits */
      		ospi-max-frequency = <DT_FREQ_M(50)>;
      		spi-bus-width = <XSPI_OCTO_MODE>;
      		data-rate = <XSPI_DTR_TRANSFER>;
      		reset-gpios = <&m2mem_ldo_en M2MEM_CONNECTOR_GPIO_LDO_EN GPIO_ACTIVE_LOW>;
      		reset-gpios-duration = <1>;
      		four-byte-opcodes;
      		status = "okay";

      		#address-cells = <1>;
      		#size-cells = <1>;
      		ranges = <0x0 0x90000000 DT_SIZE_M(32)>;

      		m2mem_flash0: flash@0 {
      			compatible = "soc-nv-flash";
      			reg = <0x0 DT_SIZE_M(32)>;
      			erase-block-size = <DT_SIZE_K(4)>;
      			write-block-size = <1>;
      		};
      	};
      };
    ```

### 5. Legacy STM32 XSPI driver JEDEC ID fix
Commit: [*drivers: flash: stm32_xspi: guard jedec_id init with JESD216_API*](https://github.com/zephyrproject-rtos/zephyr/pull/109004/changes/80550a04acc97ff40b889d69d3ad80941f596536)

While building the `spi_flash` sample against the shield, I hit this error: `error: 'struct flash_stm32_xspi_data' has no member named 'jedec_id'`

The legacy driver initialized `.jedec_id` whenever the devicetree node had a `jedec-id` property. The field itself only exists when `CONFIG_FLASH_JESD216_API` is enabled, and `spi_flash` doesn't enable it. The fix guards the initializer with the same Kconfig option:

*drivers/flash/flash_stm32_xspi.c*
```diff
 		.page_size = SPI_NOR_PAGE_SIZE, /* by default, to be updated by sfdp */
-		IF_ENABLED(DT_INST_NODE_HAS_PROP(inst, jedec_id), (
-			.jedec_id = DT_INST_PROP(inst, jedec_id),
-		))
+		IF_ENABLED(DT_INST_NODE_HAS_PROP(inst, jedec_id), (
+			IF_ENABLED(CONFIG_FLASH_JESD216_API,
+				(.jedec_id = DT_INST_PROP(inst, jedec_id),))
+		))
 
 		XSPI_DMA_CHANNEL(STM32_XSPI_NODE(inst), tx, TX, MEMORY, PERIPHERAL)
```

This fixed the build issues I have faced and at a first glance it looked like a finished PR. Even though the flash samples built for all shield variants, the maintainers noticed a major problem with this implementation. 

During review, [@FRASTM](https://github.com/FRASTM) and [@erwango](https://github.com/erwango) pointed at this line in the shield overlay:
```dts
ranges = <0x0 0x90000000 DT_SIZE_M(32)>;
```

`0x90000000` is the base address for the XSPI memory-mapped window, and it depends on the STM32 series. A shield is supposed to be portable, so it shouldn't contain a SoC-specific address. The hard-coded address ended here because of how the legacy XSPI driver models devicetree. The XSPI node has `#size-cells = <0>`, so its child nodes need to be addressed only by chip select. As a result, `ranges` can't be defined at a controller level (at the SoC-level devicetrees), where the window logically belongs, and has to be repeated in every flash node. The window is already defined once in the SoC dtsi, as the controller's second entry:

```dts
xspi1: spi@420d1400 {
	reg = <0x420d1400 0x400>, <0x90000000 DT_SIZE_M(256)>;
	...
};
```
The legacy driver simply didn't use it. 


This is the moment [@erwango](https://github.com/erwango) suggested [MSPI](https://docs.zephyrproject.org/latest/hardware/peripherals/mspi.html) as a solution for this problem, and gave me the hint that legacy STM32 QSPI, OSPI and XSPI drivers will be replaced with MSPI in the future Zephyr releases. The STM32 MSPI driver reads the memory-map base directly from `reg[1]`, so with MSPI the shield doesn't need to know about address at all. This was enough for me to start migration to the MSPI.

### 6. Convert the STM32U3 SoC XSPI node to MSPI controller
Commit: [*dts: arm: st: stm32u3: convert SoC XSPI node to MSPI controller*](https://github.com/zephyrproject-rtos/zephyr/pull/109004/changes/991917310866b7827e8fca22b49dd0547ce63b46)

At the time, the STM32U3 was a fairly new SoC, and only one board in Zephyr used it: the nucleo_u3c5zi_q, the same board I was working on. That meant I could start the migration from the root XSPI node (xspi1: spi@420d1400) and switch it to MSPI directly, without worrying about breaking other boards that depend on it.


With these changes, the xspi1 node has been converted from legacy to MSPI driver:

*dts/arm/st/u3/stm32u3.dtsi*
```diff
 xspi1: spi@420d1400 {
-	compatible = "st,stm32-xspi";
+	compatible = "st,stm32-xspi-controller";
 	reg = <0x420d1400 0x400>, <0x90000000 DT_SIZE_M(256)>;
 	interrupts = <76 0>;
 	clock-names = "xspix", "xspi-ker";
 	clocks = <&rcc STM32_CLOCK(AHB2_2, 4)>,
 		 <&rcc STM32_SRC_SYSCLK OCTOSPI_SEL(0)>;
+	clock-frequency = <DT_FREQ_M(50)>;
+	op-mode = "MSPI_OP_MODE_CONTROLLER";
 	#address-cells = <1>;
 	#size-cells = <0>;
 	status = "disabled";
 };
```

### 7. Convert the shield overlays to MSPI
Commit: [*boards: shields: st_b_m2mem_pack1: migrate to MSPI*](https://github.com/zephyrproject-rtos/zephyr/pull/109004/changes/5dc71b80ebf1ae9b5be80b4938236ba25439ad39)

With the controller converted, the shield overlays could follow. Every NOR variant moved from the legacy `st,stm32-xspi-nor` binding to the generic `jedec,mspi-nor`. The diff below shows one of the shield variants - `mb1927_33ba` (Macronix MX25LM51245):

*boards/shields/st_b_m2mem_pack1/b_m2mem_pack1_mb1927_33ba.overlay*
```diff
-	mx25lm51245: xspi-nor-flash@0 {
-		compatible = "st,stm32-xspi-nor";
+	m2mem_flash: mx25lm51245: mspi-nor-flash@0 {
+		compatible = "mxicy,mx25u", "jedec,mspi-nor";
 		reg = <0>;
 		size = <DT_SIZE_M(512)>; /* 512 Mbits */
-		ospi-max-frequency = <DT_FREQ_M(50)>;
-		spi-bus-width = <XSPI_OCTO_MODE>;
-		data-rate = <XSPI_DTR_TRANSFER>;
 ...
-		#address-cells = <1>;
-		#size-cells = <1>;
-		ranges = <0x0 0x90000000 DT_SIZE_M(64)>;
+		mspi-max-frequency = <DT_FREQ_M(50)>;
+		mspi-io-mode = "MSPI_IO_MODE_OCTAL";
+		mspi-data-rate = "MSPI_DATA_RATE_SINGLE";
+		mspi-hardware-ce-num = <0>;
+
+		read-command = <0xec13>;
+		write-command = <0x12ed>;
+		command-length = "INSTR_2_BYTE";
+		rx-dummy = <20>;
```

The most important change is the line that is gone: `ranges = <0x0 0x90000000 ...>`. The shield no longer contains any SoC-specific address, because the MSPI driver takes the memory-map base from the controller's `reg[1]`.

A few other changes came with the migration:
- **Stable `m2mem_flash` label**: every variant exposes its flash under the same label, so a sample can use `flash0 = &m2mem_flash;` without knowing which memory is populated.
- **`mspi_flash` sample overlay** for `nucleo_u3c5zi_q`, built on that label.
- **Dropped the `mb1928_18la` variant**: its M95P32 is a page EEPROM (512-byte pages, non-standard page-program opcode) rather than a regular NOR flash and needs the dedicated driver for itself. 

### 8. Add shield scenarios to the mspi_flash sample
Commit: [*samples: drivers: mspi_flash: add B_M2MEM_PACK1 shield scenarios*](https://github.com/zephyrproject-rtos/zephyr/pull/109004/changes/e5da174d34a4d2f59bb42f3d5b9d8fac5afd687b)

The `nucleo_u3c5zi_q` has no external flash of its own, so the existing `mspi_flash` scenario filters it out. To get the shield covered by CI and Twister, I added one scenario per variant. Each scenario passes the shield through `extra_args`:

*samples/drivers/mspi/mspi_flash/sample.yaml*
```yaml
  sample.drivers.mspi.flash.b_m2mem_pack1_mb1927_33ba:
    tags:
      - shield
    platform_allow:
      - nucleo_u3c5zi_q
    integration_platforms:
      - nucleo_u3c5zi_q
    extra_args: SHIELD=b_m2mem_pack1_mb1927_33ba
```

To keep the expected console output defined only once, I moved the shared `tags` and the console `harness_config` from the original scenario into the `common` section. The existing platform scenario and the four new shield scenarios now validate against the same erase/write/read sequence.

To build the sample for a specific variant (e.g. mb1927_33ba), pass the matching shield name to `west build`:

```shell
west build -b nucleo_u3c5zi_q samples/drivers/mspi/mspi_flash --shield b_m2mem_pack1_mb1927_33ba
```
