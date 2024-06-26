# Yet Another Arduino Wiegand Library

A library to received data from Wiegand RFID Card readers.


## Features

_Support multiple data formats_!
- It can detect the message size format automatically!
- 4, 8, 26, 32, and 34 bits are tested and work fine
- Should work with other formats (Let me know)

_It is event-driven_!
- You don't poll all the time checking if there is a card -- a callback will tell you when there is one.
- The extra `void*` parameter on the callbacks are useful if you are using multiple Wiegand instances in your code.

_It is hardware-agnostic_!
- It's up to you to detect input changes. You may use [External Interruptions](examples/interrupts/interrupts.ino), [Polling](examples/polling/polling.ino), or something else.


# How to use it

```c++
/*
 * Example on how to use the Wiegand reader library with interruptions.
 */

#include <Wiegand.h>

// These are the pins connected to the Wiegand D0 and D1 signals.
// Ensure your board supports external Interruptions on these pins
#define PIN_D0 2
#define PIN_D1 3

// The object that handles the wiegand protocol
Wiegand wiegand;

// Initialize Wiegand reader
void setup() {
  Serial.begin(9600);

  //Install listeners and initialize Wiegand reader
  wiegand.onReceive(receivedData, "Card readed: ");
  wiegand.onReceiveError(receivedDataError, "Card read error: ");
  wiegand.onStateChange(stateChanged, "State changed: ");
  wiegand.begin(Wiegand::LENGTH_ANY, true);

  //initialize pins as INPUT and attaches interruptions
  pinMode(PIN_D0, INPUT);
  pinMode(PIN_D1, INPUT);
  attachInterrupt(digitalPinToInterrupt(PIN_D0), pinStateChanged, CHANGE);
  attachInterrupt(digitalPinToInterrupt(PIN_D1), pinStateChanged, CHANGE);

  //Sends the initial pin state to the Wiegand library
  pinStateChanged();
}

// Every few milliseconds, check for pending messages on the wiegand reader
// This executes with interruptions disabled, since the Wiegand library is not thread-safe
void loop() {
  noInterrupts();
  wiegand.flush();
  interrupts();
  //Sleep a little -- this doesn't have to run very often.
  delay(100);
}

// When any of the pins have changed, update the state of the wiegand library
void pinStateChanged() {
  wiegand.setPin0State(digitalRead(PIN_D0));
  wiegand.setPin1State(digitalRead(PIN_D1));
}

// Notifies when a reader has been connected or disconnected.
// Instead of a message, the seconds parameter can be anything you want -- Whatever you specify on `wiegand.onStateChange()`
void stateChanged(bool plugged, const char* message) {
    Serial.print(message);
    Serial.println(plugged ? "CONNECTED" : "DISCONNECTED");
}

// Notifies when a card was read.
// Instead of a message, the seconds parameter can be anything you want -- Whatever you specify on `wiegand.onReceive()`
void receivedData(uint8_t* data, uint8_t bits, const char* message) {
    Serial.print(message);
    Serial.print(bits);
    Serial.print("bits / ");
    //Print value in HEX
    uint8_t bytes = (bits+7)/8;
    for (int i=0; i<bytes; i++) {
        Serial.print(data[i] >> 4, 16);
        Serial.print(data[i] & 0xF, 16);
    }
    Serial.println();
}

// Notifies when an invalid transmission is detected
void receivedDataError(Wiegand::DataError error, uint8_t* rawData, uint8_t rawBits, const char* message) {
    Serial.print(message);
    Serial.print(Wiegand::DataErrorStr(error));
    Serial.print(" - Raw data: ");
    Serial.print(rawBits);
    Serial.print("bits / ");

    //Print value in HEX
    uint8_t bytes = (rawBits+7)/8;
    for (int i=0; i<bytes; i++) {
        Serial.print(rawData[i] >> 4, 16);
        Serial.print(rawData[i] & 0xF, 16);
    }
    Serial.println();
}
```


# Wiegand Protocol

The Wiegand protocol is very and easy to implement, but is poorly standarlized.


