# Adding B_M2MEM_PACK1 shield support to Zephyr
- [PR #109004](https://github.com/zephyrproject-rtos/zephyr/pull/109004), merged **24.06.2026**.

## Intro
After I added upstream support for [STM32U3C5 SoC and NUCLEO-U3C5ZI-Q board](https://github.com/zephyrproject-rtos/zephyr/pull/105503), I observed the board and I got the idea to make one more contribution for it. I noticed that this board has a M.2 Key-A slot at the bottom side, and it spiked my curiosity.
After some research, I found out the intention behind this slot - the [B_M2MEM_PACK1](https://www.st.com/en/evaluation-tools/b-m2mem-pack1.html) shield.
<p align="center">
  <img src=".docs/nucleo_u3c5zi_q_M.2.jpeg" alt="NUCLEO-U3C5ZI-Q top and bottom view; the M.2 Key-A connector is highlighted on the underside of the board." width="760">
  <br>
  <em>The M.2 Key-A connector (highlighted) on the underside of the NUCLEO-U3C5ZI-Q, routed to the SoC's XSPI bus.</em>
</p>

## What is B_M2MEM_PACK1 shield?
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
Example: Every board that supports the arduino_r3_connector, must have a file `arduino_r3_header.dtsi` which has the `arduino_header` GPIO Nexus node which maps pins exposed on the board's SoC to the common pin-names on the interface itself. It looks like this:
```
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
2. **Shield side** - TODO: Explain from which files it consists of, what it does

TODO: Add example of some popular Zephyr shield

## The PR workflow
When I first got an idea to work on this PR, I didn't know the level of complexity it could gain, and that was the main thing that fed my curiosity. At the same time, I said to myself - _It's just the shield support, I have supported many boards in the past, what could be so complex about it?_ and _...but on the connector never supported before in Zephyr, which connects multiple IO XSPI protocol you never worked with..._. Nevertheless, I took the challenge, and this is how it went: 

### 1. XSPI controller support
Firstly, by reading the official B_M2MEM_PACK1 and NUCLEO_U3C5ZI_Q docs, I noticed that the M.2 connector is tied to **XSPI** peripheral, so I jumped to the docs and [AN5050](https://www.st.com/resource/en/application_note/an5050-getting-started-with-octospi-hexadecaspi-and-xspi-interface-on-stm32-mcus-stmicroelectronics.pdf) had all answers I needed. Long story short, the **XSPI** - e**X**tended **S**erial **P**eripheral **I**nterface, is the new interface that responds to the growing demand for faster, bigger external memories, triggered by richer graphics, wider range of multimedia and data-intensive applications in embedded systems, which require higher data throughputs and extra demands on the limited on-chip MCU memory. The XSPI is a flexible interface which can be configured as:

 1. **SPI** for 1-line data transmission
 2. **Dual-SPI** for 2-line data transmission
 3. **Quad-SPI** for 4-line data transmission
 4. **Octo-SPI** for 8-line data transmission
 5. **Hexadexa-SPI** for 16-line data transmission

I checked out the SoC devicetree files, and there wasn't XSPI node at all, I didn't include it in my SoC and board PR previously. I modified the `dts/arm/st/u3/stm32u3.dtsi` and added the node:
```
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
- **xspi1** is the nodelabel of the spi node, but on the internal SoC address where XSPI1 peripheral begins `0x420d1400` - **unit address**
- **compatible** property describes which bindings will be used for this node - in short, which driver will be used for this node
- **reg** property has two address windows, first one is the **register block** - `<0x420d1400 0x400>` where first address matches unit address, and second one represents size of the register space; Second address window represents predefined address space reserved for **memory-mapped** mode, start address of the memory-mapped region is `0x90000000` and size of that region can be maximally 256MB (This is the maximal size of the external memory we can do memory-mapping with, limitation of address space) 
_! Note that we will have problem later on with this !_ 
- **interrupts** property has two values, where first one describes which IRQ line is used for this node - IRQ line 76 is the _OCTOSPI1 global interrupt_ in the STM32U3 vector table [RM0487](https://www.st.com/resource/en/reference_manual/rm0487-stm32u3-series-armbased-32bit-mcus-stmicroelectronics.pdf); Second value represents the Zephyr interrupt priority.
- **clock-names/clocks** properties have two clocks, and this can be a little bit confusing. These two clocks actually have two different jobs: 

	- **xspix**	-> `STM32_CLOCK(AHB2_2, 4)` is the bus clock where bit 4 selects (`OCTOSPI1EN`) of `RCC_AHB2ENR2` (_2 sufix means second AHB2 enable register). This is the gate that makes the peripheral's AHB ports alive, and with this being closed, access to peripheral registers triggers bus fault.
	- **xspi-ker** -> `<&rcc STM32_SRC_SYSCLK OCTOSPI_SEL(0)>` is the kernel clock, the one that actually clocks the peripheral. This clock actually produces the bus clock.
- **#address-cells / #size-cells** - describes how **child** nodes are addressed: 
```
	#address-cells = <1>; -> One address cell
	#size-cells = <0>; -> No size cell
``` 
means the child is written as a xspi-nor-flash@0 where 0 is the chip-select index, not a memory address. _! This will be a problem later on...!_
- **status = "disabled"** - The node initializes in a disabled state.

### 2. M.2 connector support
As mentioned earlier, the B_M2MEM_PACK1 attaches to Nucleo boards through an M.2 Key A connector. A shield is only portable if the connector it plugs into is described once, centrally, and then reused by every board that carries it. So before writing anything shield-related, I had to define the connector.

TODO: Explain every commit by respecting hronology - feed the MSPI plot, and provide concrete reasons of the migration to MSPI API (Give detailed explanation of reg limitations that are being solved with MSPI) 