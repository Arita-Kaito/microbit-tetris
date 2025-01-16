# 概要
micro:bitとPCをBluetoothで接続し、取得したセンシングデータを用いて、JavaScript上のテトリス風ゲームをコントロールします．

# 使用するセンシングデータと役割
1. 加速度センサー
    - x軸
        左右に傾けると、ブロックが左右に移動．
    - y軸 
        奥に傾けると、ブロックが90°回転．
1. 温度センサー
        温度に応じて、ブロックの落下速度を調節．
1. 照度センサー
        周囲の明るさに応じて、ゲーム画面の明るさを調節．
1. (Bボタン)
        Bボタンを押すと、ブロックを落下．

# micro:bit(MakeCode)のプログラム
micro:bitのプログラムは、センシングデータをBletoothで送信するだけですので、非常にシンプルです．

<img width="700" alt="microbit-bluetooth-sensor.hex" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3087800/324c9e72-1447-51de-6d6b-5200907860eb.jpeg">

<details><summary>micro:bitプログラムのJavaScript版はこちら</summary><div>

```micro:bit-makecode.js
bluetooth.onBluetoothConnected(function () {
    basic.showIcon(IconNames.Yes)
})
bluetooth.onBluetoothDisconnected(function () {
    basic.showIcon(IconNames.No)
})
// Bボタンの状態
let bButtonState = 0
let temperature = 0
let lightLevel = 0
let accZ = 0
let accY = 0
let accX = 0
bluetooth.startUartService()
bluetooth.startAccelerometerService()
bluetooth.startTemperatureService()
bluetooth.startLEDService()
basic.forever(function () {
    // 加速度センサーのデータ
    accX = input.acceleration(Dimension.X)
    accY = input.acceleration(Dimension.Y)
    accZ = input.acceleration(Dimension.Z)
    // 光センサーのデータ
    lightLevel = input.lightLevel()
    // 温度センサーのデータ
    temperature = input.temperature()
    // Bボタンの状態
    if (input.buttonIsPressed(Button.B)) {
        // Bボタンが押されたと認識
        bButtonState = 1
    } else {
        // Bボタンが離された状態
        bButtonState = 0
    }
    // センサーのデータを送信
    bluetooth.uartWriteString(`a:${accX},${accY},${accZ}`)
    basic.pause(100)
    bluetooth.uartWriteString(`t:${temperature};l:${lightLevel};b:${bButtonState}`)
    basic.pause(100)
})

```
</div></details>

# micro:bitとPCをBluetoothで接続
通常のWeb Bluetooth APIではセンシングデータが取得できなかったため、ラッパーライブラリ **"BlueJelly"** を使用しました．
3年前にBlueJellyをmicro:bit用に改良していましたので、今回はそちらを使用しました．

<details><summary>BlueJelly.jsのコード</summary><div>

