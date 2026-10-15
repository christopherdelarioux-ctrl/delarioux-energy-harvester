# delarioux-energy-harvester
An open-source, solid-state 240V energy harvester blueprint using compressed permanent magnetic fields and resonant graphene.

/*
 *  De La Rioux Solid-State Energy Harvester - Inverter Firmware
 *  Author: Christopher De La Rioux
 *  License: CERN-OHL-W-2.0 / CC BY-SA 4.0
 *  Target: Arduino / ESP32 Architecture (High-Speed PWM Module)
 *  
 *  Function: Drives a Full H-Bridge to output a 240V AC Pure Sine Wave.
 *            Features automatic frequency selection (50Hz/60Hz) via pin toggle.
 */

// Hardware Pin Definitions for H-Bridge Gate Drivers
const int Gate_A_High = 16;  // Top Left Switch
const int Gate_A_Low  = 17;  // Bottom Left Switch
const int Gate_B_High = 18;  // Top Right Switch
const int Gate_B_Low  = 19;  // Bottom Right Switch
const int Frequency_Select_Pin = 4; // Low = 50Hz, High = 60Hz

// Lookup table representing one half-cycle of a Pure Sine Wave (32 steps of resolution)
const int sineLookUp[] = {
  0, 25, 49, 74, 98, 121, 143, 164, 184, 202, 219, 233, 244, 252, 255, 255,
  252, 244, 233, 219, 202, 184, 164, 143, 121, 98, 74, 49, 25, 0
};
const int numSteps = sizeof(sineLookUp) / sizeof(sineLookUp[0]);

int stepIndex = 0;
bool activeHalfCycle = true; // true = Positive Half, false = Negative Half
unsigned long stepDelayMicros = 260; // Default timing step in microseconds

void setup() {
  // Configure High-Speed Gate Driver Pins as Outputs
  pinMode(Gate_A_High, OUTPUT);
  pinMode(Gate_A_Low, OUTPUT);
  pinMode(Gate_B_High, OUTPUT);
  pinMode(Gate_B_Low, OUTPUT);
  
  // Configure Frequency Selection Pin with internal pullup
  pinMode(Frequency_Select_Pin, INPUT_PULLUP);
  
  // Determine Grid Standard timing dynamically
  // Total period: 60Hz = 16,666 µs | 50Hz = 20,000 µs
  // Half period divided by step resolution sets step delay
  if (digitalRead(Frequency_Select_Pin) == HIGH) {
    stepDelayMicros = 16666 / (numSteps * 2); // ~260µs for 60Hz North America
  } else {
    stepDelayMicros = 20000 / (numSteps * 2); // ~312µs for 50Hz Europe/Asia
  }
}

void loop() {
  int pwmValue = sineLookUp[stepIndex];

  if (activeHalfCycle) {
    // ---- POSITIVE HALF CYCLE ----
    // Drive Left High and Right Low switches simultaneously
    analogWrite(Gate_A_High, pwmValue);
    digitalWrite(Gate_B_Low, HIGH);
    
    // Ensure opposing gates are completely dead/off to prevent short circuits
    digitalWrite(Gate_A_Low, LOW);
    analogWrite(Gate_B_High, 0);
  } else {
    // ---- NEGATIVE HALF CYCLE ----
    // Drive Right High and Left Low switches simultaneously
    analogWrite(Gate_B_High, pwmValue);
    digitalWrite(Gate_A_Low, HIGH);
    
    // Ensure opposing gates are completely dead/off
    digitalWrite(Gate_B_Low, LOW);
    analogWrite(Gate_A_High, 0);
  }

  // Progress to the next slice of the sine wave
  stepIndex++;
  
  if (stepIndex >= numSteps) {
    stepIndex = 0;
    activeHalfCycle = !activeHalfCycle; // Invert polarities at the zero-crossing point
  }

  // Maintain precise timing frequency
  delayMicroseconds(stepDelayMicros);
}

# 🛠️ GaN Gate Driver Technical Troubleshooting Guide
**Author:** Christopher De La Rioux  
**Target Hardware:** LMG1210 Gate Driver & EPC2104 GaN Bridge Array

Operating Gallium Nitride (GaN) HEMTs at multi-megahertz frequencies requires highly sensitive timing parameters. Because GaN switches lack a traditional body diode and have incredibly low gate-source thresholds, any signal noise or parasitic inductance can cause circuit failure. Use this diagnostic matrix on your benchtop.

---

## ⚡ Symptom 1: Excessive Thermal Overheating on the EPC2104 Bridge
* **Root Cause A: "Shoot-Through" Currents.** Both the High-Side and Low-Side GaN switches in a single half-bridge column are turning on at the same microsecond, creating a direct short-circuit between the 380V DC Bus and Ground.
* **Correction:** Adjust the physical resistor values on the LMG1210 `R_DEAD` pins. Increase the dead-time delay to a minimum of **10 to 15 nanoseconds** to guarantee that one switch is fully closed before its counterpart opens.
* **Root Cause B: Gate-Drive Overvoltage.** Driving the GaN gates beyond their maximum 5.5V limit causes lattice degradation and high leakage current.
* **Correction:** Verify that your low-dropout (LDO) linear regulator supplying the LMG1210 VDD is locked strictly at **5.0V DC**. Do not let it drift toward 6.0V.

