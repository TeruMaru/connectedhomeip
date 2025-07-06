## Source Code

### Directory Structure

The CHIP `src` directory is structured as follows:

| File / Folder | Contents                                           |
| ------------- | -------------------------------------------------- |
| app           | Application Layer -- Zigbee Cluster Library (ZCL)  | -> Read server dir
| ble           | BLE Layer -- Bluetooth Transport Protocol (BTP)    | -> Just bluetooth lower level API, can skip
| controller    | Controller API                                     | -> See EstablishPASEConnection() to see how devices are connected via BLE (Focus on code blocks denoted by CONFIG_NETWORK_LAYER_BLE)
| crypto        | Cryptography libraries                             |
| darwin        | Darwin Framework (iOS and macOS)                   |
| include       | Public headers                                     |
| inet          | Network Layer -- TCP and UDP endpoints             |
| lib           | Core and Support libraries                         | -> Read dnssd dir
| lwip          | Lightweight IP adaptation (to third_party library) |
| platform      | Device Layer -- platform portability adaptations   | -> Focus on NRF Connect and Zephyr
| qrcodetool    | QR code tool                                       |
| setup_payload | QR code setup data encode / decode library         |
| ------------  | -------------------------------------------------- | -> Creates a pool of Timers that invoke registered callback funcitons 
|               |                                                    | when they expire, but not immediately. The system layer keeps a list
| system        | System Layer -- common APIs for mem, work, etc.    | at expired timers, and invoke mentioned callbacks inside the PlatformManager's
|               |                                                    | event handling loop (specifically RunEventLoop). This event handling loop also
| ------------  | -------------------------------------------------- | handles Socket events and messages in Zephyr's Message Queue (for Zephyr implemented PlatformManager).
| test_driver   | Framework for on-device testing                    |

#### Darwin

##### Near Field Communication Tag Reading

NFC Tag Reading is disabled by default because a paid Apple developer account is
required to have it enabled. If you want to enable it and you have a paid Apple
developer account, go to the CHIPTool iOS target and turn on Near Field
Communication Tag Reading under the Capabilities tab.