```bluejelly.js
/*
============================================================
BlueJelly.js
============================================================
Web Bluetooth API Wrapper Library

Copyright 2017-2020 JellyWare Inc.
https://monomonotech.jp/

GitHub
https://github.com/electricbaka/bluejelly
This software is released under the MIT License.

Web Bluetooth API
https://webbluetoothcg.github.io/web-bluetooth/
*/

//--------------------------------------------------
//BlueJelly constructor
//--------------------------------------------------
var BlueJelly = function(){
  this.bluetoothDevice = null;
  this.dataCharacteristic = null;
  this.hashUUID ={};
  this.hashUUID_lastConnected;

  //callBack
  this.onScan = function(deviceName){console.log("onScan");};
  this.onConnectGATT = function(uuid){console.log("onConnectGATT");};
  this.onRead = function(data, uuid){console.log("onRead");};
  this.onWrite = function(uuid){console.log("onWrite");};
  this.onStartNotify = function(uuid){console.log("onStartNotify");};
  this.onStopNotify = function(uuid){console.log("onStopNotify");};
  this.onDisconnect = function(){console.log("onDisconnect");};
  this.onClear = function(){console.log("onClear");};
  this.onReset = function(){console.log("onReset");};
  this.onError = function(error){console.log("onError");};
}


//--------------------------------------------------
//setUUID
//--------------------------------------------------
BlueJelly.prototype.setUUID = function(name, serviceUUID, characteristicUUID){
  console.log('Execute : setUUID');
  console.log(this.hashUUID);

  this.hashUUID[name] = {'serviceUUID':serviceUUID, 'characteristicUUID':characteristicUUID};
}


//--------------------------------------------------
//scan
//--------------------------------------------------
BlueJelly.prototype.scan = function(uuid){
  return (this.bluetoothDevice ? Promise.resolve() : this.requestDevice(uuid))
  .catch(error => {
    console.log('Error : ' + error);
    this.onError(error);
  });
}


//--------------------------------------------------
//requestDevice
//--------------------------------------------------
BlueJelly.prototype.requestDevice = function(uuid) {
  console.log('Execute : requestDevice');
  return navigator.bluetooth.requestDevice({
      acceptAllDevices: true,
      optionalServices: [this.hashUUID[uuid].serviceUUID]})
  .then(device => {
    this.bluetoothDevice = device;
    this.bluetoothDevice.addEventListener('gattserverdisconnected', this.onDisconnect);
    this.onScan(this.bluetoothDevice.name);
  });
}


//--------------------------------------------------
//connectGATT
//--------------------------------------------------
BlueJelly.prototype.connectGATT = function(uuid) {
  if(!this.bluetoothDevice)
  {
    var error = "No Bluetooth Device";
    console.log('Error : ' + error);
    this.onError(error);
    return;
  }
  if (this.bluetoothDevice.gatt.connected && this.dataCharacteristic) {
    if(this.hashUUID_lastConnected == uuid)
      return Promise.resolve();
  }
  this.hashUUID_lastConnected = uuid;

  console.log('Execute : connect');
  return this.bluetoothDevice.gatt.connect()
  .then(server => {
    console.log('Execute : getPrimaryService');
    return server.getPrimaryService(this.hashUUID[uuid].serviceUUID);
  })
  .then(service => {
    console.log('Execute : getCharacteristic');
    return service.getCharacteristic(this.hashUUID[uuid].characteristicUUID);
  })
  .then(characteristic => {
    this.dataCharacteristic = characteristic;
    this.dataCharacteristic.addEventListener('characteristicvaluechanged',this.dataChanged(this, uuid));
    this.onConnectGATT(uuid);
  })
  .catch(error => {
      console.log('Error : ' + error);
      this.onError(error);
    });
}


//--------------------------------------------------
//dataChanged
//--------------------------------------------------
BlueJelly.prototype.dataChanged = function(self, uuid) {
  return function(event) {
    self.onRead(event.target.value, uuid);
  }
}


//--------------------------------------------------
//read
//--------------------------------------------------
BlueJelly.prototype.read= function(uuid) {
  return (this.scan(uuid))
  .then( () => {
    return this.connectGATT(uuid);
  })
  .then( () => {
    console.log('Execute : readValue');
    return this.dataCharacteristic.readValue();
  })
  .catch(error => {
    console.log('Error : ' + error);
    this.onError(error);
  });
}


//--------------------------------------------------
//write
//--------------------------------------------------
BlueJelly.prototype.write = function(uuid, array_value) {
  return (this.scan(uuid))
  .then( () => {
    return this.connectGATT(uuid);
  })
  .then( () => {
    console.log('Execute : writeValue');
    data = Uint8Array.from(array_value);
    return this.dataCharacteristic.writeValue(data);
  })
  .then( () => {
    this.onWrite(uuid);
  })
  .catch(error => {
    console.log('Error : ' + error);
    this.onError(error);
  });
}


//--------------------------------------------------
//startNotify
//--------------------------------------------------
BlueJelly.prototype.startNotify = function(uuid) {
  return (this.scan(uuid))
  .then( () => {
    return this.connectGATT(uuid);
  })
  .then( () => {
    console.log('Execute : startNotifications');
    this.dataCharacteristic.startNotifications()
  })
  .then( () => {
    this.onStartNotify(uuid);
  })
  .catch(error => {
    console.log('Error : ' + error);
    this.onError(error);
  });
}


//--------------------------------------------------
//stopNotify
//--------------------------------------------------
BlueJelly.prototype.stopNotify = function(uuid){
  return (this.scan(uuid))
  .then( () => {
    return this.connectGATT(uuid);
  })
  .then( () => {
  console.log('Execute : stopNotifications');
  this.dataCharacteristic.stopNotifications()
})
  .then( () => {
    this.onStopNotify(uuid);
  })
  .catch(error => {
    console.log('Error : ' + error);
    this.onError(error);
  });
}


//--------------------------------------------------
//disconnect
//--------------------------------------------------
BlueJelly.prototype.disconnect= function() {
  if (!this.bluetoothDevice) {
    var error = "No Bluetooth Device";
    console.log('Error : ' + error);
    this.onError(error);
    return;
  }

  if (this.bluetoothDevice.gatt.connected) {
    console.log('Execute : disconnect');
    this.bluetoothDevice.gatt.disconnect();
  } else {
   var error = "Bluetooth Device is already disconnected";
   console.log('Error : ' + error);
   this.onError(error);
   return;
  }
}


//--------------------------------------------------
//clear
//--------------------------------------------------
BlueJelly.prototype.clear= function() {
   console.log('Excute : Clear Device and Characteristic');
   this.bluetoothDevice = null;
   this.dataCharacteristic = null;
   this.onClear();
}


//--------------------------------------------------
//reset(disconnect & clear)
//--------------------------------------------------
BlueJelly.prototype.reset= function() {
  console.log('Excute : reset');
  this.disconnect(); //disconnect() is not Promise Object
  this.clear();
  this.onReset();
}


//--------------------------------------------------
//micro:bit UUID(class constant)
//--------------------------------------------------
Object.defineProperty(BlueJelly, 'MICROBIT_BASE_UUID', {value: "e95d0000-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_SERVICESGENERIC_ACCESS', {value: "00001800-0000-1000-8000-00805f9b34fb", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_DEVICE_NAME', {value: "00002a00-0000-1000-8000-00805f9b34fb", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_APPEARANCE', {value: "00002a01-0000-1000-8000-00805f9b34fb", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_PERIPHERAL_PREFERRED_CONNECTION_PARAMETERS', {value: "00002a04-0000-1000-8000-00805f9b34fb", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_GENERIC_ATTRIBUTE', {value: "00001801-0000-1000-8000-00805f9b34fb", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_SERVICE_CHANGED', {value: "2a05", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_DEVICE_INFORMATION', {value: "0000180a-0000-1000-8000-00805f9b34fb", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_MODEL_NUMBER_STRING', {value: "00002a24-0000-1000-8000-00805f9b34fb", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_SERIAL_NUMBER_STRING', {value: "00002a25-0000-1000-8000-00805f9b34fb", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_HARDWARE_REVISION_STRING', {value: "00002a27-0000-1000-8000-00805f9b34fb", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_FIRMWARE_REVISION_STRING', {value: "00002a26-0000-1000-8000-00805f9b34fb", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_MANUFACTURER_NAME_STRING', {value: "00002a29-0000-1000-8000-00805f9b34fb", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_ACCELEROMETER_SERVICE', {value: "e95d0753-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_ACCELEROMETER_DATA', {value: "e95dca4b-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_ACCELEROMETER_PERIOD', {value: "e95dfb24-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_MAGNETOMETER_SERVICE', {value: "e95df2d8-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_MAGNETOMETER_DATA', {value: "e95dfb11-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_MAGNETOMETER_PERIOD', {value: "e95d386c-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_MAGNETOMETER_BEARING', {value: "e95d9715-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_MAGNETOMETER_CALIBRATION', {value: "e95db358-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_BUTTON_SERVICE', {value: "e95d9882-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_BUTTON_A_STATE', {value: "e95dda90-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_BUTTON_B_STATE', {value: "e95dda91-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_IO_PIN_SERVICE', {value: "e95d127b-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_PIN_DATA', {value: "e95d8d00-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_PIN_AD_CONFIGURATION', {value: "e95d5899-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_PIN_IO_CONFIGURATION', {value: "e95db9fe-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_PWM_CONTROL', {value: "e95dd822-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_LED_SERVICE', {value: "e95dd91d-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_LED_MATRIX_STATE', {value: "e95d7b77-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_LED_TEXT', {value: "e95d93ee-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_SCROLLING_DELAY', {value: "e95d0d2d-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_EVENT_SERVICE', {value: "e95d93af-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_MICROBIT_REQUIREMENTS', {value: "e95db84c-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_MICROBIT_EVENT', {value: "e95d9775-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_CLIENT_REQUIREMENTS', {value: "e95d23c4-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_CLIENT_EVENT', {value: "e95d5404-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_DFU_CONTROL_SERVICE', {value: "e95d93b0-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_DFU_CONTROL', {value: "e95d93b1-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_TEMPERATURE_SERVICE', {value: "e95d6100-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_TEMPERATURE', {value: "e95d9250-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_TEMPERATURE_PERIOD', {value: "e95d1b25-251d-470a-a062-fa1922dfa9a8", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_UART_SERVICE', {value: "6e400001-b5a3-f393-e0a9-e50e24dcca9e", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_TX_CHARACTERISTIC', {value: "6e400002-b5a3-f393-e0a9-e50e24dcca9e", writable: false});
Object.defineProperty(BlueJelly, 'MICROBIT_RX_CHARACTERISTIC', {value: "6e400003-b5a3-f393-e0a9-e50e24dcca9e", writable: false});
```
</div></details>

