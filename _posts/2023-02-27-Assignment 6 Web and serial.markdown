---
layout: post
title:  "Assignment 6: Talking to the web! (via Serial)"
date:   2023-02-27 01:23:00 -0800
categories: jekyll update
---

## Demo
**Sending joystick data to webpage & receiving keyboard input from browser**  
Demo video showing the joystick controlling a ball on screen that changes color depending on its distance from the boarder of the frame. The LCD connected to Arduino prints out keystrokes entered on the webpage.

![Demo video]({{site.baseurl}}/assets/hw6_demo.gif){: width="75%"}

## Circuit drawing
The LCD is connected to the microcontroller via digital pins, 5V power, and ground. The pin for its backlight is connected in series to a 220Ω resistor to limit the current through the backlight as suggested by the guide found on Arduino Docs. https://docs.arduino.cc/learn/electronics/lcd-displays

The joystick is powered by the same 5V pin on Arduino and its two analog output pins are connected to port A0 and A1 for x & y axis respectively.

![Circuit drawing]({{site.baseurl}}/assets/hw6_circuit.png){: width="80%"}

![Circuit photo]({{site.baseurl}}/assets/hw6_circuit_photo.png){: width="80%"}


## Arduino code
{% highlight c %}
{% raw %}
// Import LCD library
#include <LiquidCrystal.h>

// Preprocessor macros
// Defines joystick pins for X & Y axis
#define X_PIN A1
#define Y_PIN A0  // Both maps to analog inputs

// Define LCD pins as constants
const int rs = 12, en = 11, d4 = 5, d5 = 4, d6 = 3, d7 = 2, d3 = 9, d0=6, d1=7, d2=8;
// Creates a LCD object
LiquidCrystal lcd(rs, en, d0, d1, d2, d3, d4, d5, d6, d7);
bool is_upper = false;

// Setup code: Run once
void setup() {
  // Set up serial communication
  Serial.begin(9600);
  // Initialize LCD object with the LCD size
  lcd.begin(16,2);
  // Move cursor to bottom right corner
  lcd.setCursor(16, 1);
  // Enable autoscroll
  lcd.autoscroll();
}

// Interactive program: Infinite loop
void loop() {
  // Read form both X and Y axis of the joystick
  int s1 = analogRead(X_PIN); // X-axis
  int s2 = analogRead(Y_PIN); // Y-axis
  // Manually generate data in JSON format
  Serial.print("[");  // Opening bracket for a JSON list
  Serial.print(s1);   // X-axis value
  Serial.print(",");  // Seperator
  Serial.print(s2);   // Y-axis value
  Serial.println("]");  // Closing bracket for a JSON list

  // Print text to LCD if valid data is received on Serial
  if(Serial.available() > 0){
    int inByte = Serial.read();     // Read serial data if it's available
    lcd.print(char(inByte));  // Print the data to LCD (cast to char)
  }
  // Slight program delay
  delay(50);
}
{% endraw %}
{% endhighlight %}

## p5.js code
{% highlight js %}
var serial; // variable to hold an instance of the serialport library
var portName = 'COM3'; //rename to the name of your port
var dataarray = []; //some data coming in over serial!
var xPos = 0;


function setup() {
  serial = new p5.SerialPort();       // make a new instance of the serialport library
  serial.on('list', printList);       // set a callback function for the serialport list event
  serial.on('connected', serverConnected); // callback for connecting to the server
  serial.on('open', portOpen);        // callback for the port opening
  serial.on('data', serialEvent);     // callback for when new data arrives
  serial.on('error', serialError);    // callback for errors
  serial.on('close', portClose);      // callback for the port closing
 
  serial.list();                      // list the serial ports
  serial.open(portName);              // open a serial port
  createCanvas(1200, 800);            // Create a 1200 x 800 canvas
  background(200, 200, 200);          // Background
}
 
// get the list of ports:
function printList(portList) {
 // portList is an array of serial port names
 for (var i = 0; i < portList.length; i++) {
 // Display the list the console:
   print(i + " " + portList[i]);
 }
}

// Log successful server connection to terminal
function serverConnected() {
  print('connected to server.');
}
 // Log port opening to terminal
function portOpen() {
  print('the serial port opened.')
}
 // Log serial connection error
function serialError(err) {
  print('Something went wrong with the serial port. ' + err);
}
 // Log serial port closure
function portClose() {
  print('The serial port closed.');
}

// Event listener: liustens for event over serial connection
function serialEvent() {
  // Check if there's data to read from serial
  if (serial.available()) {
    var datastring = serial.readLine(); // readin some serial
    var newarray; 
    try {
      newarray = JSON.parse(datastring); // attempt to parse as a JSON stream
      if (typeof newarray == 'object') {
        dataarray = newarray;
      }
      // Log to console
      console.log("got back " + datastring);
      } catch(err) {
      // got something that's not a json
    }
  } 
}

function graphData(newData) {
  // map the range of the input to the window height:
  var yPos = map(newData, 0, 1023, 0, height);
  // draw the line
  line(xPos, height, xPos, height - yPos);
  // at the edge of the screen, go back to the beginning:
  if (xPos >= width) {
    xPos = 0;
    // clear the screen by resetting the background:
    background(0x08, 0x16, 0x40);
  } else {
    // pass
  }
}

function drawSphere(x, y) {
  // clear the screen by resetting the background:
  background(200, 200, 200);
  let diameter = 40;
  // Maps Arduion analog read value to the dimensions of the frame
  let xPos = map(x, 0, 1023, 0, width);
  let yPos = map(y, 0, 1023, 0, height);  // Arduion 0-1023 to width/height
  // Change circle color depending on it's location within the frame
  if (abs(xPos - width/2)/width > 0.4 \|\| abs(yPos - height/2)/height > 0.4 ) {
    // Draws a red curcle if it's less than 10% of the width/height of the frame
    fill(color(256,0,0));   // Set shape filling color to red
  } else if(abs(xPos - width/2)/width > 0.1 \|\| abs(yPos - height/2)/height > 0.1 ) {
    // Draws a red curcle if it's less than 10% of the width/height of the frame
    fill(color(0,0,256));   // Set shape filling color to blue
  } else  {
    // Otherwise set to filling color to green
    fill(color(0,256,0));
  }
  // Draw a circle based on joystick movement
  circle(xPos, yPos, diameter);
}

// Draw function: pass data to draw the sphere
function draw() {
  let x = dataarray[0];
  let y = dataarray[1];
  drawSphere(x, y);
}

// Event listener: Waits for key press 
function keyPressed() {
	// Write the pressed key to serial (Arduino)
	serial.write(key);
}
{% endhighlight %}
