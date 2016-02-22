Android BT
==========

## System Services

* When system server starts, it

    Log.i(TAG, "Starting Bluetooth Service.");
    bluetooth = new BluetoothDeviceService(context);
    bluetooth.init();
    ServiceManager.addService(Context.BLUETOOTH_SERVICE, bluetooth);
    bluetoothA2dp = new BluetoothA2dpService(context);
    ServiceManager.addService(BluetoothA2dpService.BLUETOOTH_A2DP_SERVICE,
                              bluetoothA2dp);

    int bluetoothOn = Settings.Secure.getInt(mContentResolver,
        Settings.Secure.BLUETOOTH_ON, 0);
    if (bluetoothOn > 0) {
        bluetooth.enable();
    }
* Their java classes are `BluetoothDeviceService` and `BluetoothA2dpService`.
  * Their jni interfaces are implemented by talking to `hcid`.

## Headset Profile

* The Phone App provides a service, `BluetoothHeadsetService`.
* It has a `BluetoothAudioGateway` and it calls `mAg.start` and `mAg.stop` when
  bt is on or off.
  * When started, it spawns a thread and listens on several sockets for incoming
    HSP and HFP connections.  
  * When a connection initiated by the device is made, it notifies the handler,
    which is `mIncomingConnectionHandler` of `BluetoothHeadsetService`.
* A connection initiated by user to the device is handled by
  `RfcommConnectThread`.
  * It calls `doHandsfreeSdp` or `doHeadsetSdp` to decide the rfcomm channel.
  * It creates `RfcommConnectThread` to connect to the channel.
* `HeadsetBase` manages the RFCOMM connection.
  * For device initiated connections, it jumps to `RFCOMM_CONNECTED` directly.
  * Otherwise, it opens RFCOMM socket, connects, and waits.
* When connection is made, `mConnectingStatusHandler` is notified and calls
  `mBtHandsfree.connectHeadset`.
  * That function installs handler to `+CKPD` to `headset`'s AT parser.
  * It calls `headset.startEventThread` to handle input from RFCOMM.
  * It calls `audioOn` to establish SCO connection when there is a call.
  * The progress of the SCO is handled by `BluetoothHandsfree`'s `mHandler`.
    After connection is made, `mAudioManager.setBluetoothScoOn` is called.
