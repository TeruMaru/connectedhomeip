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
| ------------  | -------------------------------------------------- | -> Transport layer includes a TransportManager that manages transport objects (in transport/raw)
| transport     | Implementation of transport layer                  | whose methods perform operations like sending messages, joining or leaving
| ------------- | -------------------------------------------------- | multicast groups, etc.
#### Darwin

##### Near Field Communication Tag Reading

NFC Tag Reading is disabled by default because a paid Apple developer account is
required to have it enabled. If you want to enable it and you have a paid Apple
developer account, go to the CHIPTool iOS target and turn on Near Field
Communication Tag Reading under the Capabilities tab.


###### WORKNOTES COPY
Ref: https://docs.nordicsemi.com/bundle/ncs-2.1.1/page/nrf/ug_matter_gs_adding_clusters.html#edit_the_main_loop_of_the_application


Most of the libraries are in: /home/phhuynh/Work/Matter/sdk-nrf/samples/matter/common

ZCL stands for Zigbee Cluster Library. See https://docs.nordicsemi.com/bundle/ncs-2.5.2/page/nrf/protocols/matter/getting_started/adding_clusters.html#edit_clusters_using_the_zap_tool

After generated, the callbacks will be initialized at /home/phhuynh/Work/Matter/sdk-nrf/samples/matter/light_bulb/src/zap-generated/callback-stub.cpp. For specific usecases, we must implement the emberAF....() API in zcl_callbacs.cpp



