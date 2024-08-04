#RUN & JUMP Arduino Game Using LCD
##Project Overview
This project implements an interactive Arduino game where the player controls a character that can run and jump using a button. The game state is displayed on a 16x2 LCD screen.

##Authors
Shresth Deorari (2201MC38)
Kashika Aggarwal (2201CS35)
##Date
December 5, 2023

##Table of Contents
1)Project Aim
2)Components Required
3)Methodology
4)Project Overview
5)Source Code
6)Acknowledgments
7)References
8)Figures and Pictures
##Project Aim
The aim of this project is to create an interactive Arduino game using an LCD display. The player can control a character to run and jump using a button, and the game state is displayed on the LCD.

##Components Required
Arduino Nano
16x2 LCD display
I2C LCD driver
Breadboard
Push Button
Potentiometer
Connecting Wires
##Methodology
The project uses an Arduino Nano connected to an LCD over I2C. The connections are as follows:

Analog Pin A4: SDA pin of I2C
Analog Pin A5: SCL pin of I2C
Digital Pin 2: Connected to the push button to serve as an interrupt
The circuit is designed on a breadboard, ensuring correct connections. If the LCD screen is not displaying correctly, adjust the potentiometer on the I2C driver to modify the LCD contrast level and check the address of the I2C driver.

##Project Overview
The project successfully implements a game where the player can control character movements on an LCD display. The use of interrupts adds interactivity, enhancing the user experience. Troubleshooting steps are provided to address potential issues with the LCD display.

##Source Code
The source code for the RUN & JUMP game using Arduino Nano is provided below:

#include <LiquidCrystal.h>

#define PIN_BUTTON 2
#define PIN_AUTOPLAY 1
#define PIN_READWRITE 10
#define PIN_CONTRAST 12

#define SPRITE_RUN1 1
#define SPRITE_RUN2 2
#define SPRITE_JUMP 3
#define SPRITE_JUMP_UPPER '.'
#define SPRITE_JUMP_LOWER 4
#define SPRITE_TERRAIN_EMPTY ' '
#define SPRITE_TERRAIN_SOLID 5
#define SPRITE_TERRAIN_SOLID_RIGHT 6
#define SPRITE_TERRAIN_SOLID_LEFT 7

#define HERO_HORIZONTAL_POSITION 1
#define TERRAIN_WIDTH 16

#define TERRAIN_EMPTY 0
#define TERRAIN_LOWER_BLOCK 1
#define TERRAIN_UPPER_BLOCK 2

#define HERO_POSITION_OFF 0
#define HERO_POSITION_RUN_LOWER_1 1
#define HERO_POSITION_RUN_LOWER_2 2
#define HERO_POSITION_JUMP_1 3
#define HERO_POSITION_JUMP_2 4
#define HERO_POSITION_JUMP_3 5
#define HERO_POSITION_JUMP_4 6
#define HERO_POSITION_JUMP_5 7
#define HERO_POSITION_JUMP_6 8
#define HERO_POSITION_JUMP_7 9
#define HERO_POSITION_JUMP_8 10
#define HERO_POSITION_RUN_UPPER_1 11
#define HERO_POSITION_RUN_UPPER_2 12

LiquidCrystal lcd(11, 9, 6, 5, 4, 3);

static char terrainUpper[TERRAIN_WIDTH + 1];
static char terrainLower[TERRAIN_WIDTH + 1];
static bool buttonPushed = false;

void initializeGraphics(){
    static byte graphics[] = {
        // Run position 1
        B01100, B01100, B00000, B01110, B11100, B01100, B11010, B10011,
        // Run position 2
        B01100, B01100, B00000, B01100, B01100, B01100, B01100, B01110,
        // Jump
        B01100, B01100, B00000, B11110, B01101, B11111, B10000, B00000,
        // Jump lower
        B11110, B01101, B11111, B10000, B00000, B00000, B00000, B00000,
        // Ground
        B11111, B11111, B11111, B11111, B11111, B11111, B11111, B11111,
        // Ground right
        B00011, B00011, B00011, B00011, B00011, B00011, B00011, B00011,
        // Ground left
        B11000, B11000, B11000, B11000, B11000, B11000, B11000, B11000,
    };

    for (int i = 0; i < 7; ++i) {
        lcd.createChar(i + 1, &graphics[i * 8]);
    }

    for (int i = 0; i < TERRAIN_WIDTH; ++i) {
        terrainUpper[i] = SPRITE_TERRAIN_EMPTY;
        terrainLower[i] = SPRITE_TERRAIN_EMPTY;
    }
}

void advanceTerrain(char* terrain, byte newTerrain){
    for (int i = 0; i < TERRAIN_WIDTH; ++i) {
        char current = terrain[i];
        char next = (i == TERRAIN_WIDTH-1) ? newTerrain : terrain[i+1];
        switch (current){
            case SPRITE_TERRAIN_EMPTY:
                terrain[i] = (next == SPRITE_TERRAIN_SOLID) ? SPRITE_TERRAIN_SOLID_RIGHT : SPRITE_TERRAIN_EMPTY;
                break;
            case SPRITE_TERRAIN_SOLID:
                terrain[i] = (next == SPRITE_TERRAIN_EMPTY) ? SPRITE_TERRAIN_SOLID_LEFT : SPRITE_TERRAIN_SOLID;
                break;
            case SPRITE_TERRAIN_SOLID_RIGHT:
                terrain[i] = SPRITE_TERRAIN_SOLID;
                break;
            case SPRITE_TERRAIN_SOLID_LEFT:
                terrain[i] = SPRITE_TERRAIN_EMPTY;
                break;
        }
    }
}

