---
title: 「Keyboard Moe」从零开始自制键盘（二）：蓝牙连接与原型键盘矩阵
date: 2020-02-21 10:03:37
tags: 键盘
---

## 前情回顾

- [「Keyboard Moe」从零开始自制键盘（一）：硬件设计](https://coderemixer.com/2020/02/16/moe-keyboard-log-1/)

## 开发板

由于之前设计的主控是 ESP-WROOM-32，而且自动刷机口用的 CP210x 芯片。理所当然，开发板就会买 [ESP32-DevKitC ESP-WROOM-32](http://akizukidenshi.com/catalog/g/gM-11819/)。最近日本疫情也是越来越重了，我就从秋月电子通商网站上买了一片。说实话这网站的硬件也不是特别全，如果最后配 BOM 可能还是要亲自去跑秋叶原。ESP32 自己的官方库不是特别好用，好在已经有兼容的 [Arduino](https://www.arduino.cc/) 库。[PlatformIO](https://platformio.org/) 对 ESP32 开发板也做了支持，所以一键就能建出基本的工程结构。

## 蓝牙连接

我对蓝牙一窍不通，就上网参考一下。我上网看了看找到一个 [Gist 样例](https://gist.github.com/sabas1080/93115fb66e09c9b40e5857a19f3e7787)。跑了一下很容易跑通了。比较奇怪的是，一引入蓝牙的协议栈，Flash 就吃掉 73% 了，其它部分程序可能要省着用了。

```
Advanced Memory Usage is available via "PlatformIO Home > Project Inspect"
RAM:   [=         ]  11.0% (used 35960 bytes from 327680 bytes)
Flash: [=======   ]  73.6% (used 964611 bytes from 1310720 bytes)
```

但我不能直接抄这代码，首先这东西是 GNU LGPL 协议的，实在是太污染了。另一个问题是，这个 C++ 代码写得非常不 C++，而且在协议栈上只实现了键盘相关的协议，而我需要同时用 HID 中键盘和鼠标两者的协议。所以我又参考了 ESP32 的另一个[实现](https://github.com/asterics/esp32_mouse_keyboard)，最后还是选择自己重写。

```cpp
// status.h
#ifndef STATUS_H_
#define STATUS_H_

bool bluetoothConnected = false;

#endif // STATUS_H_
```

```cpp
// BluetoothService.h
#ifndef BLUETOOTH_SERVICE_H_
#define BLUETOOTH_SERVICE_H_

#include <Arduino.h>
#include <BLEDevice.h>
#include <BLEHIDDevice.h>
#include <BLE2902.h>
#include <status.h>

static const uint8_t HID_REPORT_MAP[] = {
  0x05, 0x01,  // Usage Page (Generic Desktop)
  0x09, 0x02,  // Usage (Mouse)
  0xA1, 0x01,  // Collection (Application)
  0x85, 0x01,  // Report Id (1)
  0x09, 0x01,  //   Usage (Pointer)
  0xA1, 0x00,  //   Collection (Physical)
  0x05, 0x09,  //     Usage Page (Buttons)
  0x19, 0x01,  //     Usage Minimum (01) - Button 1
  0x29, 0x03,  //     Usage Maximum (03) - Button 3
  0x15, 0x00,  //     Logical Minimum (0)
  0x25, 0x01,  //     Logical Maximum (1)
  0x75, 0x01,  //     Report Size (1)
  0x95, 0x03,  //     Report Count (3)
  0x81, 0x02,  //     Input (Data, Variable, Absolute) - Button states
  0x75, 0x05,  //     Report Size (5)
  0x95, 0x01,  //     Report Count (1)
  0x81, 0x01,  //     Input (Constant) - Padding or Reserved bits
  0x05, 0x01,  //     Usage Page (Generic Desktop)
  0x09, 0x30,  //     Usage (X)
  0x09, 0x31,  //     Usage (Y)
  0x09, 0x38,  //     Usage (Wheel)
  0x15, 0x81,  //     Logical Minimum (-127)
  0x25, 0x7F,  //     Logical Maximum (127)
  0x75, 0x08,  //     Report Size (8)
  0x95, 0x03,  //     Report Count (3)
  0x81, 0x06,  //     Input (Data, Variable, Relative) - X & Y coordinate
  0xC0,        //   End Collection
  0xC0,        // End Collection

  0x05, 0x01,  // Usage Pg (Generic Desktop)
  0x09, 0x06,  // Usage (Keyboard)
  0xA1, 0x01,  // Collection: (Application)
  0x85, 0x02,  // Report Id (2)
  //
  0x05, 0x07,  //   Usage Pg (Key Codes)
  0x19, 0xE0,  //   Usage Min (224)
  0x29, 0xE7,  //   Usage Max (231)
  0x15, 0x00,  //   Log Min (0)
  0x25, 0x01,  //   Log Max (1)
  //
  //   Modifier byte
  0x75, 0x01,  //   Report Size (1)
  0x95, 0x08,  //   Report Count (8)
  0x81, 0x02,  //   Input: (Data, Variable, Absolute)
  //
  //   Reserved byte
  0x95, 0x01,  //   Report Count (1)
  0x75, 0x08,  //   Report Size (8)
  0x81, 0x01,  //   Input: (Constant)
  //
  //   LED report
  0x95, 0x05,  //   Report Count (5)
  0x75, 0x01,  //   Report Size (1)
  0x05, 0x08,  //   Usage Pg (LEDs)
  0x19, 0x01,  //   Usage Min (1)
  0x29, 0x05,  //   Usage Max (5)
  0x91, 0x02,  //   Output: (Data, Variable, Absolute)
  //
  //   LED report padding
  0x95, 0x01,  //   Report Count (1)
  0x75, 0x03,  //   Report Size (3)
  0x91, 0x01,  //   Output: (Constant)
  //
  //   Key arrays (6 bytes)
  0x95, 0x06,  //   Report Count (6)
  0x75, 0x08,  //   Report Size (8)
  0x15, 0x00,  //   Log Min (0)
  0x25, 0x65,  //   Log Max (101)
  0x05, 0x07,  //   Usage Pg (Key Codes)
  0x19, 0x00,  //   Usage Min (0)
  0x29, 0x65,  //   Usage Max (101)
  0x81, 0x00,  //   Input: (Data, Array)
  //
  0xC0,        // End Collection
  //
  0x05, 0x0C,   // Usage Pg (Consumer Devices)
  0x09, 0x01,   // Usage (Consumer Control)
  0xA1, 0x01,   // Collection (Application)
  0x85, 0x03,   // Report Id (3)
  0x09, 0x02,   //   Usage (Numeric Key Pad)
  0xA1, 0x02,   //   Collection (Logical)
  0x05, 0x09,   //     Usage Pg (Button)
  0x19, 0x01,   //     Usage Min (Button 1)
  0x29, 0x0A,   //     Usage Max (Button 10)
  0x15, 0x01,   //     Logical Min (1)
  0x25, 0x0A,   //     Logical Max (10)
  0x75, 0x04,   //     Report Size (4)
  0x95, 0x01,   //     Report Count (1)
  0x81, 0x00,   //     Input (Data, Ary, Abs)
  0xC0,         //   End Collection
  0x05, 0x0C,   //   Usage Pg (Consumer Devices)
  0x09, 0x86,   //   Usage (Channel)
  0x15, 0xFF,   //   Logical Min (-1)
  0x25, 0x01,   //   Logical Max (1)
  0x75, 0x02,   //   Report Size (2)
  0x95, 0x01,   //   Report Count (1)
  0x81, 0x46,   //   Input (Data, Var, Rel, Null)
  0x09, 0xE9,   //   Usage (Volume Up)
  0x09, 0xEA,   //   Usage (Volume Down)
  0x15, 0x00,   //   Logical Min (0)
  0x75, 0x01,   //   Report Size (1)
  0x95, 0x02,   //   Report Count (2)
  0x81, 0x02,   //   Input (Data, Var, Abs)
  0x09, 0xE2,   //   Usage (Mute)
  0x09, 0x30,   //   Usage (Power)
  0x09, 0x83,   //   Usage (Recall Last)
  0x09, 0x81,   //   Usage (Assign Selection)
  0x09, 0xB0,   //   Usage (Play)
  0x09, 0xB1,   //   Usage (Pause)
  0x09, 0xB2,   //   Usage (Record)
  0x09, 0xB3,   //   Usage (Fast Forward)
  0x09, 0xB4,   //   Usage (Rewind)
  0x09, 0xB5,   //   Usage (Scan Next)
  0x09, 0xB6,   //   Usage (Scan Prev)
  0x09, 0xB7,   //   Usage (Stop)
  0x15, 0x01,   //   Logical Min (1)
  0x25, 0x0C,   //   Logical Max (12)
  0x75, 0x04,   //   Report Size (4)
  0x95, 0x01,   //   Report Count (1)
  0x81, 0x00,   //   Input (Data, Ary, Abs)
  0x09, 0x80,   //   Usage (Selection)
  0xA1, 0x02,   //   Collection (Logical)
  0x05, 0x09,   //     Usage Pg (Button)
  0x19, 0x01,   //     Usage Min (Button 1)
  0x29, 0x03,   //     Usage Max (Button 3)
  0x15, 0x01,   //     Logical Min (1)
  0x25, 0x03,   //     Logical Max (3)
  0x75, 0x02,   //     Report Size (2)
  0x81, 0x00,   //     Input (Data, Ary, Abs)
  0xC0,           //   End Collection
  0x81, 0x03,   //   Input (Const, Var, Abs)
  0xC0,            // End Collection
  0x06, 0xFF, 0xFF, // Usage Page(Vendor defined)
  0x09, 0xA5,       // Usage(Vendor Defined)
  0xA1, 0x01,       // Collection(Application)
  0x85, 0x04,   // Report Id (4)
  0x09, 0xA6,   // Usage(Vendor defined)
  0x09, 0xA9,   // Usage(Vendor defined)
  0x75, 0x08,   // Report Size
  0x95, 0x7F,   // Report Count = 127 Btyes
  0x91, 0x02,   // Output(Data, Variable, Absolute)
  0xC0,         // End Collection
};

class BluetoothService {
private:
  BLEHIDDevice* hid;
  BLEAdvertising *pAdvertising;

public:
  BLECharacteristic* input;
  BLECharacteristic* output;

  BluetoothService();
  void startAdvertising();
  void stopAdvertising();
};

class BluetoothCallbacks: public BLEServerCallbacks {
  BluetoothService* service;
  void onConnect(BLEServer* _);
  void onDisconnect(BLEServer* pServer);

public:
  BluetoothCallbacks(BluetoothService* service);
};

class BluetoothOutputCallbacks : public BLECharacteristicCallbacks {
  void onWrite(BLECharacteristic* me);
};

#endif // BLUETOOTH_SERVICE_H_
```

```cpp
// BluetoothService.cpp
#ifndef BLUETOOTH_SERVICE_CPP_
#define BLUETOOTH_SERVICE_CPP_

#include "BluetoothService.h"

BluetoothCallbacks::BluetoothCallbacks(BluetoothService* service) {
  this->service = service;
}

void BluetoothCallbacks::onConnect(BLEServer* _) {
  BLE2902* desc = (BLE2902*)service->input->getDescriptorByUUID(BLEUUID((uint16_t)0x2902));
  desc->setNotifications(true);
  bluetoothConnected = true;
  Serial.println("Bluetooth Connected.");
}

void BluetoothCallbacks::onDisconnect(BLEServer* pServer){
  BLE2902* desc = (BLE2902*)service->input->getDescriptorByUUID(BLEUUID((uint16_t)0x2902));
  desc->setNotifications(false);
  bluetoothConnected = false;
  Serial.println("Bluetooth Disconnected.");
}

void BluetoothOutputCallbacks::onWrite(BLECharacteristic* me){
  uint8_t* value = (uint8_t*)(me->getValue().c_str());
  Serial.printf("special keys: %d\n", *value);
}

BluetoothService::BluetoothService() {
  BLEDevice::init("Keyboard Moe");
  BLEServer *pServer = BLEDevice::createServer();
  pServer->setCallbacks(new BluetoothCallbacks(this));

  hid = new BLEHIDDevice(pServer);
  input = hid->inputReport(1);
  output = hid->outputReport(1);

  output->setCallbacks(new BluetoothOutputCallbacks());

  std::string manufacturer = "CodeRemixer";
  hid->manufacturer()->setValue(manufacturer);

  hid->pnp(0x02, 0xe502, 0xa111, 0x0210);
  BLESecurity *pSecurity = new BLESecurity();
  pSecurity->setAuthenticationMode(ESP_LE_AUTH_BOND);

  // https://www.bluetooth.com/wp-content/uploads/Sitecore-Media-Library/Gatt/Xml/Characteristics/org.bluetooth.characteristic.hid_information.xml
  hid->hidInfo(0x00, 0x02);

  hid->reportMap((uint8_t*)HID_REPORT_MAP, sizeof(HID_REPORT_MAP));
  hid->startServices();

  pAdvertising = pServer->getAdvertising();
  pAdvertising->setAppearance(HID_KEYBOARD);
  pAdvertising->addServiceUUID(hid->hidService()->getUUID());
}

void BluetoothService::startAdvertising() {
  pAdvertising->start();
  Serial.println("Advertising Started.");
}

void BluetoothService::stopAdvertising() {
  pAdvertising->stop();
  Serial.println("Advertising Stopped.");
}

#endif // BLUETOOTH_SERVICE_CPP_
```

写完发现一个奇怪的事情就是蓝牙一连接会把循环阻塞，于是直接用 FreeRTOS 来执行蓝牙代码。

```cpp
// main.cpp
#include <Arduino.h>
#include <status.h>

#include "BluetoothService.cpp"

BluetoothService* bluetoothService;

void bluetoothServiceProcess(void*) {
  bluetoothService = new BluetoothService();
  bluetoothService->startAdvertising();
  delay(portMAX_DELAY);
}

void setup() {
  Serial.begin(115200);
  Serial.println("Moe Keyboard is Starting Up...");

  xTaskCreate(bluetoothServiceProcess, "bluetooth_server", 20000, NULL, 5, NULL);
}
```

## 4x3 键盘矩阵

接下来就是需要调键盘按键了，一个节省 pin 的典型设计就是键盘矩阵。为了解耦，这里使用的是键盘矩阵套件，等测试通过后才会换成正式的键盘矩阵。我买的是 [4x3 键盘矩阵套件](http://akizukidenshi.com/catalog/g/gK-12229/)。

![Keyboard Matrix](/static/keyboard-matrix-kit-1.jpg)

买回来花了两个小时自己焊接了起来。比较奇怪的是二极管正负极标注和脚的长度不匹配，吓得我赶紧用万用表出来重新测了一下。

![Keyboard Matrix](/static/keyboard-matrix-kit-2.jpg)

焊得非常吃力，焊完反应过来，把焊台温度调得太低了...

![Keyboard Matrix](/static/keyboard-matrix-kit-3.jpg)
![Keyboard Matrix](/static/keyboard-matrix-kit-4.png)

## 键盘矩阵扫描

这个键盘矩阵套件自带上拉电阻，这和最后我的矩阵有一定出入（因为我还要实现灯阵），但程序还是差不多的。

一开始我选的是 34 35 32 33 25 26 27 这几 pin 来做。结果，写完后发现前两排按键死活不能用。一开始以为自己焊接短路了，用万用表测了半天也没找到短路点。以防万一又重新焊了一遍结果问题依旧。

最后查了一下发现 GPIO 34 35 36 39 是[无法作为 output 模式运行](https://randomnerdtutorials.com/esp32-pinout-reference-gpios/)的，最后把需要 output 的更换到了 19 18 和 5 上。

```cpp
// gpio.h

#ifndef GPIO_H_
#define GPIO_H_

#include "KeyCode.cpp"

static const uint8_t COL_NUM = 3;
static const uint8_t ROW_NUM = 4;

static const uint8_t COL_GPIO[] = {
  19, 18, 5
};

static const uint8_t ROW_GPIO[] = {
  33, 25, 26, 27
};

static const KeyCode* KEYCODE_MAP[] = {
  new KeyCode(NORMAL, 0x55), // *
  new KeyCode(NORMAL, 0x5f), // 7
  new KeyCode(NORMAL, 0x5c), // 4
  new KeyCode(NORMAL, 0x59), // 1
  new KeyCode(NORMAL, 0x62), // 0
  new KeyCode(NORMAL, 0x80), // 8
  new KeyCode(NORMAL, 0x5d), // 5
  new KeyCode(NORMAL, 0x5a), // 2
  new KeyCode(NORMAL, 0xcc), // #
  new KeyCode(NORMAL, 0x61), // 9
  new KeyCode(NORMAL, 0x5e), // 6
  new KeyCode(NORMAL, 0x5b), // 3
};

static bool keyStatus[COL_NUM][ROW_NUM] = { false };
static bool keyStatusSaved[COL_NUM][ROW_NUM] = { false };

#endif // GPIO_H_
```

```cpp
// KeyCode.h
#ifndef KEYCODE_H_
#define KEYCODE_H_

enum KeyType {
  MULTIMEDIA,
  MODIFIER,
  NORMAL,
};

class KeyCode {
public:
  KeyType type;
  uint8_t code;

  KeyCode(KeyType type, uint8_t code);
};

#endif // KEYCODE_H_
```

```cpp
// KeyCode.cpp
#ifndef KEYCODE_CPP_
#define KEYCODE_CPP_

#include "KeyCode.h"

KeyCode::KeyCode(KeyType type, uint8_t code) {
  this->type = type;
  this->code = code;
}

#endif // KEYCODE_CPP_
```

```cpp
#include <Arduino.h>
#include <status.h>

#include "gpio.h"
#include "BluetoothService.cpp"

BluetoothService* bluetoothService;

void bluetoothServiceProcess(void*) {
  bluetoothService = new BluetoothService();
  bluetoothService->startAdvertising();
  delay(portMAX_DELAY);
}

void setup() {
  Serial.begin(115200);
  Serial.println("Moe Keyboard is Starting Up...");

  for (uint8_t i = 0; i < COL_NUM; i++) {
    pinMode(COL_GPIO[i], OUTPUT);
  }

  for (uint8_t i = 0; i < ROW_NUM; i++) {
    pinMode(ROW_GPIO[i], INPUT);
  }

  xTaskCreate(bluetoothServiceProcess, "bluetooth_server", 20000, NULL, 5, NULL);
}

void loop() {
  // Matrix Scan
  for (uint8_t i = 0; i < COL_NUM; i++) {
    for (uint8_t j = 0; j < COL_NUM; j++)
      i == j ? digitalWrite(COL_GPIO[j], LOW) : digitalWrite(COL_GPIO[j], HIGH);
    for (uint8_t j = 0; j < ROW_NUM; j++) {
      keyStatus[i][j] = !digitalRead(ROW_GPIO[j]);
    }
  }

  // Output
  for (uint8_t i = 0; i < COL_NUM; i++) {
    for (uint8_t j = 0; j < ROW_NUM; j++) {
      if (keyStatus[i][j] == true && keyStatusSaved[i][j] == false) {
        Serial.printf("Key Down: (col: %d, row: %d, code: %d)\n", i, j, KEYCODE_MAP[i * ROW_NUM + j]->code);
      } else if  (keyStatus[i][j] == false && keyStatusSaved[i][j] == true) {
        Serial.printf("Key Up: (col: %d, row: %d, code: %d)\n", i, j, KEYCODE_MAP[i * ROW_NUM + j]->code);
      }
      keyStatusSaved[i][j] = keyStatus[i][j];
    }
  }

  delay(1);
}
```

运行结果：

![Matrix Scan](/static/matrix-scan.png)

完美。