PlatformManager (/home/phhuynh/Work/Matter/connectedhomeip/src/include/platform/PlatformManager.h) is the interface class. PlatformManagerImpl (/home/phhuynh/Work/Matter/connectedhomeip/src/platform/Zephyr/PlatformManagerImpl.h) is the implementation class for Zephyr. PlatformManagerImpl also deploys Curiously Recurring Template Pattern (CRTP: https://en.wikipedia.org/wiki/Curiously_recurring_template_pattern) from base class GenericPlatformManagerImpl_Zeyphyr(/home/phhuynh/Work/Matter/connectedhomeip/src/include/platform/internal/GenericPlatformManagerImpl_Zephyr.h). GenericPlatformManagerImpl_Zeyphyr is inherited from GenericPlatformManagerImpl(/home/phhuynh/Work/Matter/connectedhomeip/src/include/platform/internal/GenericPlatformManagerImpl.h). With CRTP, base classes (GenericPlatformManagerImpl and GenericPlatformManagerImpl_Zephyr) can access methods from derived class (PlatformManagerImpl) via static casting.


For CHIPController, the correspondance for InitChipStack() is CHIPCommand::MaybeSetUpStack()


Functions implemented in /home/phhuynh/Work/Matter/connectedhomeip/src/app/util/attribute-table.cpp will be used in zap-generated/attribtues/Accessors.cpp that will enventually be used by Protocols::InteractionModel

================================================================
Data receipt flow:
    UDPEndpoint (or different transport endpoint) listens on bound port for UDP (or corresponding) events (see UDPEndPointImplSockets::ListenImpl()). When the socket receives an event, UDPEndPoint calls its regisgtered callback (for example, UDPEndPointImplSockets::HandlePendingIO()). The UDPEndpoint will then parse received message from the socket and eventually call its registered OnMessageReceived() function. OnMessageReceived() is registered when Transport object is initilized (See connectedhomeip/src/transport/raw/UDP.cpp for example). OnMessageReceived() will call the Base Transport classes' HandleMessageReceived() method, which will then call it's delegated classes' HandleMessageReceived(). The delegated class for Base's HandleMessageReceived() is TransportMgrBase.
    |--> TransportMgrBase.HandleMessageReceived()
        |--> SessionManager.OnMessageReceived()
            |--> SecureGroupMessageDispatch()|SecureUnicastMessageDispatch()|UnauthenticatedMessageDispatch()
                |--> SessionManager.mCB->OnMessageReceived() == ExchangeManager.OnMessageReceived()
                    |--> ExchangeContext.HandleMessage()
                        |--> Retrieve delegate from Registered UnsolicitedMessageHandlers by calling OnUnsolicitedMessageReceived() 
                        |--> Call Registered UnsolicitedMessageHandlers' OnMessageReceived() method based on matching protocol or message ID
                            |--> Call retrieved delegate->OnMessageReceived() method

For example, when the device receives Protocols::SecureChannel::MsgType::PBKDFParamRequest (For PASE session establishment), TransportMgrBase.HandleMessageReceived() following the above flow will call CommissioningWindowManager.OnUnsolicitedMessageReceived() since CommissioningWindowManager object is the handler registered to the ExchangeManager when CommissioningWindowManager.AdvertiseAndListenForPASE() had been invoked previously. CommissioningWindowManager.OnUnsolicitedMessageReceived() will return its mPairingSession member(defined in connectedhomeip/src/protocols/secure_channel/PASESession.cpp) as the delegate whose OnMessageReceived() method will be called by the created ExchangeContext onbject (last step in the above flow).

+-------------+    +--------------+    +-------------------+    +-----------------+    +------------------+
| UDP Sockets | -> | UDP Endpoint | -> | Transport Manager | -> | Session Manager | -> | Exchange Manager |
+-------------+    +--------------+    +-------------------+    +-----------------+    +------------------+     

================================================================


PlatformManager manages or inlcudes:
1. SystemLayer
2. UDPEndPointManager, a.k.a UDP layer
3. Configuration Manager

================================================================

FabricTable is stored in /tmp/chip_config.ini with keys defined in connectedhomeip/src/lib/support/DefaultStorageKeyAllocator.h (maybe more), and value stored in TLV (Tag Lenght Value) format. TAG int TLV is a 64-bit value wrapped in TAG class defined in connectedhomeip/src/lib/core/TLVTags.h

    // The storage of the tag value uses the following encoding:
    //
    //  63                              47                              31
    // +-------------------------------+-------------------------------+----------------------------------------------+
    // | Vendor id (bitwise-negated)   | Profile num (bitwise-negated) | Tag number                                   |
    // +-------------------------------+-------------------------------+----------------------------------------------+
    //
    // OR
    //
    //  63                              47                              31
    // +-------------------------------+-------------------------------+----------------------------------------------+
    // | kSpecialTagProfileId == 0xFFFF.FFFF                            | Tag number                                   |
    // +-------------------------------+-------------------------------+----------------------------------------------+
    // Vendor id and profile number are bitwise-negated in order to optimize the code size when
    // using context tags, the most commonly used tags in the SDK.

TLV Control byte

    //  7                  5                                         0
    // +-------------------+-----------------------------------------+
    // |    Tag control    |                TLV type                 |
    // +-------------------+-----------------------------------------+

    * Tag control: How many bytes of tag in the following TLV. The bytes are encoded as following:
        +--------------------+----------------------+
        | Tag control value  |  Number of tag bytes |
        +--------------------+----------------------+
        |        0           |          0           |
        |        1           |          1           |
        |        2           |          2           |
        |        3           |          4           |
        |        4           |          2           |
        |        5           |          4           |
        |        6           |          6           |
        |        7           |          8           |
        +--------------------+----------------------+
    * TLV type represents the data type of the Length/Value portion in TLV
        +--------------------+-------------------+------------------------+---------------------|
        |      TLV Type      | TLV Type encoding | LV field size encoding |   LV field size     |
        +--------------------+-------------------+------------------------+---------------------|
        |NotSpecified        | -1                |                        | kTLVFieldSize_0Byte |
        |UnknownContainer    | -2                |                        | kTLVFieldSize_0Byte |
        |SignedInteger       | 0x00              | 0x00 & 0x3 = 0x0       | kTLVFieldSize_1Byte |
        |UnsignedInteger     | 0x04              | 0x04 & 0x3 = 0x0       | kTLVFieldSize_1Byte |
        |Boolean             | 0x08              | 0x08 & 0x3 = 0x0       | kTLVFieldSize_1Byte |
        |FloatingPointNumber | 0x0A              | 0x0A & 0x3 = 0x2       | kTLVFieldSize_4Byte |
        |UTF8String          | 0x0C              | 0x0C & 0x3 = 0x0       | kTLVFieldSize_1Byte |
        |ByteString          | 0x10              | 0x10 & 0x3 = 0x0       | kTLVFieldSize_1Byte |
        |Null                | 0x14              | 0x14 & 0x3 = 0x0       | kTLVFieldSize_1Byte |
        |Structure           | 0x15              | 0x15 & 0x3 = 0x1       | kTLVFieldSize_2Byte |
        |Array               | 0x16              | 0x16 & 0x3 = 0x2       | kTLVFieldSize_4Byte |
        |List                | 0x17              | 0x17 & 0x3 = 0x3       | kTLVFieldSize_8Byte |
        +--------------------+-------------------+------------------------+---------------------|
    * Only UTF8String has Length field, see `inline bool TLVTypeHasLength(TLVElementType type)` in connectedhomeip/src/lib/core/TLVTypes.h



================================================================
CASEServer will handle Unsolicited  messages of type Protocols::SecureChannel::CASE_Sigma1



===============================Opened files=================================
*** Fabric table Related
        LastKnownGoodTime.h / LastKnownGoodTime.cpp
        FabricTable.h / FabricTable.cpp
        DefaultStorageKeyAllocator.h, DataModelTypes.h
        TLVTypes.h, TLVCommon.h, TLVTags.h, TLVReader.h/TLVReader.cpp
        ExamplePersistentStorage.cpp
        ConfigurationManger.h, ConfigurationManagerImpl.h, ConfigurationManagerImpl.cpp, GenericConfigurationManagerImpl.h, GenericConfigurationManagerImpl.ipp
        SingletonConfigurationManager.cpp
        PosixConfig.h
        CHIPLinuxStorage.h, CHIPLinuxStorage.cpp, CHIPLinuxStorageIni.h, CHIPLinuxStorageIni.cpp, 

*** Session Related
        SessionManager.h/SessionManager.cpp
        ExchangeMgr.cpp
        TransportMgr.h, TransportMgrBase.cpp
        transport/Session.h, transport/Session.cpp, transport/SecureSession.h, transport/SecureSession.cpp
        SessionHolder.cpp, SecureSessionTable.h, SecureSessionTable.cpp
        ReferenceCountedHandle.h, MessageCounterManager.h, MessageCounterManager.cpp
        UnsolicitedStatusHandler.h, UnsolicitedStatusHandler.cpp, ExchangeDelegate.h
        CASEServer.cpp, CASESession.h, CASESession.cpp, PairingSession.h, PairingSession.cpp
        PASESession.h, PASESession.cpp

*** Endpoint Related
        TransportMgr.h, TransportMgrBase.h, TransportMgrBase.cpp
        raw/Tuple.h, raw/Base.h, raw/UDP.h, raw/UDP.cpp
        InetLayer.h, ConnectivityManager.h, ConnectivityManagerImpl.h, ConnectivityManagerImpl.cpp, GenericConnectivityManagerImpl_UDP.ipp
        UDPEndPoint.h, UDPEndPoint.cpp, UDPEndPointImpl.h, UDPEndPointImplSockets.h, UDPEndPointImplSockets.cpp
        GenericConnectivityManagerImpl_BLE.h, BLEManager, BLEManagerImpl.h, BLEManagerImpl.cpp, BleLayer.h, BleLayer.cpp
        GenericConnectivityManagerImpl_Thread.h, GenericConnectivityManagerImpl_Thread.ipp

*** System Layer Related
        SystemLayer.h, SystemLayerImpl.h, SystemLayerImplSelect.h, SystemLayerImplSelect.cpp
        SystemTimer.h, Globals.cpp
        WakeEvent.h, WakeEvent.cpp
        Pool.h, Pool.cpp