bool drawHero(byte position, char* terrainUpper, char* terrainLower, unsigned int score) {
    bool collide = false;
    char upperSave = terrainUpper[HERO_HORIZONTAL_POSITION];
    char lowerSave = terrainLower[HERO_HORIZONTAL_POSITION];
    byte upper, lower;

    switch (position) {
        case HERO_POSITION_OFF:
            upper = lower = SPRITE_TERRAIN_EMPTY;
            break;
        case HERO_POSITION_RUN_LOWER_1:
            upper = SPRITE_TERRAIN_EMPTY;
            lower = SPRITE_RUN1;
            break;
        case HERO_POSITION_RUN_LOWER_2:
            upper = SPRITE_TERRAIN_EMPTY;
            lower = SPRITE_RUN2;
            break;
        case HERO_POSITION_JUMP_1:
        case HERO_POSITION_JUMP_8:
            upper = SPRITE_TERRAIN_EMPTY;
            lower = SPRITE_JUMP;
            break;
        case HERO_POSITION_JUMP_2:
        case HERO_POSITION_JUMP_7:
            upper = SPRITE_JUMP_UPPER;
            lower = SPRITE_JUMP_LOWER;
            break;
        case HERO_POSITION_JUMP_3:
        case HERO_POSITION_JUMP_4:
        case HERO_POSITION_JUMP_5:
        case HERO_POSITION_JUMP_6:
            upper = SPRITE_JUMP;
            lower = SPRITE_TERRAIN_EMPTY;
            break;
        case HERO_POSITION_RUN_UPPER_1:
            upper = SPRITE_RUN1;
            lower = SPRITE_TERRAIN_EMPTY;
            break;
        case HERO_POSITION_RUN_UPPER_2:
            upper = SPRITE_RUN2;
            lower = SPRITE_TERRAIN_EMPTY;
            break;
    }

    if (upper != ' ') {
        terrainUpper[HERO_HORIZONTAL_POSITION] = upper;
        collide = (upperSave == SPRITE_TERRAIN_EMPTY) ? false : true;
    }

    if (lower != ' ') {
        terrainLower[HERO_HORIZONTAL_POSITION] = lower;
        collide |= (lowerSave == SPRITE_TERRAIN_EMPTY) ? false : true;
    }

    byte digits = (score > 9999) ? 5 : (score > 999) ? 4 : (score > 99) ? 3 : (score > 9) ? 2 : 1;

    terrainUpper[TERRAIN_WIDTH] = '\0';
    terrainLower[TERRAIN_WIDTH] = '\0';
    char temp = terrainUpper[16-digits];
    terrainUpper[16-digits] = '\0';
    lcd.setCursor(0,0);
    lcd.print(terrainUpper);
    terrainUpper[16-digits] = temp;
    lcd.setCursor(0,1);
    lcd.print(terrainLower);
    lcd.setCursor(16 - digits,0);
    lcd.print(score);
    terrainUpper[HERO_HORIZONTAL_POSITION] = upperSave;
    terrainLower[HERO_HORIZONTAL_POSITION] = lowerSave;

    return collide;
}

void buttonPush() {
    buttonPushed = true;
}

void setup(){
    pinMode(PIN_READWRITE, OUTPUT);
    digitalWrite(PIN_READWRITE, LOW);
    pinMode(PIN_AUTOPLAY, INPUT_PULLUP);
    lcd.begin(16,2);
    initializeGraphics();
    attachInterrupt(digitalPinToInterrupt(PIN_BUTTON), buttonPush, FALLING);
}

void loop(){
    static byte heroPosition = HERO_POSITION_RUN_LOWER_1;
    static byte heroNextPosition = HERO_POSITION_RUN_LOWER_2;
    static bool paused = true;
    static unsigned int score = 0;

    if (!paused) {
        delay(50);
        advanceTerrain(terrainUpper, TERRAIN_EMPTY);
        advanceTerrain(terrainLower, random(10) == 0 ? SPRITE_TERRAIN_SOLID : SPRITE_TERRAIN_EMPTY);
    }
    else {
        delay(250);
    }

    if (buttonPushed) {
        buttonPushed = false;
        paused = false;
        if (heroPosition < HERO_POSITION_JUMP_1 || heroPosition > HERO_POSITION_JUMP_8) {
            heroNextPosition = HERO_POSITION_JUMP_1;
        }
    }

    if (drawHero(heroPosition, terrainUpper, terrainLower, score)) {
        delay(2000);
        paused = true;
        heroNextPosition = HERO_POSITION_RUN_LOWER_1;
        heroPosition = HERO_POSITION_RUN_LOWER_1;
        for (int i = 0; i < TERRAIN_WIDTH; ++i) {
            terrainUpper[i] = SPRITE_TERRAIN_EMPTY;
            terrainLower[i] = SPRITE_TERRAIN_EMPTY;
        }
        return;
    }

    heroPosition = heroNextPosition;

    switch (heroPosition) {
        case HERO_POSITION_RUN_LOWER_1:
            heroNextPosition = HERO_POSITION_RUN_LOWER_2;
            break;
        case HERO_POSITION_RUN_LOWER_2:
            heroNextPosition = HERO_POSITION_RUN_LOWER_1;
            break;
        case HERO_POSITION_RUN_UPPER_1:
            heroNextPosition = HERO_POSITION_RUN_UPPER_2;
            break;
        case HERO_POSITION_RUN_UPPER_2:
            heroNextPosition = HERO_POSITION_RUN_UPPER_1;
            break;
        default:
            ++heroNextPosition;
    }

    if (!paused) {
        ++score;
    }
}


##Acknowledgments
This project was created as part of an assignment for the Arduino course. Special thanks to our instructor for the guidance and support.

##References
Arduino Documentation: https://www.arduino.cc/reference/en/
LiquidCrystal Library: https://www.arduino.cc/en/Reference/LiquidCrystal