## Data transmission

The data is sent over 2 wires, `Data 0` and `Data 1`. Both wires are set to `HIGH` most of the time.

When a message is being transmitted, bits are sent one at a time:
- If the bit value is `0`, `Data 0` is set to `LOW` for a few microseconds (between 20µs and 100µs).
- If the bit value is `1`, `Data 1` is set to `LOW` for a few microseconds (between 20µs and 100µs).

After a small delay (between 200µs and 20 ms), the next bit is sent – until the message is complete.

Having both pins to `LOW` is an invalid state, and usually means that the card reader is disconnected.


## Message format

This is where things get hairy: Many vendors have defined their own message formats, with different sizes, fields, parity checks and layouts.

These are the most common formats seen in the wild:


### 26 and 34 bits formats

These are the most common formats, typically used on RFID card readers.

- The first and the last bits of the message are used for parity checks.
- The first half of the bits must have EVEN parity.
- The second half of the bits must have ODD parity.


### 4 and 8 bit formats

These formats are sometimes used on keypads.

On the 4-bit format, the digit is encoded in 4 bits, no extra stuff is added.

On the 8-bit format, the digit is encoded in the lower 4 bits, and the higher 4 bits if the "NOT" of the digit.


### 32 bit format

This format might be non-standard, 4 digits are encoded in 32 bits, no parity included.


# This Library

## Hardware integration

Since the hardware changes a lot, this library doesn't assume anything on how to read the data pins.

When the state of a pin has changed, it's up to you to call `Wiegand.setPinState(pin, state)`.

There are examples on how to use it with [Interruptions](examples/interrupts/interrupts.ino) and [Polling](examples/polling/polling.ino) on Arduinos. 

(You probably want to use interruptions)


## Receiving Data

Use `Wiegand.onReceive()` to listen to messages.

The listener receives the databuffer and number of bits.

If the Wiegand is initialized with `decode_messages=true` (the default), any parity/check bits are removed from the payload. (E.g., On a 26-bits message, the decoded payload has only 24-bits)


## Handling error Data

Use `Wiegand.onReceiveError()` to listen to errors.

The callback receive the raw (non-decoded) message.

The error can be any of these:
- `Communication`: The message was received right away after the library started listening, so the first bits are likely lost.
- `SizeTooBig`: The payload is bigger than the internal buffer. The payload will be truncated to `Wiegand::MAX_BITS` 
- `SizeUnexpected`: The library was configured to expect a specific number of bits, but we received a truncated message.
- `DecodeFailed`: The library was initialized with `decode_messages=true` (the default), but doesn't support the message format it received.
- `VerificationFailed`: The library was initialized with `decode_messages=true`, received a message with one of the known formats, but the message failed the parity checks.


## Padding

Wiegand messages have weird amounts of bits, but callbacks receive byte arrays.

Because of this, if the payload on with wiegand data is not multiple of 8 bits, it will be padded with zeros on the most significant bits of the first byte.

This is _probably_ what you want, so that a 4-bit message with `0xf` is encoded as `[0x0f]` instead of `[0xf0]`


## Automatic message size detection

If the message size is specified on `Wiegand.begin(size)`, your listener will be called as soon as the last bit is received, inside the call to `Wiegand.setPinState()`. Easy!

On the other hand, if you are using automatic message size detections (`Wiegand.begin(Wiegand::LENGTH_ANY)`), the library considers a message finished after a few milliseconds without events.

In this case, you must call `Wiegand.flush()` after suficient time (`WIEGAND_TIMEOUT` milliseconds) has elapsed. The easiest way is to call it from your main loop.

__This library is not thread safe__. If you are using interruptions to detect changes in pin state, call `Wiegand.flush()` with interruptions disabled.


## Device detection

This library supports detection of the card reader.

Use `Wiegand.onStateChange()` to listen to plug/unplug events.

To use this feature, you'll need to add a pull-down resistor on both data pins. This will set the input on a invalid state (LOW-LOW) when the reader is unplugged.