# センシングデータ 受信確認プログラム (HTML/JavaScript)
加速度・照度・温度センサーの値が正しく取得できているかを確認します．

:::note warn
BlueJelly.jsを同じディレクトリに置く必要があります．
:::

<details><summary>受信確認用のHTML/JavaScriptのコード</summary><div>

```data-recive-test.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>micro:bit Sensor Data</title>
    <script src="bluejelly.js"></script> <!-- BlueJelly ライブラリの読み込み -->
    
    <style>
        #sensorData {
            font-size: 20px;
            margin-top: 20px;
        }
        #dataList {
            font-size: 16px;
            margin-top: 20px;
            max-height: 200px;
            overflow-y: scroll;
        }
    </style>
</head>
<body>
    <h1>micro:bit 取得データ表示</h1>

    <button id="startButton">micro:bit 接続</button>

    <div id="sensorData">
        <p><strong>加速度 (X, Y, Z): </strong><span id="accData">0, 0, 0</span></p>
        <p><strong>温度: </strong><span id="tempData">0</span></p>
        <p><strong>明るさ: </strong><span id="lightData">0</span></p>
    </div>

    <div id="dataList">
        <ul id="sensorDataList"></ul>
    </div>

    <script>
        const blueJelly = new BlueJelly();

        // UUIDの設定（UARTサービスを利用）
        const serviceUUID = '6e400001-b5a3-f393-e0a9-e50e24dcca9e'; // UART Service
        const characteristicUUID = '6e400002-b5a3-f393-e0a9-e50e24dcca9e'; // TX Characteristic

        let characteristic;

        // ボタンが押されたときにBluetoothデバイスを接続
        document.getElementById('startButton').addEventListener('click', async () => {
            try {
                // デバイスをスキャンして接続
                const device = await navigator.bluetooth.requestDevice({
                    filters: [{ namePrefix: 'BBC micro:bit' }], // 名前が "BBC micro:bit" を含むデバイスに限定
                    optionalServices: [serviceUUID]
                });
                console.log('Device selected:', device);

                const server = await device.gatt.connect();
                console.log('Connected to GATT server');

                const service = await server.getPrimaryService(serviceUUID);
                console.log('Service found');

                characteristic = await service.getCharacteristic(characteristicUUID);
                console.log('Characteristic found');

                // 通知の開始
                await characteristic.startNotifications();
                console.log('Notifications started');

                // データ受信時のイベントリスナーを設定
                characteristic.addEventListener('characteristicvaluechanged', event => {
                    const value = event.target.value; // DataView
                    const decoder = new TextDecoder('utf-8');
                    const decodedString = decoder.decode(value); // UTF-8デコード
                    console.log(decodedString);

                    // リストに追加
                    //const dataList = document.getElementById('sensorDataList');
                    //const newListItem = document.createElement('li');
                    //newListItem.textContent = `${decodedString}`;
                    //dataList.appendChild(newListItem);

                    // データ解析
                    const accMatch = decodedString.match(/a:([\d\-,]+)/);
                    const tempMatch = decodedString.match(/t:(\d+)/);
                    const lightMatch = decodedString.match(/l:(\d+)/);

                    if (accMatch) {
                        const accData = accMatch[1].split(',');
                        document.getElementById('accData').textContent = `X: ${accData[0]}, Y: ${accData[1]}, Z: ${accData[2]}`;
                    }

                    if (tempMatch) {
                        document.getElementById('tempData').textContent = tempMatch[1];
                    }

                    if (lightMatch) {
                        document.getElementById('lightData').textContent = lightMatch[1];
                    }
                });
            } catch (error) {
                console.error('Connection Error:', error);
            }
        });
    </script>
</body>
</html>
```
</div></details>

