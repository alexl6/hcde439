---
layout: post
title:  "Assignment 1: Blink!"
date:   2023-01-14 22:13:07 -0800
categories: jekyll update
---

## Demo
![Demo video for 3 LEDs blinking randomly]({{site.baseurl}}/assets/hw1_demo.gif){: width="75%"}

Demo showing 3 LEDs connected to an Arudino microcontroller blinking randomly.


## Circuit drawing
![Circuit drawing]({{site.baseurl}}/assets/hw1_circuit.png){: width="80%"}

The circuit is is powered via USB connected to the Arudino microcontroller. The red, green, and blue LEDs are connected to pin 2, 3, and 4 respectively. Each LED is connected in series with a resistor to reduce the current through the LED.


## Calculating resistor values
Because each LED used in this assignment can operate safely with $\leq 30 \text{mA}$ of current, we need to use resisters to reduce the amount of current going through each LED.

Since we are connecting the resister & the LED in series, the amount of current through the LED is the same as that for the resister ($I_\text{L} = I_\text{r}$). This also means that the sum of voltage drop across the resister and LED should be equal to the voltage provided by the Arduino microcontroller (5V).

With these information, we calculate the minimum resistance by applying Ohm's law on the resister.
$$
\begin{align*}
  V_\text{r} &= I_\text{r} R_\text{r} && \text{Ohm's law}\cr
  V_\text{PSU} - V_\text{L} &= I_\text{L} R_\text{r} &&\text{Substitute resister V & I}\cr
  5\texttt{V} - V_\text{L} &= 20\texttt{mA} \cdot R_\text{r}\cr
\end{align\*}
$$


A 220 Ω resister is used for red and green LEDs because these LEDs have a voltage drop of 1.7V. Their resisters' resistance to be at least
$$
\begin{align\*}
  V&=IR\cr
  5\;\texttt{V} - 1.7\;\texttt{V} &=20\;\texttt{mA} \cdot \textrm{R}\cr
  \frac{3.3\;\texttt{V}}{0.02\;\texttt{A}} &= \textrm{R}\cr
  \textrm{R} &= 165 \;\Omega
\end{align\*}
$$

A 100 Ω resister is used for the blue LED (voltage drop = 3.3V). It needs an resistor that is at least
$$
\begin{align\*}
    V&=IR\cr
    5\;\texttt{V} - 3.3\;\texttt{V} &=20\;\texttt{mA} \cdot \textrm{R}\cr
    \frac{1.7\;\texttt{V}}{0.02\;\texttt{A}} &= \textrm{R}\cr
    \textrm{R} &= 85 \;\Omega
\end{align\*}
$$


## Arduino code:
{% highlight c %}
// Preprocessor macros for LED -> pin mapping & No. of LEDs
#define RED_PIN 2
#define GREEN_PIN 3
#define BLUE_PIN 4
#define NUM_LEDS 3

// Array of LEDs by their pin
int led_pins[NUM_LEDS] = {RED_PIN, GREEN_PIN, BLUE_PIN};
int i;  // Loop variable

void setup() {
  // Set all used pins to output mode
  for (i= 0; i < NUM_LEDS; i++){  // Loop through every LED
    pinMode(led_pins[i], OUTPUT); // set a single pin to output
  }
}

void loop() {
  // Modify the state of every connected LED (might remain the same)
  for(i = 0; i < NUM_LEDS; i++){  // Loop through every LED
    digitalWrite(led_pins[i], random(2)); // Randomly decide if this LED is on or off
  }
  delay(random(500, 1500)); // Random delay between 500 and 1500 ms
}
{% endhighlight %}