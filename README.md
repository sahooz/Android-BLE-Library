# Android BLE Library

[ ![Download](https://api.bintray.com/packages/nordic/android/no.nordicsemi.android%3Able/images/download.svg) ](https://bintray.com/nordic/android/no.nordicsemi.android%3Able/_latestVersion)

An Android library that solves a lot of Android's Bluetooth Low Energy problems. 
The [BleManager](https://github.com/NordicSemiconductor/Android-BLE-Library/blob/master/ble/src/main/java/no/nordicsemi/android/ble/BleManager.java)
class exposes high level API for connecting and communicating with Bluetooth LE peripherals.
The API is clean and easy to read.

## Features

**BleManager** class provides the following features:

1. Connection, with automatic retries
2. Service discovery
3. Bonding (optional) and removing bond information (using reflections)
4. Automatic handling of Service Changed indications
5. Device initialization
6. Asynchronous and synchronous BLE operations using queue
7. Splitting and merging long packets when writing and reading characteristics and descriptors
8. Requesting MTU and connection priority (on Android Lollipop or newer)
9. Reading and setting preferred PHY (on Android Oreo or newer)
10. Reading RSSI
11. Refreshing device cache (using reflections)
12. Reliable Write support
13. Operation timeouts (for *connect*, *disconnect* and *wait for notification* requests)
14. Error handling
15. Logging
16. GATT server (since version 2.2)

The library **does not provide support for scanning** for Bluetooth LE devices.
For scanning, we recommend using 
[Android Scanner Compat Library](https://github.com/NordicSemiconductor/Android-Scanner-Compat-Library)
which brings almost all recent features, introduced in Lollipop and later, to the older platforms. 

### Version 2.2

New features added in version 2.2:

1. GATT Server support. This includes setting up the local GATT server on the Android device, new 
   requests for server operations: 
   * *wait for read*, 
   * *wait for write*, 
   * *send notification*, 
   * *send indication*,
   * *set characteristic value*,
   * *set descriptor value*.
2. New conditional requests: 
   * *waif if*,
   * *wait until*.
3. BLE operations are no longer called from the main thread.
4. There's a new option to set a handler for invoking callbacks. A handler can also be set per-callback.

### Migration to version 2.2

Version 2.2 breaks some API known from version 2.1.1.
Check out [migration guide](MIGRATION.md).

## Importing

#### Maven Central or jcenter

The library may be found on jcenter and Maven Central repository. 
Add it to your project by adding the following dependency:

```grovy
implementation 'no.nordicsemi.android:ble:2.2.4'
```
The last version not migrated to AndroidX is 2.0.5.

To import the BLE library with set of parsers for common Bluetooth SIG characteristics, use:
```grovy
implementation 'no.nordicsemi.android:ble-common:2.2.4'
```
For more information, read [this](BLE-COMMON.md).

An extension for easier integration with `LiveData` is available after adding:
```grovy
implementation 'no.nordicsemi.android:ble-livedata:2.2.4'
```
This extension adds `ObservableBleManager` with `state` and `bondingState` properties, which 
notify about connection and bond state using `androidx.lifecycle.LiveData`.

#### As a library module

Clone this project and add *ble* module as a dependency to your project:

1. In *settings.gradle* file add the following lines:
```groovy
include ':ble'
project(':ble').projectDir = file('../Android-BLE-Library/ble')
```
2. In *app/build.gradle* file add `implementation project(':ble')` inside dependencies.
3. Sync project and build it.

You may do the same with other modules available in this project. Keep in mind, that
*ble-livedata* module requires Kotlin, but no special changes are required in the app.

#### Setting up

The library uses Java 1.8 features. Make sure your *build.gradle* includes the following 
configuration:

```groovy
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
    // For Kotlin projects additionally:
    kotlinOptions {
        jvmTarget = "1.8"
    }
```

## Usage

A `BleManager` instance is responsible for connecting and communicating with a single peripheral.
Multiple manager instances are allowed. Extend `BleManager` with you manager where you define the
high level device's API.

`BleManager` may be used in different ways:
1. In a Service, for a single connection - see [nRF Toolbox](https://github.com/NordicSemiconductor/Android-nRF-Toolbox) -> RSC profile, 
2. In a Service with multiple connections - see nRF Toolbox -> Proximity profile, 
3. From ViewModel's repo - see [Architecture Components](https://developer.android.com/topic/libraries/architecture/index.html) 
and [nRF Blinky](https://github.com/NordicSemiconductor/Android-nRF-Blinky),
4. As a singleton - not recommended, see nRF Toolbox -> HRM.

The first step is to create your BLE Manager implementation, like below. The manager should
act as API of your remote device, to separate lower BLE layer from the application layer.

```java
class MyBleManager extends BleManager {
    final static UUID SERVICE_UUID = UUID.fromString("XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX");
    final static UUID FIRST_CHAR   = UUID.fromString("XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX");
    final static UUID SECOND_CHAR  = UUID.fromString("XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX");

    // Client characteristics
    private BluetoothGattCharacteristic firstCharacteristic, secondCharacteristic;

    MyBleManager(@NonNull final Context context) {
        super(context);
    }

    @NonNull
    @Override
    protected BleManagerGattCallback getGattCallback() {
        return new MyManagerGattCallback();
    }

    @Override
    public void log(final int priority, @NonNull final String message) {
        if (Build.DEBUG || priority == Log.ERROR) {
            Log.println(priority, "MyBleManager", message);
        }
    }

    /**
     * BluetoothGatt callbacks object.
     */
    private class MyManagerGattCallback extends BleManagerGattCallback {

        // This method will be called when the device is connected and services are discovered.
        // You need to obtain references to the characteristics and descriptors that you will use.
        // Return true if all required services are found, false otherwise.
        @Override
        public boolean isRequiredServiceSupported(@NonNull final BluetoothGatt gatt) {
            final BluetoothGattService service = gatt.getService(SERVICE_UUID);
            if (service != null) {
                firstCharacteristic = service.getCharacteristic(FIRST_CHAR);
                secondCharacteristic = service.getCharacteristic(SECOND_CHAR);
            }
            // Validate properties
            boolean notify = false;
            if (firstCharacteristic != null) {
                final int properties = dataCharacteristic.getProperties();
                notify = (properties & BluetoothGattCharacteristic.PROPERTY_NOTIFY) != 0;
            }
            boolean writeRequest = false;
            if (secondCharacteristic != null) {
                final int properties = controlPointCharacteristic.getProperties();
                writeRequest = (properties & BluetoothGattCharacteristic.PROPERTY_WRITE) != 0;
                secondCharacteristic.setWriteType(BluetoothGattCharacteristic.WRITE_TYPE_DEFAULT);
            }
            // Return true if all required services have been found
            return firstCharacteristic != null && secondCharacteristic != null
                    && notify && writeRequest;
        }

        // If you have any optional services, allocate them here. Return true only if
        // they are found. 
        @Override
        protected boolean isOptionalServiceSupported(@NonNull final BluetoothGatt gatt) {
            return super.isOptionalServiceSupported(gatt);
        }

        // Initialize your device here. Often you need to enable notifications and set required
        // MTU or write some initial data. Do it here.
        @Override
        protected void initialize() {
            // You may enqueue multiple operations. A queue ensures that all operations are 
            // performed one after another, but it is not required.
            beginAtomicRequestQueue()
                .add(requestMtu(247) // Remember, GATT needs 3 bytes extra. This will allow packet size of 244 bytes.
                    .with((device, mtu) -> log(Log.INFO, "MTU set to " + mtu))
                    .fail((device, status) -> log(Log.WARN, "Requested MTU not supported: " + status)))
                .add(setPreferredPhy(PhyRequest.PHY_LE_2M_MASK, PhyRequest.PHY_LE_2M_MASK, PhyRequest.PHY_OPTION_NO_PREFERRED)
                    .fail((device, status) -> log(Log.WARN, "Requested PHY not supported: " + status)))
                .add(enableNotifications(firstCharacteristic))
                .done(device -> log(Log.INFO, "Target initialized"))
                .enqueue();			
            // You may easily enqueue more operations here like such:
            writeCharacteristic(secondCharacteristic, "Hello World!".getBytes())
                .done(device -> log(Log.INFO, "Greetings sent"))
                .enqueue();
            // Set a callback for your notifications. You may also use waitForNotification(...).
            // Both callbacks will be called when notification is received.
            setNotificationCallback(firstCharacteristic, callback);
            // If you need to send very long data using Write Without Response, use split()
            // or define your own splitter in split(DataSplitter splitter, WriteProgressCallback cb). 
            writeCharacteristic(secondCharacteristic, "Very, very long data that will no fit into MTU")
                .split()
                .enqueue();
        }

        @Override
        protected void onDeviceDisconnected() {
            // Device disconnected. Release your references here.
            firstCharacteristic = null;
            secondCharacteristic = null;
        }
    }
    
    // Define your API.
    
    private abstract class FluxHandler implements ProfileDataCallback {		
        @Override
        public void onDataReceived(@NonNull final BluetoothDevice device, @NonNull final Data data) {
            // Some validation?
            if (data.size() != 1) {
                onInvalidDataReceived(device, data);
                return;
            }
            onFluxCapacitorEngaged();
        }
        
        abstract void onFluxCapacitorEngaged();
    }
    
    /** Initialize time machine. */
    public void enableFluxCapacitor(final int year) {
        waitForNotification(firstCharacteristic)
            .trigger(
                writeCharacteristic(secondCharacteristic, new FluxJumpRequest(year))
                    .done(device -> log(Log.INDO, "Power on command sent"))
             )
            .with(new FluxHandler() {
                public void onFluxCapacitorEngaged() {
                    log(Log.WARN, "Flux Capacitor enabled! Going back to the future in 3 seconds!");
                    
                    sleep(3000).enqueue();
                    write(secondCharacteristic, "Hold on!".getBytes())
                        .done(device -> log(Log.WARN, "It's " + year + "!"))
                        .fail((device, status) -> "Not enough flux? (status: " + status + ")")
                        .enqueue();
                }
            })
            .enqueue();
    }
    
    /** 
    * Aborts time travel. Call during 3 sec after enabling Flux Capacitor and only if you don't 
    * like 2020. 
    */
    public void abort() {
        cancelQueue();
    }
}
```

To connect to a Bluetooth LE device using GATT, create a manager instance:

```java
class MyRepo implements ConnectionObserver {
    private MyBleManager manager;

    // [...]

    void connect(@NonNull final BluetoothDevice device) {
        manager = new MyBleManager(context);
        manager.setConnectionObserver(this);
        manager.connect(device)
            .timeout(100000)
            .retry(3, 100)
            .done(device -> Log.i(TAG, "Device initiated"))
            .enqueue();
    }
    
    // [...]

}
```

#### Adding GATT Server support

Starting from version 2.2 you may now define and use the GATT server in the BLE Library.

First, override a `BleServerManager` class and override `initializeServer()` method. Some helper
methods, like `characteristic(...)`, `descriptor(...)` and their shared counterparts were created 
for making the initialization more readable.

```java
public class ServerManager extends BleServerManager {

    ServerManager(@NonNull final Context context) {
        super(context);
    }

    @NonNull
    @Override
    protected List<BluetoothGattService> initializeServer() {
        // In this example there's only one service, so singleton list is created.
        return Collections.singletonList(
            service(SERVICE_UUID,
                characteristic(CHAR_UUID,
                    BluetoothGattCharacteristic.PROPERTY_WRITE // properties
                      | BluetoothGattCharacteristic.PROPERTY_NOTIFY
                      | BluetoothGattCharacteristic.PROPERTY_EXTENDED_PROPS,
                    BluetoothGattCharacteristic.PERMISSION_WRITE, // permissions
                    null, // initial data
                    cccd(), reliableWrite(), description("Some description", false) // descriptors
            ))
        );
    }
}
```
Instantiate the server and set the callback listener:
```java
class MyRepo implements ServerObserver {
    private ServerManager serverManager;

    // [...]

    void init() {
        serverManager = new ServerManager(context);
        serverManager.setServerObserver(this);
    }
    
    // [...]

}
```
Set the server manager for each client connection:
```java
class MyRepo implements ConnectionObserver, BondingObserver {
    private ServerManager serverManager;
    private MyBleManager manager;

    // [...]

    // This method may be called from e.g. 
    // ServerObserver#onDeviceConnectedToServer(@NonNull final BluetoothDevice device)
    void connect(@NonNull final BluetoothDevice device) {
        manager = new MyBleManager(context);
        manager.setConnectionObserver(this);
        manager.setBondingObserver(this);
        // Use the manager with the server.
        manager.useServer(serverManager);

        manager.connect(device)
            // [...]
            .enqueue();
    }

    // [...]
}
```
The `ServerObserver.onServerReady()` will be invoked when all service were added.
You may initiate your connection there.

In your client manager class, override the following method:
```java
class MyBleManager extends BleManager {
    // [...]	

    // Server characteristics
    private BluetoothGattCharacteristic serverCharacteristic;

    // [...]

    /**
     * BluetoothGatt callbacks object.
     */
    private class MyManagerGattCallback extends BleManagerGattCallback {
        // [...]	
    
        @Override
        protected void onServerReady(@NonNull final BluetoothGattServer server) {
            // Obtain your server attributes.
            serverCharacteristic = server
                .getService(SERVICE_UUID)
                .getCharacteristic(CHAR_UUID);
            
            // Set write callback, if you need. It will be called when the remote device writes
            // something to the given server characteristic.
            setWriteCallback(serverCharacteristic)
                .with((device, data) ->
                    sendNotification(otherCharacteristic, "any data".getBytes())
                        .enqueue()
                );
        }
        
        // [...]

        @Override
        protected void onDeviceDisconnected() {
            // [...]
            serverCharacteristic = null;
        }
    }
    
    // [...]
}
``` 

## Examples

Find the simple example here [Android nRF Blinky](https://github.com/NordicSemiconductor/Android-nRF-Blinky).

For an example how to use it from an Activity or a Service, check the base Activity and Service 
classes in [nRF Toolbox](https://github.com/NordicSemiconductor/Android-nRF-Toolbox/tree/master/app/src/main/java/no/nordicsemi/android/nrftoolbox/profile).

## Version 1.x

The BLE library v 1.x is no longer supported. Please migrate to 2.2+ for bug fixing releases.
Find it on [version/1x branch](https://github.com/NordicSemiconductor/Android-BLE-Library/tree/version/1x).

Migration guide is available [here](MIGRATION.md).