# センシングデータを用いたテトリス風ゲーム

<details><summary>センシングデータを用いたテトリス風ゲームのHTML/JavaScriptのコード</summary><div>

```sensor-data-tetris.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>micro:bit Tetris</title>
    <script src="bluejelly.js"></script>
    <style>
        body {
            margin: 0;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            background-color: rgba(0, 0, 0, 1); /* 初期の背景色 */
            transition: background-color 0.5s;
        }
        canvas {
            display: block;
            margin: 20px auto;
            border: 5px solid white; /* 枠線を追加 */
            border-radius: 10px;
        }
        button {
            font-size: 18px;
            padding: 10px 20px;
            margin-top: 20px;
        }
    </style>
</head>
<body>
    <button id="connectButton">micro:bit 接続</button>
    <canvas id="gameCanvas" width="300" height="600"></canvas>

    <script>
        const blueJelly = new BlueJelly();
        const serviceUUID = '6e400001-b5a3-f393-e0a9-e50e24dcca9e'; // UART Service
        const characteristicUUID = '6e400002-b5a3-f393-e0a9-e50e24dcca9e'; // TX Characteristic

        let characteristic;
        let gameStarted = false;
        let currentBlock;
        let board = Array.from({ length: 20 }, () => Array(10).fill(0)); // 横10列
        let fallSpeed = 1000; // ms
        let lastFallTime = Date.now();
        let lastRotationTime = 0; // 回転のクールダウン管理
        const rotationCooldown = 1000; // 回転のクールダウン時間 (ms)
        let brightness = 1; // 背景の明るさ
        const canvas = document.getElementById("gameCanvas");
        const ctx = canvas.getContext("2d");

        // ブロックの形状パターン（7種類）
        const blockShapes = [
            [[1, 1], [1, 1]], // 正方形
            [[0, 1, 0], [1, 1, 1]], // T字
            [[1, 1, 0], [0, 1, 1]], // Z字
            [[0, 1, 1], [1, 1, 0]], // 逆Z字
            [[1, 0], [1, 0], [1, 1]], // L字
            [[0, 0, 1], [1, 1, 1]], // 逆L字
            [[1, 1, 1, 1]], // 長方形
        ];

        let bButtonPressed = false; // Bボタンの状態
        let lastBPressTime = 0; // 最後にBボタンが押された時間

        document.getElementById('connectButton').addEventListener('click', async () => {
            try {
                const device = await navigator.bluetooth.requestDevice({
                    filters: [{ namePrefix: 'BBC micro:bit' }],
                    optionalServices: [serviceUUID]
                });
                const server = await device.gatt.connect();
                const service = await server.getPrimaryService(serviceUUID);
                characteristic = await service.getCharacteristic(characteristicUUID);

                await characteristic.startNotifications();
                characteristic.addEventListener('characteristicvaluechanged', handleSensorData);

                document.getElementById('connectButton').style.display = 'none'; // 接続ボタンを非表示
                gameStarted = true; // ゲームを開始
                startGame();
            } catch (error) {
                console.error('Connection Error:', error);
            }
        });

        function handleSensorData(event) {
            const value = event.target.value;
            const decoder = new TextDecoder('utf-8');
            const decodedString = decoder.decode(value);

            const accMatch = decodedString.match(/a:([\d\-,]+)/);
            const tempMatch = decodedString.match(/t:(\d+)/);
            const lightMatch = decodedString.match(/l:(\d+)/);
            const buttonBMatch = decodedString.match(/b:(\d+)/); // Bボタンの入力

            // 加速度センサーのデータ
            if (accMatch) {
                const accData = accMatch[1].split(',');
                const tiltX = parseInt(accData[0]) / 2; // 感度調整
                const tiltY = parseInt(accData[1]) / 2; // 感度調整

                if (tiltX > 100 && canMove(currentBlock, 1, 0)) currentBlock.x++; // 右移動
                if (tiltX < -100 && canMove(currentBlock, -1, 0)) currentBlock.x--; // 左移動

                // 手前に傾けた際の回転
                const now = Date.now();
                if (tiltY > 200 && now - lastRotationTime > rotationCooldown) {
                    rotateBlock(currentBlock);
                    lastRotationTime = now; // 回転時間を更新
                }
            }

            // 温度センサーで落下速度調整
            if (tempMatch) {
                const temperature = parseInt(tempMatch[1]);
                fallSpeed = Math.max(200, 1000 - (temperature - 20) * 250); // 温度で落下速度調整
            }

            // 明るさ調整
            if (lightMatch) {
                const lightLevel = parseInt(lightMatch[1]);
                brightness = Math.min(1, lightLevel / 255); // 明るさを計算
                document.body.style.backgroundColor = `rgba(0, 0, 0, ${1 - brightness})`; // 背景色に適用
            }

            // Bボタンが押された時の処理
            if (buttonBMatch && buttonBMatch[1] === "1") {
                const now = Date.now();
                if (!bButtonPressed || now - lastBPressTime > 1000) { // 1秒間のクールダウン
                    bButtonPressed = true;
                    lastBPressTime = now;
                    dropBlockImmediately();
                }
            } else {
                bButtonPressed = false; // ボタンが離された場合はリセット
            }
        }

        function rotateBlock(block) {
            const newShape = block.shape[0].map((_, index) =>
                block.shape.map(row => row[index]).reverse()
            );
            if (canMove({ ...block, shape: newShape }, 0, 0)) block.shape = newShape; // 衝突がない場合のみ回転
        }

        function dropBlockImmediately() {
            while (canMove(currentBlock, 0, 1)) {
                currentBlock.y++;
            }
            fixBlockToBoard(currentBlock);
            clearLines();
            currentBlock = generateBlock();
        }

        function canMove(block, dx, dy) {
            return block.shape.every((row, y) =>
                row.every((value, x) => {
                    if (!value) return true; // 空白部分は無視
                    const newX = block.x + x + dx;
                    const newY = block.y + y + dy;
                    return newX >= 0 && newX < 10 && newY < 20 && board[newY]?.[newX] === 0;
                })
            );
        }

        function fixBlockToBoard(block) {
            block.shape.forEach((row, y) => {
                row.forEach((value, x) => {
                    if (value) board[block.y + y][block.x + x] = value;
                });
            });
        }

        function clearLines() {
            board = board.filter(row => row.some(cell => cell === 0)); // 空白の行のみ残す
            while (board.length < 20) board.unshift(Array(10).fill(0)); // 上に空白行を追加
        }

        function generateBlock() {
            const shape = blockShapes[Math.floor(Math.random() * blockShapes.length)];
            return { x: Math.floor((10 - shape[0].length) / 2), y: 0, shape }; // 初期位置調整
        }

        function drawBoard() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            for (let y = 0; y < 20; y++) {
                for (let x = 0; x < 10; x++) {
                    if (board[y][x]) {
                        ctx.fillStyle = "blue";
                        ctx.fillRect(x * 30, y * 30, 30, 30);
                    }
                }
            }
        }

        function drawBlock(block) {
            ctx.fillStyle = "red";
            block.shape.forEach((row, dy) => {
                row.forEach((value, dx) => {
                    if (value) {
                        ctx.fillRect((block.x + dx) * 30, (block.y + dy) * 30, 30, 30);
                    }
                });
            });
        }

        function updateGame() {
            if (!gameStarted) return;

            // ゲームオーバーチェック
            if (board[0].some(cell => cell !== 0)) {
                alert("Game Over!");
                gameStarted = false;
                return;
            }

            const now = Date.now();
            if (now - lastFallTime > fallSpeed) {
                if (canMove(currentBlock, 0, 1)) {
                    currentBlock.y++;
                } else {
                    fixBlockToBoard(currentBlock);
                    clearLines();
                    currentBlock = generateBlock();
                }
                lastFallTime = now;
            }

            drawBoard();
            drawBlock(currentBlock);
            requestAnimationFrame(updateGame);
        }

        function startGame() {
            currentBlock = generateBlock();
            updateGame();
        }
    </script>
</body>
</html>
```
</div></details>

**こちらからプレイできます．↓**

https://arita-kaito.github.io/microbit-tetris/

# 工夫した点
- ブロックの回転
    micro:bitを手前に傾けるとブロックが90°回転しますが、1度傾けると2回連続で回転してしまったため、1度傾けられたら1秒間は反応しないようにしました．
- 文字列でのセンシングデータ送信
    micro:bitがBluetoothで送信できる文字列には上限があるようで、全てのセンシングデータを一度に送信することはできなかったため、2度に分けて送信するようにしました．

# 改善したかった点
温度によってブロックの落下スピードが変化するようにしておりますが、ゲームスタート時の温度を基準にして、micro:bitで操作しきれる範囲での調整にすれば良かったです．
(micro:bitは大学に返却してしまい、実機でのテストができないため、どうにもできず...)
