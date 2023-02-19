---
layout: post
title:  "Assignment 5: Higher Voltage and Transistors!"
date:   2023-02-18 23:20:00 -0800
categories: jekyll update
---

## Demo
**Controlling LED strip with IR remote**  
Demo video for LED strip controlled by IR remote.
![Demo video showing LED strip controlled by IR remote]({{site.baseurl}}/assets/hw5_demo.gif){: width="75%"}

## Circuit drawing
The IR receiver is connected to the microcontroller's digital pins. The LED strip is connected to 12V power and is controlled by the transistor connected to Arduino's digital output pin & ground.
![Circuit drawing]({{site.baseurl}}/assets/hw5_circuit.png){: width="80%"}

The IR receiver is powered by the 5V pin on the microcontroller, and the LED strip is powered by an external power supply instead. By looking at the labels, I was able to determine that the light strip is rated at 12V and 1.5A which is within the PSU's capacity of supplying 12V and 2A. The current drawn by the LED strip is also within the rated maximum drain current of 32A (at 25°C) for the FQP30N06L-ND transistor.

![Circuit photo]({{site.baseurl}}/assets/hw5_circuit_photo.png){: width="80%"}


## Arduino code
{% highlight c %}
{% raw %}
// Include external libraries
#include <Arduino.h>
#include <IRremote.hpp>

// Preprocessor macros for input and output pin etc.
#define IR_RECEIVE_PIN 11  // IR receiver pin
#define DECODE_NEC        // Use NEC decode for IR signal

#define LED_PIN 6 // LED pin

// State variable: LED brightness
uint8_t led_brightness = 25;
// State variable: LED is on/off
bool led_on = false;
// Tracks if the power button was pressed
bool button_down = false;
// Tracks the time of last button press
unsigned long last_btn_press = 0;

// Setup code, run once
void setup() {
  // Initialize Serial communications
  Serial.begin(9600);
  // Set LED to output mode
  pinMode(LED_PIN, OUTPUT);
  // Start the receiver and if not 3 parameter specified, take LED_BUILTIN pin from the internal boards definition as default feedback LED
  IrReceiver.begin(IR_RECEIVE_PIN, ENABLE_LED_FEEDBACK);
  // Print ready prompt to terminal
  Serial.print(F("Ready to receive IR signals of protocols: "));
  printActiveIRProtocols(&Serial);    // Print active IR protocol used
}

// Interactive code, infinite loop
void loop() {
  // Check for received data & attempt to decode
  if (IrReceiver.decode()) {
    // Process if the received 
    if(IrReceiver.decodedIRData.protocol == NEC){
      // IrReceiver.decodedIRData.command
      // Print raw IR data to terminal
      Serial.println(IrReceiver.decodedIRData.command);
      // Switch statements is used to handle different button presses
      switch(IrReceiver.decodedIRData.command){
        case 12:  // User pressed 1
          led_brightness = 25; // lower brightness
          break;
        case 24:  // User pressed 2
          led_brightness = 90; // higher brightness
          break;
        case 69:  // User pressed power button
          if (!button_down){  // If power button wasn't already pressed
            led_on = !led_on; // Toggle led state
            button_down = true;   // Set power button as pressed
          }
          break;
      }
      // Update last_btn_press to reflect most recent button press
      last_btn_press = millis();
    }
    // Resume IR data receive
    IrReceiver.resume(); // Enable receiving of the next value
  } else {  // IR info not decoded correctly
    // Reset button state of the last press was more than 200ms ago
    if (millis()-last_btn_press > 200)
      button_down = false;
  }
  // Debug prints
  // Prints expected led state & brightness to terminal via serial
  Serial.print(led_on);
  Serial.print("   ");
  Serial.println(led_brightness);
  // If LED is supposed to be on
  if(led_on){ 
    analogWrite(LED_PIN, led_brightness); // Set LED to specified brightness
  } else {  // Otherwise it's supposed to be off
    digitalWrite(LED_PIN, LOW);// Turn off LED
  }
  // Program loop delay
  delay(20);
}
{% endraw %}
{% endhighlight %}