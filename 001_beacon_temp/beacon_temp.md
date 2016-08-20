# [Recipe 001] iBeaconで温度を飛ばしiPhoneアプリで受信する

# 本レシピについて

本レシピは、FaBo Arduinoで温度センサーの値をiBeaconで飛ばし、Swiftアプリで受信して表示するためのレシピです。

## レシピの基礎マテリアル

* Arduino
* [[FaBo Docs]#501 Shield Outin Arduino](http://docs.fabo.io/fabo/arduino/outin/501_shield_outin_arduino.html)
* [[FaBo Docs]#108 FaBo Temperature](http://docs.fabo.io/fabo/arduino/brick_analog/108_brick_analog_temperature.html)
* [[FaBo Docs]#307 FaBo Nordic BLE](http://docs.fabo.io/fabo/arduino/brick_serial/307_brick_serial_nordic.html)
* [[Swift Docs]iBeaconをモニタリングする](http://docs.fabo.io/swift/corelocation/003_ibeacon_monitaring.html)

## 本レシピで使用するハードウェア

* [Arduino](https://www.arduino.cc/)
* [#501 Shield Outin Arduino](http://www.fabo.io/501.html)
* [#108 FaBo Temperature](http://www.fabo.io/108.html)
* [#307 FaBo Nordic BLE](http://www.fabo.io/307.html])
* iPhone

# [手順1] FaBoで温度を取得

## Arduinoのインストール

Arduinoのページにアクセスし、Arduinoをインストールする。

https://www.arduino.cc/en/Main/Software

![](./img/beacon001.png)

![](./img/beacon003.png)

## Arduinoの実行

Arduinoのアイコンをクリックし、Arduinoを実行します。

![](./img/beacon002.png)

![](./img/beacon004.png)

## LEDを点灯

Arduinoのメニューから[ファイル]-[スケッチの例]-[01.Basics]-[Blink]を選択します。

![](./img/beacon005.png)

![](./img/beacon006.png)

```c
/*
  Blink
  Turns on an LED on for one second, then off for one second, repeatedly.

  Most Arduinos have an on-board LED you can control. On the Uno and
  Leonardo, it is attached to digital pin 13. If you're unsure what
  pin the on-board LED is connected to on your Arduino model, check
  the documentation at http://www.arduino.cc

  This example code is in the public domain.

  modified 8 May 2014
  by Scott Fitzgerald
 */


// the setup function runs once when you press reset or power the board
void setup() {
  // initialize digital pin 13 as an output.
  pinMode(13, OUTPUT);
}

// the loop function runs over and over again forever
void loop() {
  digitalWrite(13, HIGH);   // turn the LED on (HIGH is the voltage level)
  delay(1000);              // wait for a second
  digitalWrite(13, LOW);    // turn the LED off by making the voltage LOW
  delay(1000);              // wait for a second
}
```

Arduinoに実行ファイルを転送するために、[ツール]-[ボード:*]-[Arduino/Genuino Uno]を選びます。
Arduino/Geduino Uno以外のボードを使用する場合は、そのボード名を選択します。

![](./img/beacon008.png)

また、USBで転送するためにシリアルポートを設定します。Macの場合は、/dev/cu.usbmodem**** と表記されているポートを選び、Windowsの場合は、ポート*を選びます。


![](./img/beacon009.png)

![](./img/beacon010.png)

# [手順2] 温度を測る

## 温度を測るコードを作成する

![](./img/beacon007.jpg)


```c
//
// FaBo Brick Sample
//
// #108 Temperature Brick
//
#define tempPin A0 // 温度センサーピン

int tempValue = 0; // 温度取得用

void setup() {
  // 温度センサーピンを入力用に設定
  pinMode(tempPin, INPUT); 

  // シリアル開始 転送レート：9600bps
  Serial.begin(9600);
}

void loop() {
  // センサーより値を取得(0〜1023)
  tempValue = analogRead(tempPin);

  // 取得した値を電圧に変換 (0〜5000mV)
  tempValue = map(tempValue, 0, 1023, 0, 5000);

  // 変換した電圧を300〜1600の値に変換後、温度に変換 (−30〜100度)
  tempValue = map(tempValue, 300, 1600, -30, 100);

  // 算出した温度を出力
  Serial.println(tempValue);

  delay(100);
}
```

Buildし、実機転送します。

![](./img/beacon011.png)

Arduinoのメニューから[ツール]-[シリアルプロッタ]を選択します。

![](./img/beacon012.png)

温度の変化をチェックします。

![](./img/beacon013.png)

# [手順3] BLE BrickでiBeaconをアドバタイジング

![](./img/beacon014.png)

![](./img/beacon015.png)

![](./img/beacon016.png)


```c
//
// FaBo BLE Brick
//
// brick_serial_ble
//
#include <SoftwareSerial.h>
#include <fabo-nordic.h>

SoftwareSerial serial(12, 13);
FaboBLE faboBLE(serial);

// Beaconデータ作成
void makeBeaconData(byte *uuid, int16_t major, int16_t minor, int8_t pow) {
  byte data[30] = {0};
  byte header[] = {0x02, 0x01, 0x06, 0x1A, 0xFF, 0x4C, 0x00, 0x02, 0x15};
  memcpy(&data[0], &header[0], 9);
  memcpy(&data[9], uuid, 16);
  int16_t swapped_major = (major>>8) | (major<<8);
  memcpy(&data[25], &swapped_major, 2);
  int16_t swapped_minor = (minor>>8) | (minor<<8);
  memcpy(&data[27], &swapped_minor, 2);
  memcpy(&data[29], &pow, 1);
  // Advデータ設定
  faboBLE.setAdvData(data, 30);
}

// 初期化完了イベント
void onReady(FaboBLE::VersionInfo ver, int8_t error)
{
  Serial.println(F("\n*onReady"));
  Serial.print(F("CompanyID: "));
  Serial.println(ver.companyID, HEX);
  Serial.print(F("FirmwareID: "));
  Serial.println(ver.firmwareID, HEX);
  Serial.print(F("LMPVersion: "));
  Serial.println(ver.lmp);
  Serial.print(F("Error: "));
  Serial.println(error, HEX);
  // Beaconデータ作成
  byte uuid[] = {0x00, 0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08, 0x09, 0x0a, 0x0b, 0x0c, 0x0d, 0x0e, 0x0f};
  makeBeaconData(uuid, 100, 1000, 0xC9);
}

// Advデータ設定イベント
void onSetAdvData(int8_t error)
{
  Serial.println(F("\n*onSetAdvData"));
  Serial.print(F("Error: "));
  Serial.println(error, HEX);
  // アドバタイズ開始
  faboBLE.startAdvertise();
}

void setup()
{
  Serial.begin(9600);

  // Debugモードでログが詳細に表示されます
  faboBLE.setDebug();
  // イベントハンドラ登録
  faboBLE.onReady = onReady;
  faboBLE.onSetAdvData = onSetAdvData;
  // 初期化
  faboBLE.init();
}

void loop()
{
  // BLE内部処理のためloop内で呼び出してください
  faboBLE.tick();
}
```


# [手順4] iBeaconのMinorに温度を乗せてアドバタイジング

![](./img/ble_pre.png)

![](./img/int001.png)

![](./img/int002.png)

![](./img/int003.png)

![](./img/int004.png)


```c
//
// FaBo BLE Brick
//
// brick_serial_ble
//
#include <SoftwareSerial.h>
#include <fabo-nordic.h>

#define USER_ID 1 // ユーザ別に変える
#define tempPin A0 // 温度センサーピン
int tempValue = 0; // 温度取得用
byte UUID[] = {0x48, 0x53, 0x44, 0x42,  0x4c, 0x45,  0x41, 0x44, 0x80, 0xc0, 0x18, 0x00, 0xff, 0xff, 0xff, 0xff};

SoftwareSerial serial(12, 13);
FaboBLE faboBLE(serial);

// Beaconデータ作成
void makeBeaconData(byte *uuid, int16_t major, int16_t minor, int8_t pow) {
  byte data[30] = {0};
  byte header[] = {0x02, 0x01, 0x06, 0x1A, 0xFF, 0x4C, 0x00, 0x02, 0x15};
  memcpy(&data[0], &header[0], 9);
  memcpy(&data[9], uuid, 16);
  int16_t swapped_major = (major>>8) | (major<<8);
  memcpy(&data[25], &swapped_major, 2);
  int16_t swapped_minor = (minor>>8) | (minor<<8);
  memcpy(&data[27], &swapped_minor, 2);
  memcpy(&data[29], &pow, 1);
  // Advデータ設定
  faboBLE.setAdvData(data, 30);
}

// 初期化完了イベント
void onReady(FaboBLE::VersionInfo ver, int8_t error)
{
  Serial.println(F("\n*onReady"));
  Serial.print(F("CompanyID: "));
  Serial.println(ver.companyID, HEX);
  Serial.print(F("FirmwareID: "));
  Serial.println(ver.firmwareID, HEX);
  Serial.print(F("LMPVersion: "));
  Serial.println(ver.lmp);
  Serial.print(F("Error: "));
  Serial.println(error, HEX);
  // Beaconデータ作成
  makeBeaconData(UUID, USER_ID, 0, 0xC9);
}

// Advデータ設定イベント
void onSetAdvData(int8_t error)
{
  Serial.println(F("\n*onSetAdvData"));
  Serial.print(F("Error: "));
  Serial.println(error, HEX);
  // アドバタイズ開始
  faboBLE.startAdvertise();
}

void setup()
{
  Serial.begin(9600);

  // 温度センサーピンを入力用に設定
  pinMode(tempPin, INPUT);
  
  // Debugモードでログが詳細に表示されます
  faboBLE.setDebug();
  // イベントハンドラ登録
  faboBLE.onReady = onReady;
  faboBLE.onSetAdvData = onSetAdvData;
  // 初期化
  faboBLE.init();
}

int i = 0;
void loop()
{

  // 100ms*10=1(s)ごとに温度を計測
  if (i > 10) {
    // センサーより値を取得(0〜1023)
    tempValue = analogRead(tempPin);

    // 取得した値を電圧に変換 (0〜5000mV)
    tempValue = map(tempValue, 0, 1023, 0, 5000);

    // 変換した電圧を300〜1600の値に変換後、温度に変換 (−30〜100度)
    tempValue = map(tempValue, 300, 1600, -30, 100);

    // Beaconデータ作成
    makeBeaconData(UUID, USER_ID, tempValue, 0xC9);
    i = 0;
  }
  i++;
  
  // BLE内部処理のためloop内で呼び出してください
  faboBLE.tick();

  // 100ms delay
  delay(100);
}
```

# Swiftアプリで受信