---

## 📉 Symptom 2: Jagged or Distortion Waves on the 240V AC Output
* **Root Cause A: Parasitic Inductance Loom.** Long wire traces between the microchip controller and the gate driver are skewing the high-speed Pulse Width Modulation (PWM) signal.
* **Correction:** Re-layout the PCB Gerber files. The path between the MCU1 pin, the LMG1210 driver, and the EPC2104 gate must be **under 5mm total length** using wide, thick copper pours.
* **Root Cause B: LC Low-Pass Filter Saturation.** The inductor core inside the output filter is saturating under high electrical loads, turning your pure sine wave into a clipped square wave.
* **Correction:** Replace the standard ferrite-core inductor with a **Kool Mµ / Sendust powder-core inductor** that maintains linear permeability even under maximum current spikes.

---

## 🔬 Diagnostic Benchtop Validation Sequence
Before connecting your generator module to a live 380V DC supply:
1. **Isolated Logic Test:** Power only the 5V MCU and Gate Driver logic. Hook up your 200MHz Oscilloscope to the GaN gate pads. Verify you see clean, square 5V logic pulses with crisp vertical rising edges.
2. **Low-Voltage Sweep:** Introduce a safe 12V DC input to the power bus. Confirm the bridge switches cleanly and outputs a tiny, scaled 12V AC wave. Only when this wave is perfectly clean should you step the bus up to its full operating voltage.


SAFETY.md
# ⚠️ Core Safety Guidelines & High-Risk Mitigation Protocol
### Author: Christopher De La Rioux | Project: Solid-State Energy Harvester

Operating, constructing, or modifying this hardware involves high-density magnetic fields, vacuum containment, and high-voltage power generation. Strict adherence to these protocols is mandatory.

## 🧲 1. Kinetic and Kinetic-Impact Hazards (Samarium-Cobalt Arrays)
Samarium-Cobalt (SmCo) permanent magnets pose a severe risk of physical crushing, pinching, and laceration due to extreme, involuntary attraction/repulsion forces.
* **Impact Isolation:** Always maintain a minimum clear radius of 1 meter between loose magnets and any ferromagnetic tools, benches, or clothing items. 
* **Eye Protection:** SmCo is highly brittle. If two magnets collide violently, they will shatter into razor-sharp, high-velocity ceramic shrapnel. Impact-rated safety glasses are required at all times.
* **Assembly Tools:** Only use non-magnetic brass, copper, or heavy-duty Delrin plastic tools when pushing or adjusting components within the Halbach matrix.

## ⚡ 2. High-Voltage / High-Frequency Electrical Hazards
The integrated power stack steps up low-voltage currents to an operational **380V DC Bus** before inverting to grid-level **240V AC**.
* **Lethal Potential:** Any voltage exceeding 50V DC or 25V AC can deliver a fatal shock under standard atmospheric conditions. Treat all internal bus lines as active and live.
* **Capacitor Discharge Protocol:** After turning the system off, the high-voltage Silicon Carbide (SiC) stage capacitors retain energy. Wait a minimum of 10 minutes and measure the bus voltage with a calibrated multimeter before touching any circuit trace.
* **Insulated Boundary:** Ensure all high-voltage traces maintain proper creepage and clearance distances matching IPC-2221B industrial standards.

## 🔬 3. Nanomaterial Handling
The processing of wafer-scale graphene sheets or carbon lattices should be restricted to cleanroom or enclosed laboratory hoods. Do not crush, sand, or vaporize carbon matrices outside of a controlled, vacuum-isolated enclosure to prevent the inhalation of micro-particulates.

04_Power_Electronics_Inverter/BOM.md

Component ID	Quantity	Description	Manufacturer	Manufacturer Part Number	Package Type	Purpose / Placement
U1	1	650V 30A Full-Bridge GaN Power Element	Efficient Power Conversion (EPC)	EPC2104	Die-level BGA / QFN	Monolithic Active Rectifier Bridge
U2	1	High-Speed Gate Driver (Low-Side/High-Side)	Texas Instruments	LMG1210RVRR	WQFN-19	Drives GaN Gates at Megahertz Speeds
Q1, Q2	2	1200V 40A Silicon Carbide (SiC) MOSFET	Wolfspeed	C3M0075120K	TO-247-4	High-Voltage DC-to-DC Step-Up Stage
C1, C2	2	450V 100µF High-Ripple DC Link Capacitor	Panasonic	EEV-EE2W101M	Surface Mount	Filters High-Voltage 380V DC Bus
MCU1	1	32-bit ARM Cortex-M4 Microcontroller
