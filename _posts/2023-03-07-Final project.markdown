---
layout: post
title:  "Final project"
date:   2023-03-07 16:50:00 -0800
categories: jekyll update
---

## Concept
The concept of this project is to make a cheap tiny arcade box that plays a clone of the classic arcade game _Pong_ on a 16x2 character LCD display. This is a “spiritual successor(ancestor)” to the first video game I ever developed which was a clone of _Breakout_ written in Processing. The challenge is to build a game with graphics using very limited hardware.

## Hardware implementation
The hardware includes two joysticks installed on the side of a character LCD mounted on the top of a cardboard box. The LCD is connected to the microcontroller via 8 digital pins, 5V power, and ground. The pin for its backlight is connected in series to a 220Ω resistor to limit the current through the backlight as suggested by the guide found on [Arduino Docs](https://docs.arduino.cc/learn/electronics/lcd-displays). A potentiometer installed on the breadboard is used to control the contrast of the display. It is hidden in the box to prevent users from accidentally changing its value.

Since the game experience heavily depends on the smoothness of its graphics & responsiveness of controls, I used all 8 digital pins for the LCD in order to get a small increase in LCD write speed & reduction in latency.

The joysticks are powered by the microcontroller and their y-axis pins are connected to the analog input pins on the Arduino board.

### Circuit drawing

![Circuit drawing]({{site.baseurl}}/assets/final_circuit.png){: width="80%"}

### Circuit photos
View of the circuit before it was mounted inside the box

![Circuit before mounting]({{site.baseurl}}/assets/final_circuit_photo0.png){: width="70%"}

View of the circuit mounted inside the box

![Mounted circuit view (left)]({{site.baseurl}}/assets/final_circuit_photo1.png){: width="49%"}
![Mounted circuit view (right)]({{site.baseurl}}/assets/final_circuit_photo2.png){: width="49%"}

Final product with mounted joysticks & LCD

![Final product with mounted joysticks & LCD]({{site.baseurl}}/assets/final_circuit_photo2.png){: width="70%"}


## Software implementation

### Overview
The software implementation is written in Arduino language (C/C++ derivative) and includes the built-in LCD library `LiquidCrystal.h`. My firmware is written in C-style code (plus a few C++ features). The main building blocks include game simulation and graphics rendering.

The flowchart below gives an overview of the overall program logic.

![Flowchart showing the high-level program logic]({{site.baseurl}}/assets/final_flowchart.png){: width="90%"}

### Graphics
The main challenge of this project is to draw the game graphics on a character LCD which does not allow individual pixels to be addressed directly. The hardware also lacks suppport for sprites and smooth scrolling. My graphics code implementation simulates those effects using 5 custom characters (2 per paddle, 1 for the ball). It mostly abstracts the LCD screen from the game simulation code, making it relatively easy to implement the actual game logic.

Each custom character is represented as a bitmap stored in an 8-byte array. A 1 in the bitmap represents a pixel on the LCD. For example, the following array will create a custom character that looks like a paddle on the right side of the screen.

![Bitmap]({{site.baseurl}}/assets/final_bitmap_example.png){: width="20%"}

{% highlight c %}
{% raw %}
byte customChar[] = {
  B00000,
  B00001,
  B00001,
  B00001,
  B00001,
  B00000,
  B00000,
  B00000
};
{% endraw %}
{% endhighlight %}

Because the `lcd.createChar()` function will create a custom character based on the first 8 bytes it reads, I can achieve the effect of moving the paddle up/down by changing where the pointer is pointing within the byte array. Because the padle might span across two vertically stacked characters, two custom characters are used per paddle. Finding the right location within the byte array bitmap is handled by `get_paddle_bitmap()`.

The ball character is generated in a similar manner using custom characters. Because the ball can move both vertically and horizontally across the entire screen, its content and the location of that character needs to change over time. Everytime the ball character is updated, the program will clear the charater at its previous location, zero out the entire array, then toggle the correct bit to generate the new bitmap. This task is primarily handled by `ball_sprite()`.

As the ball gets closer and close to one of the paddles on screen, it will eventually need to share a custom character with one of the paddles. This is handled by `draw_merge_sprite()` that 'copies' the paddle bitmap and merge it with the ball sprite before drawing it.

### Game simulation
The game simulation code simlates a _Pong_ game (it is not physically accurate, however). It reads in player input from the joystick, updates the paddles on screen, and moves the ball around the screen. While the graphics code supports letting the ball move in any direction, I decided to limit it to diagnal motion (45 degrees) after testing the actual display's clarity & responsiveness with different motion patterns. The pong ball will bounce and change direction if it hits the top/bottom wall or a player controlled paddle. The progam delay at the end of `run_session()` dictates the game speed.

To improve playability, each paddle is 1 pixel wider in the game logic than it appears on-screen. This is done to compensate the 'ghosting' effect of the LCD character display, and it also makes the game feel easier & more forgiving to the players. Because the ball always move in 45 degrees angle, it will also help make the game look more 'visually correct'.

The game runs at just over 10 fps (frames per second) as determined by the program delay. The main limiting factor is the character LCD as opposed to the software. In my testing, the poor pixel response time makes it very difficult to see the moving ball and paddles if I attempt to run the game at a faster speed. I might be able to resolve this by improving the ball drawing code, but a hardware upgrade to an OLED screen would yield a much more significant improvement. 


## Demo video
![]({{site.baseurl}}/assets/final_demo.mp4)


## Arduino code
{% highlight c %}
{% raw %}
// Include external libraries
#include <LiquidCrystal.h>
#include <Arduino.h>

// Preprocessor macros
// Defines LCD pins
#define RS 12
#define ENABLE 11
#define D0 6
#define D1 7
#define D2 8
#define D3 9
#define D4 5
#define D5 4
#define D6 3
#define D7 2
#define PD_L 4
#define PD_R 11

// helper enum for handling overlapping ball & paddle
enum DrawOption{regular, upper_left, lower_left, upper_right, lower_right};


// Global variables
// Initializes LCD object with the predefined pins
LiquidCrystal lcd(RS, ENABLE, D0, D1, D2, D3, D4, D5, D6, D7);

// Source bitmap for ball sprite
byte ball[8] = {
  0b00000,
  0b00000,
  0b00000,
  0b00000,
  0b00000,
  0b00000,
  0b00000,
  0b00000
};

// Source bitmap for paddle sprite
byte paddle_src[17] = {
  0b00000,
  0b00000,
  0b00000,
  0b00000,
  0b00000,
  0b00000,
  0b00000,
  0b00000,
  0b00001,
  0b00001,
  0b00001,
  0b00001,
  0b00000,
  0b00000,
  0b00000,
  0b00000,
  0b00000
};

// Struct for holding the two byte pointers pointing to the location to represent
// paddles
struct paddle_loc {
  byte* upper;
  byte* lower;
};

// Struct for holding a pair of coordinates or (col, row) pair
struct coordinate {
  int8_t x;
  int8_t y;
};


// Declare helper functions
// Flips the character bitmap for paddle from left to right or vice versa
// @param src: source bitmap for paddle
// @param to_right: whether to flip the paddle to the right or left
void flip_paddle_bitmap(byte* src, bool to_right);

// Generate a struct of two pointers pointing to the starting address within the bitmap
// that will be drawn as a paddle in game
// @param src: A byte pointer to the source bitmap for paddles
// @param posn: The paddle's position
// @return: A paddle_loc struct with 2 pointers to where the upper & lower bitmap should start
struct paddle_loc get_paddle_bitmap(byte* src, int8_t posn);

// Handles drawing for overlapping ball & paddle sprite
// @param b_loc: The character location to draw the merged sprite
// @param ball: Byte pointer to the btimap for the ball
// @param paddle: Byte pointer to the bitmap of the paddle
void draw_merged_sprite(struct coordinate& b_loc, byte* ball, byte* paddle);

// Generate the character bitmap for the ball at `posn`, calculates the character location,
// and clears the previous character location that contained the ball
// `ret_val` is a return variable
// @param posn: Current coordinate of the ball
// @param prev: Previous coordinate of the ball
// @param ret_val: Return variable, returns the character location containing the ball
// @return: A byte pointer to the ball sprite
byte* ball_sprite(const struct coordinate& posn, const struct coordinate& prev, struct coordinate& ret_val);

// Draws a custom character at a given character location on screen
// @param custom_char: The custom character to use (betwwen 0 and 7)
// @param bitmap: A byte pointer to the bitmap to apply to this character
// @param col: The column to place the character
// @param row: The row to place the character
void draw_char(const uint8_t& custom_char, byte* bitmap, const uint8_t& col, const uint8_t& row);

// Reads joystick input from a given analog pin
// @param:  Analog pin to read
// @return: an int16_t between -2 and 2
int16_t read_input(uint8_t pin);

// Renders the game graphics based on the ball location/sprite
// @param b_loc: A coordinate struct representing the ball's character location
// @param b: A byte pointer pointing to the bitmap for ball sprite
// @param paddle_src: Pointer to the source bitmap for paddles
// @param paddle_l: Represents the left paddle's position
// @param paddle_r: Represents the right paddle's position
void render(struct coordinate& b_loc, byte* b, byte* paddle_src, int8_t paddle_l, int8_t paddle_r);


// Main program
// Setup code, run once
void setup() {
  // set up the LCD's number of columns and rows:
  lcd.begin(16, 2);
  // Disable cursor
  lcd.noCursor();
}

// Loop: Repeatly run the game
void loop() {
    run_session();
    delay(100);
}


// Helper function implementations
int16_t read_input(uint8_t pin){
  // Read from the supplied analog pin then subtract 511
  int16_t reading = analogRead(pin) - 511;
  // If the joystick is pushed to the extremes, return -2 or 2
  if(abs(reading) > 495){
    return reading < 0 ? -2 : 2;
  } else if(abs(reading) > 200){  // or return -1 or 1
    return reading < 0 ? -1 : 1;
  } else {  // Joystick is in deadzone, return 0
    return 0;
  }
}

void run_session(){
  // Struct for holding the ball character location
  struct coordinate ball_char_loc;

  // Coordinate structs holding the ball's current and previous position.
  // coordiantes are absolute to the top left corner of the screen in logical pixels
  struct coordinate curr_posn;
  struct coordinate prev_posn;

  // Represents the current velocity of the ball in x and y directions
  int v_x;
  int v_y;

  // Pointer to the ball sprite
  byte* b;

  // Represents the left & right paddle's position
  int8_t paddle_l;
  int8_t paddle_r;

  // Initialize ball position
  curr_posn = {45, (int8_t)(millis() % 17)};
  prev_posn = {0, 0};

  // Generate initial velocity
  v_x = millis() % 2 == 0 ? 1 : -1;
  v_y = 1;
  // Initial paddle location
  paddle_l = 3;
  paddle_r = 3;
  bool sensor_clock = true;
  // Tracks game state
  bool is_alive = true;
  while(is_alive){
    // Optional: read from joystick every/every other program cycle
    if(sensor_clock){
      paddle_r += read_input(A0);
      paddle_l -= read_input(A1);
      // sensor_clock = false;  // Comment or uncomment this line to half the paddle speed
    } else {
      sensor_clock = true;
    }
    // Constrain paddles to valid range
    paddle_l = constrain(paddle_l, 0, 13);
    paddle_r = constrain(paddle_r, 0, 13);

    // Update current position based on x,y velocities
    curr_posn.x+= v_x;
    curr_posn.y+= v_y;
    
    // Bounce off the top/bottom of the screen
    if (curr_posn.y <= 0 || curr_posn.y >= 16){
      v_y *= -1;
    }
    
    // Checks if the ball will land on the left paddle
    if (curr_posn.x <= 24){
      // Bounce the ball if it will land on the left paddle
      if(curr_posn.y >= paddle_l-1 && curr_posn.y < paddle_l + 5){
        v_x *= -1;
      } else {  // Otherwise end the game
        is_alive = false;
      }      
    }

    // Checks if the ball will land on the right paddle
    if (curr_posn.x >= 70){
      // Bounce the ball if it will land on the right paddle
      if(curr_posn.y >= paddle_r-1 && curr_posn.y < paddle_r + 5){
        v_x *= -1;
      } else {  // Otherwise end the game
        is_alive = false;
      }      
    }

    // Get the bitmap & character location for the ball
    b = ball_sprite(curr_posn, prev_posn, ball_char_loc);
    
    // Save this position (x,y)
    prev_posn.x = curr_posn.x;
    prev_posn.y = curr_posn.y;

    // Draw graphics based on ball & paddle locations
    render(ball_char_loc, b, paddle_src, paddle_l, paddle_r);
    
    // Program delay, controls game speed & latency
    delay(70);
  }
  // Add a longer delay when a player wins
  delay(2000);
}


// Flips the character bitmap for paddle from left to right or vice versa
void flip_paddle_bitmap(byte* src, bool to_right) {
  uint8_t i = 8;
  // Right shift by 4 (to get right paddle)
  if (to_right) {
    for (; i < 12; ++i) {
      src[i] = src[i] >> 4;
    }
    return;
  }
  // Left shift by 4 (to get left paddle)
  for (; i < 12; ++i) {
    src[i] = src[i] << 4;
  }
}


struct paddle_loc get_paddle_bitmap(byte* src, int8_t posn) {
  struct paddle_loc ret_val;
  // Calculate upper character for the paddle
  if (posn <= 8) {
    ret_val.upper = src + (8 - posn);
  } else {
    ret_val.upper = src;
  }
  // Calculate lower character for the paddle
  if (posn <= 5) {
    ret_val.lower = src;
  } else {
    ret_val.lower = src + (17 - posn);
  }
  return ret_val;
}


byte* ball_sprite(const struct coordinate& posn, const struct coordinate& prev, struct coordinate& ret_val){
  // Calculate where the pixel should be within the character
  int8_t relative_x = posn.x % 6;
  int8_t relative_y = posn.y % 9;
  
  // Calculate the row & column to place the sprite
  ret_val.x = posn.x / 6;
  ret_val.y = posn.y / 9;
  *((uint64_t*)ball) = 0;   // Reset ball sprite to empty

  // Ball is on the boundary between characters
  // Clear previous character and return
  if (relative_x == 5 || relative_y == 8){
    // Move cursor and directly write an empty charcter
    lcd.setCursor(prev.x / 6, prev.y / 9);
    lcd.write(byte(32));
    return ball;
  }

  // Clear the previous character if the ball is now on a different character
  if (posn.x != prev.x/6 || posn.y != prev.y/9){
    // Move cursor and directly write an empty charcter
    lcd.setCursor(prev.x / 6, prev.y / 9);
    lcd.write(byte(32));
  }

  // Create the new ball sprite
  ball[relative_y] = 0x1 << (4-relative_x); // Shift the ball pixel
  return ball;
}


void draw_merged_sprite(struct coordinate& b_loc, byte* ball, byte* paddle){
  *(uint64_t*)ball |= *(uint64_t*)paddle;
  draw_char(6, ball, b_loc.x, b_loc.y);
}


void draw_char(const uint8_t& custom_char, byte* bitmap, const uint8_t& col, const uint8_t& row){
  // Create custome character from bitmap
  lcd.createChar(custom_char, bitmap);
  // Move cursor
  lcd.setCursor(col, row);
  // Write custome character
  lcd.write(byte(custom_char));
}


void render(struct coordinate& b_loc, byte* b, byte* paddle, int8_t paddle_l, int8_t paddle_r) {
  // Determine the necessary draw options
  DrawOption opt = regular; // Assume ball does not overlap with any paddles
  
  // Optionally merge with left/right paddle
  if (b_loc.x == PD_L){ // left
    opt = (b_loc.y == 0) ? upper_left : lower_left;   // overlap with upper or lower half?
  } else if (b_loc.x == PD_R) { // right
    opt = (b_loc.y == 0) ? upper_right : lower_right; // overlap with upper or lower half?
  }
  
  // Get bitmap for right paddles
  struct paddle_loc pd = get_paddle_bitmap(paddle, paddle_r);

  // Draw the upper half right paddle
  if(opt == upper_right){
    // Merge upper half of right paddle with the ball
    draw_merged_sprite(b_loc, b, pd.upper);
  } else {
    // draw directly
    draw_char(2, pd.upper, PD_R, 0);
  }
  // Draws the lower half of right paddle
  if(opt == lower_right){
    // Merge lower half of right paddle with the ball
    draw_merged_sprite(b_loc, b, pd.lower);
  } else {
    // draw directly
    draw_char(3, pd.lower, PD_R, 1);
  }

  //  Get right paddle bitmap & flip the 1 bits to left side
  pd = get_paddle_bitmap(paddle, paddle_l);
  flip_paddle_bitmap(paddle, false);

  // Draws the upper half of left paddle
  if(opt == upper_left){
    // Merge upper half of left paddle with the ball
    draw_merged_sprite(b_loc, b, pd.upper);
  } else{
    // draw directly
    draw_char(0, pd.upper, PD_L, 0);
  }
  // Draws the lower half of left paddle
  if(opt == lower_left){
    // Merge lower half of right paddle with the ball
    draw_merged_sprite(b_loc, b, pd.lower);
  } else {
    // draw directly
    draw_char(1, pd.lower, PD_L, 1);
  }

  // Flips the paddle bitmap back to its original position
  flip_paddle_bitmap(paddle, true);

  // Draws the ball separately if it doesn't overlap with anything
  if(opt == regular){
    draw_char(6, b, b_loc.x, b_loc.y);
  }
}
{% endraw %}
{% endhighlight %}

