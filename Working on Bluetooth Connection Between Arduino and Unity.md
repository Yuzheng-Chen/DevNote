# What do we want to do? (Target)
We are designing an Electric Muscle Stimulator(EMS) based game with a wearable device. We need to use the device to connect with other devices and we aimed to make it as smaller as possible. Bluetooth technology allows devices to communicate with each other without the need for a central device like a router or access point, which exactly meets our requirements.

# Bluetooth Connection on Certain Devices

## Bluetooth Connection on HC-06 Bluetooth Module

The [HC-06](https://www.sgbotic.com/index.php?dispatch=products.view&product_id=2471) is a class 2 slave Bluetooth module designed for transparent wireless serial communication.

### Wire Connection

There are 4 pins on the HC-06 Bluetooth Module, which are:
- VCC
- GND
- TXD
- RXD

VCC & GND are for **power supply**, and TXD & RXD are for **Transmitting** and **Receiving** data.

The connection is very simple. I'm taking Arduino UNO as an example, but you can also find serial descriptions for most Arduino boards [here](https://www.arduino.cc/reference/en/language/functions/communication/serial/).

![[Arduino Uno - HC-06 Connection.png]]

The only thing we need to pay attention to is the need to **connect TXD on HC-06 Bluetooth Module to RXD on the Arduino board to transmit data to the Arduino board, and connect RXD on HC-06 Bluetooth Module to TXD on the Arduino board for receiving data**.

### Code Example

#### Code in Unity

We used a paid Unity plugin called [Arduino Bluetooth Plugin](https://assetstore.unity.com/packages/tools/input-management/arduino-bluetooth-plugin-98960) for building the Bluetooth connection between devices. The "Demo" scene works very well for the HC-06 connection.

The "Demo" scene code provided a very nice structure for finding Bluetooth devices and sending basic data. The way of data transmitting is only in the following 3 steps:

**Step 1.** Create a variable named `helper`, and assign it to the instance of object `BluetoothHelper`
~~~cs
helper = BluetoothHelper.GetInstance();
~~~

**Step 2.** Make `helper` start listening when device connected
~~~cs
void OnConnected(BluetoothHelper helper){
    isConnecting=false;
    helper.StartListening(); // Start Listening
}
~~~

**Step 3.** Once Steps 1 & 2 have been set up, we can send data whenever we want
~~~cs
helper.SendData(data);
~~~

We just duplicated the code in the "Demo" scene and made some small changes such as setting up the singleton, changing appearance, etc. to fit our project.

#### Code in Arduino

The code for the HC-06 Bluetooth Module is also straightforward. We just checked if the serial is available, then we get data from `Serial.read()` method. For our project, we used **upper case** and **lower case** letters to control the stage of the EMS device.

~~~Arduino
#define Pin1 31
#define Pin2 33
#define Pin3 35
​
char data = 0;                //Variable for storing received data
​
char EMSOn = HIGH;
char EMSOff = LOW;
​
​
void setup()
{
  Serial.begin(9600);
  pinMode(Pin1, OUTPUT);
  pinMode(Pin2, OUTPUT);
  pinMode(Pin3, OUTPUT);
  digitalWrite(Pin1, EMSOff);
  digitalWrite(Pin2, EMSOff);
  digitalWrite(Pin3, EMSOff);
}
void loop()
{
  if (Serial.available() > 0) // Send data only when you receive data:
  {
    data = Serial.read();      //Read the incoming data and store it into variable data
    Serial.print(data);
    Serial.print("\n");
    if (data == 'A')
      digitalWrite(Pin1, EMSOn);
    else if (data == 'a')
      digitalWrite(Pin1, EMSOff);
    else if (data == 'B')
      digitalWrite(Pin2, EMSOn);
    else if (data == 'b')
      digitalWrite(Pin2, EMSOff);
    else if (data == 'C')
      digitalWrite(Pin3, EMSOn);
    else if (data == 'c')
      digitalWrite(Pin3, EMSOff);
  }
}
~~~

## Bluetooth Connection on Adafruit Feather nRF52840 Sense

To make the setup smaller, we found an 'all-in-one' Arduino-compatible + Bluetooth Low Energy with built-in USB plus battery charging called [Adafruit Feather nRF52 Sense](https://learn.adafruit.com/bluefruit-nrf52-feather-learning-guide/ktownsend-assembly).

![[Adafruit Feather nRF52 Sense.png]]

### Code Example

We followed [the official instruction of Adafruit Feather nRF52 Sense](https://learn.adafruit.com/bluefruit-nrf52-feather-learning-guide/ktownsend-assembly) to do the basic setup, and we found an example code for transmitting & receiving data in the official example pack, which called `blueart_cmdmode`. We simply modified the data receiving part and commoned out the "while (!Serial);" code in the setup method to make it work outside of the serial window.

Here is the final version of our Arduino code (with `BluefruitConfig.h` file from `blueart_cmdmode` example):

~~~Arduino
#define Pin1 10
#define Pin2 11
#define Pin3 12

#include <Arduino.h>
#include <SPI.h>
#include "Adafruit_BLE.h"
#include "Adafruit_BluefruitLE_SPI.h"
#include "Adafruit_BluefruitLE_UART.h"

#include "BluefruitConfig.h"

#if SOFTWARE_SERIAL_AVAILABLE
  #include <SoftwareSerial.h>
#endif


SoftwareSerial bluefruitSS = SoftwareSerial(BLUEFRUIT_SWUART_TXD_PIN, BLUEFRUIT_SWUART_RXD_PIN);

Adafruit_BluefruitLE_UART ble(bluefruitSS, BLUEFRUIT_UART_MODE_PIN,
                      BLUEFRUIT_UART_CTS_PIN, BLUEFRUIT_UART_RTS_PIN);

Adafruit_BluefruitLE_SPI ble(BLUEFRUIT_SPI_CS, BLUEFRUIT_SPI_IRQ, BLUEFRUIT_SPI_RST);

void error(const __FlashStringHelper*err) {
  Serial.println(err);
  while (1);
}

/**************************************************************************/
/*!
    @brief  Sets up the HW an the BLE module (this function is called
            automatically on startup)
*/
/**************************************************************************/
void setup(void)
{
  pinMode(Pin1, OUTPUT);
  pinMode(Pin2, OUTPUT);
  pinMode(Pin3, OUTPUT);
  digitalWrite(Pin1, HIGH);
  digitalWrite(Pin2, HIGH);
  digitalWrite(Pin3, HIGH);
  //while (!Serial);  // required for Flora & Micro
  delay(500);

  Serial.begin(115200);
  Serial.println(F("Adafruit Bluefruit Command Mode Example"));
  Serial.println(F("---------------------------------------"));

  /* Initialise the module */
  Serial.print(F("Initialising the Bluefruit LE module: "));

  if ( !ble.begin(VERBOSE_MODE) )
  {
    error(F("Couldn't find Bluefruit, make sure it's in CoMmanD mode & check wiring?"));
  }
  Serial.println( F("OK!") );

  if ( FACTORYRESET_ENABLE )
  {
    /* Perform a factory reset to make sure everything is in a known state */
    Serial.println(F("Performing a factory reset: "));
    if ( ! ble.factoryReset() ){
      error(F("Couldn't factory reset"));
    }
  }

  /* Disable command echo from Bluefruit */
  ble.echo(false);

  Serial.println("Requesting Bluefruit info:");
  /* Print Bluefruit information */
  ble.info();

  Serial.println(F("Please use Adafruit Bluefruit LE app to connect in UART mode"));
  Serial.println(F("Then Enter characters to send to Bluefruit"));
  Serial.println();

  ble.verbose(false);  // debug info is a little annoying after this point!

  /* Wait for connection */
  while (! ble.isConnected()) {
      delay(500);
  }

  // LED Activity command is only supported from 0.6.6
  if ( ble.isVersionAtLeast(MINIMUM_FIRMWARE_VERSION) )
  {
    // Change Mode LED Activity
    Serial.println(F("******************************"));
    Serial.println(F("Change LED activity to " MODE_LED_BEHAVIOUR));
    ble.sendCommandCheckOK("AT+HWModeLED=" MODE_LED_BEHAVIOUR);
    Serial.println(F("******************************"));
  }
}

/**************************************************************************/
/*!
    @brief  Constantly poll for new command or response data
*/
/**************************************************************************/
void loop(void)
{
  // Check for user input
  char inputs[BUFSIZE+1];

  if ( getUserInput(inputs, BUFSIZE) )
  {
    // Send characters to Bluefruit
    Serial.print("[Send] ");
    Serial.println(inputs);

    ble.print("AT+BLEUARTTX=");
    ble.println(inputs);

    // check response stastus
    if (! ble.waitForOK() ) {
      Serial.println(F("Failed to send?"));
    }
  }

  // Check for incoming characters from Bluefruit
  ble.println("AT+BLEUARTRX");
  ble.readline();
  if (strcmp(ble.buffer, "OK") == 0) {
    // no data
    return;
  }
  // Some data was found, its in the buffer
  Serial.print(F("[Recv] ")); Serial.println(ble.buffer);
  String data = ble.buffer;
  Serial.print(data);
 // Pin1
  if (data == "A")
    digitalWrite(Pin1, LOW);
  else if (data == "a")
    digitalWrite(Pin1, HIGH);
  // Pin1 & Pin2
  else if (data == "B") {
    digitalWrite(Pin1, LOW);
    digitalWrite(Pin2, LOW);
  }
  else if (data == "b") {
    digitalWrite(Pin1, HIGH);
    digitalWrite(Pin2, HIGH);
  }
  // Pin2
  else if (data == "C")
    digitalWrite(Pin2, LOW);
  else if (data == "c")
    digitalWrite(Pin2, HIGH);
  // Pin3
  else if (data == "D")
    digitalWrite(Pin3, LOW);
  else if (data == "d")
    digitalWrite(Pin3, HIGH);
  // Turn Off All
  else if (data == "r") {
    digitalWrite(Pin1, HIGH);
    digitalWrite(Pin2, HIGH);
    digitalWrite(Pin3, HIGH);
  }
  ble.waitForOK();
}

/**************************************************************************/
/*!
    @brief  Checks for user input (via the Serial Monitor)
*/
/**************************************************************************/
bool getUserInput(char buffer[], uint8_t maxSize)
{
  // timeout in 100 milliseconds
  TimeoutTimer timeout(100);

  memset(buffer, 0, maxSize);
  while( (!Serial.available()) && !timeout.expired() ) { delay(1); }

  if ( timeout.expired() ) return false;

  delay(2);
  uint8_t count=0;
  do
  {
    count += Serial.readBytes(buffer+count, maxSize);
    delay(2);
  } while( (count < maxSize) && (Serial.available()) );

  return true;
}
~~~

Fortunately, the Unity code in the previous setup is also working on this setup. We don't need to make any modifications on the Unity part.

## Bluetooth Connection on Seeeduino XIAO BLE Sense

To make our setup more concise, we got to [Seeeduino XIAO BLE Sense](https://www.seeedstudio.com/Seeed-XIAO-BLE-Sense-nRF52840-p-5253.html), which is a smaller Bluetooth BLE development board designed for **IoT** and **AI** applications. It features an **onboard antenna**, **6 Dof IMU**, **microphone**, all of which make it an ideal board to run AI using TinyML and TensorFlow Lite.

![[Seeeduino XIAO BLE Sense.png]]

### Code Example

We followed the instructions on [Seeedstudio wiki page](https://wiki.seeedstudio.com) of [Bluetooth Usage on XIAO BLE (Sense)](https://wiki.seeedstudio.com/XIAO-BLE-Sense-Bluetooth-Usage/), and successfully made a built-in LED using an app called "Light Blue" on a smartphone.

**Orginal Example Code:**
~~~Arduino
#include <ArduinoBLE.h>
 
BLEService ledService("19B10000-E8F2-537E-4F6C-D104768A1214"); // Bluetooth® Low Energy LED Service
 
// Bluetooth® Low Energy LED Switch Characteristic - custom 128-bit UUID, read and writable by central
BLEByteCharacteristic switchCharacteristic("19B10001-E8F2-537E-4F6C-D104768A1214", BLERead | BLEWrite);
 
const int ledPin = LED_BUILTIN; // pin to use for the LED
 
void setup() {
  Serial.begin(9600);
  while (!Serial);
 
  // set LED pin to output mode
  pinMode(ledPin, OUTPUT);
 
  // begin initialization
  if (!BLE.begin()) {
    Serial.println("starting Bluetooth® Low Energy module failed!");
 
    while (1);
  }
 
  // set advertised local name and service UUID:
  BLE.setLocalName("LED");
  BLE.setAdvertisedService(ledService);
 
  // add the characteristic to the service
  ledService.addCharacteristic(switchCharacteristic);
 
  // add service
  BLE.addService(ledService);
 
  // set the initial value for the characeristic:
  switchCharacteristic.writeValue(0);
 
  // start advertising
  BLE.advertise();
 
  Serial.println("BLE LED Peripheral");
}
 
void loop() {
  // listen for Bluetooth® Low Energy peripherals to connect:
  BLEDevice central = BLE.central();
 
  // if a central is connected to peripheral:
  if (central) {
    Serial.print("Connected to central: ");
    // print the central's MAC address:
    Serial.println(central.address());
 
    // while the central is still connected to peripheral:
  while (central.connected()) {
        if (switchCharacteristic.written()) {
          if (switchCharacteristic.value()) {   
            Serial.println("LED on");
            digitalWrite(ledPin, LOW); // changed from HIGH to LOW       
          } else {                              
            Serial.println(F("LED off"));
            digitalWrite(ledPin, HIGH); // changed from LOW to HIGH     
          }
        }
      }
 
    // when the central disconnects, print it out:
    Serial.print(F("Disconnected from central: "));
    Serial.println(central.address());
  }
}
~~~
However, it seems that the previous Unity code can not transmit any data to this device. The reason is this example code in Seeeduino is receiving data from a subscribed Bluetooth service.

A Bluetooth device contains a table of data called an Attribute which are Services, Characteristics and Descriptors.
![[illustrates the data hierarchy introduced by GATT.png]]
Services, Characteristics and Descriptors are organized in a hierarchy with Services at the top and Descriptors at the bottom.

With this basic Bluetooth concept, let's go through the example code core steps:

**Step 1.** In this example code, it at first claims a service and a characteristic with their types & UUIDs
~~~Arduino
BLEService ledService("19B10000-E8F2-537E-4F6C-D104768A1214"); // Bluetooth® Low Energy LED Service

BLEByteCharacteristic switchCharacteristic("19B10001-E8F2-537E-4F6C-D104768A1214", BLERead | BLEWrite);
~~~

**Step 2.** Set the advertised service
~~~Arduino
BLE.setAdvertisedService(ledService);
~~~

Step 3. Add characteristic to the service
~~~Arduino
ledService.addCharacteristic(switchCharacteristic);
~~~

Step 4. Start advertising
~~~Arduino
BLE.advertise();
~~~

#### Code in Unity
Let's modify the Unity code.

For setup, in the `OnConnected` method, instead of calling
~~~cs
helper.StartListening();
~~~

We use the following code instead
~~~cs
var service = new BluetoothHelperService("19B10000-E8F2-537E-4F6C-D104768A1214");

characteristic = new BluetoothHelperCharacteristic("19B10001-E8F2-537E-4F6C-D104768A1214");

service.addCharacteristic(characteristic);

helper.Subscribe(service);
~~~

Notice that the UUID in the Unity code is the same as the UUID in the Seeeduino code.

To transmit data, instead of calling
~~~cs
helper.SendData(data);
~~~

We use the following code instead
~~~cs
helper.WriteCharacteristic(characteristic, data);
~~~
The `characteristic` here is the characteristic we created before.

#### Code in Arduino
According to the [API](https://github.com/arduino-libraries/ArduinoBLE/blob/master/docs/api.md), we modified the BLEByteCharacteristic into BLECharCharacteristic to receive string/char message and modified the data receiving part to fit our project. Here is our final code:
~~~Arduino
#define Pin1 1
#define Pin2 2
#define Pin3 3

#include <ArduinoBLE.h>

BLEService ledService("19B10000-E8F2-537E-4F6C-D104768A1214"); // Bluetooth® Low Energy LED Service

// Bluetooth® Low Energy LED Switch Characteristic - custom 128-bit UUID, read and writable by central
BLECharCharacteristic switchCharacteristic("19B10001-E8F2-537E-4F6C-D104768A1214", BLERead | BLEWrite);

const int ledPin = LED_BUILTIN; // pin to use for the LED

void setup() {
  pinMode(Pin1, OUTPUT);
  pinMode(Pin2, OUTPUT);
  pinMode(Pin3, OUTPUT);
  Serial.begin(9600);
  digitalWrite(Pin1, HIGH);
  digitalWrite(Pin2, HIGH);
  digitalWrite(Pin3, HIGH);

  // set LED pin to output mode
  pinMode(ledPin, OUTPUT);

  // begin initialization
  if (!BLE.begin()) {
    Serial.println("starting Bluetooth® Low Energy module failed!");

    while (1);
  }

  // set advertised local name and service UUID:
  BLE.setLocalName("Paizo");
  BLE.setAdvertisedService(ledService);

  // add the characteristic to the service
  ledService.addCharacteristic(switchCharacteristic);

  // add service
  BLE.addService(ledService);

  // set the initial value for the characeristic:
  switchCharacteristic.writeValue('r');

  // start advertising
  BLE.advertise();

  Serial.println("BLE LED Peripheral");
}

void loop() {
  // listen for Bluetooth® Low Energy peripherals to connect:
  BLEDevice central = BLE.central();

  // if a central is connected to peripheral:
  if (central) {
    Serial.print("Connected to central: ");
    // print the central's MAC address:
    Serial.println(central.address());

    // while the central is still connected to peripheral:
    while (central.connected()) {
      // if the remote device wrote to the characteristic,
      // use the value to control the LED:
      if (switchCharacteristic.written()) {
        if (switchCharacteristic.value() == 'r') {
          Serial.println("LED off");
          digitalWrite(ledPin, HIGH);
        } else {
          Serial.println("LED on");
          digitalWrite(ledPin, LOW);
        }
        controlLight(switchCharacteristic.value());
      }
    }

    // when the central disconnects, print it out:
    Serial.print(F("Disconnected from central: "));
    Serial.println(central.address());
  }
}

void controlLight(char data) {
  Serial.print("Received: ");
  // Pin1
  if (data == 'A') {
    digitalWrite(Pin1, LOW);
    Serial.println("A");
  }
  else if (data == 'a') {
    digitalWrite(Pin1, HIGH);
    Serial.println("a");
  }
  // Pin1 & Pin2
  else if (data == 'B') {
    digitalWrite(Pin1, LOW);
    digitalWrite(Pin2, LOW);
    Serial.println("B");
  }
  else if (data == 'b') {
    digitalWrite(Pin1, HIGH);
    digitalWrite(Pin2, HIGH);
    Serial.println("b");
  }
  // Pin2
  else if (data == 'C') {
    digitalWrite(Pin2, LOW);
    Serial.println("C");
  }
  else if (data == 'c') {
    digitalWrite(Pin2, HIGH);
    Serial.println("c");
  }
  // Pin3
  else if (data == 'D') {
    digitalWrite(Pin3, LOW);
    Serial.println("D");
  }
  else if (data == 'd') {
    digitalWrite(Pin3, HIGH);
    Serial.println("d");
  }
  // Turn Off All
  else if (data == 'r') {
    digitalWrite(Pin1, HIGH);
    digitalWrite(Pin2, HIGH);
    digitalWrite(Pin3, HIGH);
    Serial.println("r");
  }
}
~~~

# Conclusion