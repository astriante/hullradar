# HullRadar: Battery-Free Wireless Hull Integrity Sensors for Spacecraft

this is a proof-of-concept for a battery-free, ultra-low-power wireless sensor node built for aerospace structures. instead of running bulky cables or relying on lithium batteries that fail in deep space thermal extremes, **hullradar** uses ambient rf backscatter physics to send telemetry data using basically zero power.

by using an msp430 mcu to rapidly toggle an rf switch, the node constantly changes its antenna impedance mid-air. this forces existing 802.11b wi-fi waves to alternate between absorbing and reflecting, which encodes the sensor data onto the wave. on the receiving end, i built a custom linux pipeline using `libpcap` to force monitor mode, grab raw radiotap headers, bypass the kernel's automatic hardware fcs drops, and decode the payload via bitwise xor operations.


---

## 1. Why this was built (The Problem)

monitoring things like structural stress, micro-meteoroid hits, or tiny leaks on a spacecraft hull usually turns into a huge engineering problem :

**cabling weight :** running wires for data and power adds way too much deadweight, and every extra kilo counts.

**battery issues :** normal batteries can leak, freeze, or just stop working completely under extreme temperatures.

hullradar solves this by using **ambient backscatter communication**. instead of generating its own power-hungry radio signal, the node hitches a ride on wi-fi waves already floating around the vessel. it shifts all the heavy lifting and power needs to an external gateway, so the actual sensor tag can run on basically nothing (sub-milliwatt envelope).


---

## 2. Core Architecture

The system pipeline is split across three layers :
```
[ Ambient 802.11b Tx ] ----> ( Radio Waves )
|
v
[ Linux Gateway (C/pcap) ] <-- [ HullRadar Passive Tag (MSP430) ]
```
### 2.1 The Passive Tag (Hardware Layer)
the hardware on the tag is pretty discreet and optimized for ultra-low power consumption:

**mcu :** msp430fr5994. used this because it has non-volatile fram instead of regular flash memory. it boots up instantly on a tiny trickle charge and won't flip bits or crash from cosmic radiation (seus)

**rf switch :** adg901 spst switch. connected directly to a 2.4 ghz printed copper antenna to handle the physical reflections.

**power :** powercast p2110 harvester chip. it grabs the ambient radio waves, runs them through a rectifier, and dumps the power into a supercapacitor instead of a heavy battery.

---

## 3. How It Works (The Physics & Logic)

### 3.1 Antenna Impedance Modulation
the physics layer here is pretty straightforward. when a radio wave hits an antenna, it either bounces back or gets absorbed depending on how the load impedance is matched. i'm using the msp430 to drive a gpio pin straight into the adg901 rf switch to flip this boundary state on the fly:

**state 0 (absorptive) :** the antenna impedance matches the incoming wave, so the rf energy passes right through to trickle-charge the supercap and power the chip.

**state 1 (reflective) :** the antenna impedance is deliberately mismatched, forcing the incoming wave to bounce right back into the air as a reflected packet.

```c
// High-speed register manipulation for precise bit-period timing
void transmit_backscatter_bit(uint8_t bit) {
    if (bit) {
        P1OUT |= BIT0;  // High: State 1 (Reflective Mismatch)
    } else {
        P1OUT &= ~BIT0; // Low: State 0 (Absorptive Match)
    }
    __delay_cycles(20); // Keep timing aligned with the carrier bit-rate
}
```

### 3.2 Overriding the Kernel Checksum Filter
Because the tag alters the wave bits *mid-air after* the transmitter already calculated and sent the packet, the original 4-byte Frame Check Sequence (FCS) checksum floating at the end of the Wi-Fi frame becomes invalid.

Normally, Linux network cards calculate checksums in the silicon layer and instantly destroy corrupt frames before they ever hit the OS. To bypass this, my receiver software configures raw socket interfaces to explicitly force the driver stack to pass up broken frames via `libpcap` hooks:

```c
// Sniffer initialization snippet
pcap_t *initialize_receiver_interface(const char *dev) {
    char errbuf[PCAP_ERRBUF_SIZE];
    pcap_t *handle = pcap_create(dev, errbuf);
    
    // Force monitor mode within pcap context
    pcap_set_rfmon(handle, 1);
    pcap_activate(handle);
    
    // Enforce raw Radiotap link layers so the kernel preserves bad FCS trailers
    if (pcap_set_datalink(handle, DLT_IEEE802_11_RADIO) < 0) {
        fprintf(stderr, "Failed to enforce raw Radiotap parsing link layer.\n");
    }
    return handle;
}
```

Once `libpcap` grabs the raw buffer from the Radiotap layer, the backend extracts the payload and extracts the sensor telemetry by applying a bitwise XOR operation against the original baseline transmitter packet array:

$$\text{Data} = \text{Excitation Frame} \oplus \text{Captured Frame}$$

---

## 4. Current Project Status
right now the core pipeline architecture is fully mapped out. hullradar is basically a working blueprint of how u can use smart low-level software to bypass hardware limits and route data across thick shielding barriers without needing an active transmitter.next step is moving past the design layout and filling out the repo with the actual functional c files for the msp430 firmware and the linux libpcap sniffer so anyone can clone it and build it.


