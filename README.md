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
