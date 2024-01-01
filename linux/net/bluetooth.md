Bluetooth
=========

## Basics

- between two bluetooth devices, either can
  - scan
  - pair
  - connect
- given a new dongle,
  - let's assume the computer initiates scan/pair/connect
  - the computer scans to discover the dongle
    - the dongle must be discoverable 
  - the computer pairs with the dongle
    - the computer must be pairable
    - the dongle must be pairable
    - a pairing code might be needed
      - in bluez, an agent is enabled to provide the code interactively
    - once paired, both the computer and the dongle remember one another in
      their databases
  - the computer connects to the paired dongle
- given a paired dongle,
  - when the dongle is powered on, it connects to one of the paired computers
    in its database
  - because the dongle is in the computer's database, connection is
    established

## `bluetoothctl`

- a CLI tool to talk to `bluetoothd`
- `show` shows the controller info
- `scan` discovers discoverable devices and add them to database
- `devices` lists devices in the database
- `info <dev>` shows the device info from the database
- `pairable on` makes the controller pairable
- `agent on` enables the agent that can provide pairing code
- `default-agent` makes the agent the default one
- `pair <dev>` pairs with the device
  - the agent provides the pairing code, if needed, interactively
- `connect <dev>` connects to the device
- `trust <dev>` trusts the device
  - profiles such as HIDP requires authorization every time the device
    connects for security
  - by marking the device trusted, the authorization can be skipped

## BlueZ Build System

- `Makefile.am`
  - `pkglibexec_PROGRAMS += src/bluetoothd`
    - `src_bluetoothd_SOURCES` includes
      - `$(builtin_sources)` includes
        - `plugins/`
        - `profiles/`
      - `$(attrib_sources)` includes `attrib/`
      - `$(btio_sources)` includes `btio/`
      - `src/`
    - `src_bluetoothd_LDADD` includes
      - `lib/libbluetooth-internal.la`
      - `gdbus/libgdbus-internal.la`
      - `src/libshared-glib.la`
  - `noinst_LTLIBRARIES += lib/libbluetooth-internal.la`
    - `lib_libbluetooth_internal_la_SOURCES` includes
      - `$(lib_sources)` includes `lib/`
      - `$(extra_sources)` includes `lib/`
  - `noinst_LTLIBRARIES += gdbus/libgdbus-internal.la`
    - `gdbus_libgdbus_internal_la_SOURCES` includes `gdbus/`
  - `noinst_LTLIBRARIES += ell/libell-internal.la`
    - `ell_libell_internal_la_SOURCES` includes `ell/`
      - <git://git.kernel.org/pub/scm/libs/ell/ell.git>
  - `noinst_LTLIBRARIES += src/libshared-glib.la`
    - `src_libshared_glib_la_SOURCES` includes
      - `$(shared_sources)` includes `src/shared/`
      - `src/shared/*glib*`
  - `noinst_LTLIBRARIES += src/libshared-mainloop.la`
    - `src_libshared_mainloop_la_SOURCES` includes
      - `$(shared_sources)` includes `src/shared/`
      - `src/shared/*mainloop*`
  - `noinst_LTLIBRARIES += src/libshared-ell.la`
    - `src_libshared_ell_la_SOURCES`
      - `$(shared_sources)` includes `src/shared/`
      - `src/shared/*ell*`
- `Makefile.tools`
  - `bin_PROGRAMS += client/bluetoothctl`
    - `client_bluetoothctl_SOURCES` includes `client/`
    - `client_bluetoothctl_LDADD` includes
      - `lib/libbluetooth-internal.la`
      - `gdbus/libgdbus-internal.la`
      - `src/libshared-glib.la`
  - `bin_PROGRAMS += monitor/btmon`
    - `monitor_btmon_SOURCES` includes `monitor/`
    - `monitor_btmon_LDADD` includes
      - `lib/libbluetooth-internal.la`
      - `src/libshared-mainloop.la`
  - `bin_PROGRAMS += tools/rctest tools/l2test tools/l2ping tools/bluemoon \
                tools/hex2hcd tools/mpris-proxy tools/btattach tools/isotest`
    - `*_SOURCES` includes `tools/`
    - `*_LDADD` includes
      - `gdbus/libgdbus-internal.la`
      - `src/libshared-mainloop.la`
      - `lib/libbluetooth-internal.la`
- `Makefile.plugins`
  - `plugin_LTLIBRARIES += plugins/sixaxis.la`
  - `plugins_sixaxis_la_SOURCES = plugins/sixaxis.c`
- `Makefile.obexd`
  - `pkglibexec_PROGRAMS += obexd/src/obexd`
    - `obexd_src_obexd_SOURCES` includes
      - `$(btio_sources)` includes `btio/`
      - `$(gobex_sources)` includes `gobex/`
      - `$(obexd_builtin_sources)` includes
        - `obexd/plugins/`
        - `obexd/client/`
      - `obexd/`
    - `obexd_src_obexd_LDADD` includes
      - `lib/libbluetooth-internal.la`
      - `gdbus/libgdbus-internal.la`
      - `src/libshared-glib.la`
- `Makefile.mesh`
  - `pkglibexec_PROGRAMS += mesh/bluetooth-meshd`
    - `mesh_bluetooth_meshd_SOURCES` includes `mesh/`
    - `mesh_bluetooth_meshd_LDADD` includes
      - `ell/libell-internal.la`
      - `src/libshared-ell.la`

## BlueZ btio

- `bt_io_connect`
  - `parse_set_opts` parses the options into `struct set_opts`
    - `BT_IO_OPT_SOURCE_BDADDR` specifies the src addr
    - `BT_IO_OPT_DEST_BDADDR` specifies the dst addr
    - `BT_IO_OPT_CHANNEL` specifies the channel
      - `type` is set to `BT_IO_RFCOMM`
    - `BT_IO_OPT_PSM` specifies the psm
      - `type` is set to `BT_IO_L2CAP`
    - `BT_IO_OPT_CID` specifies the cid
      - `type` is set to `BT_IO_L2CAP`
    - `BT_IO_OPT_MODE` specifies the mode
      - `type` is set to `BT_IO_ISO` if the mode is `BT_IO_MODE_ISO`
  - `create_io` creates a client `GIOChannel`
    - if `type` is `BT_IO_L2CAP`
      - `socket(PF_BLUETOOTH, SOCK_SEQPACKET, BTPROTO_L2CAP)`
      - `l2cap_bind` binds the socket to `struct sockaddr_l2`
    - if `type` is `BT_IO_RFCOMM`
      - `socket(PF_BLUETOOTH, SOCK_STREAM, BTPROTO_RFCOMM)`
      - `rfcomm_bind` binds the socket to `struct sockaddr_rc`
    - if `type` is `BT_IO_SCO` (default)
      - `socket(PF_BLUETOOTH, SOCK_SEQPACKET, BTPROTO_SCO)`
      - `sco_bind` binds the socket to `struct sockaddr_sco`
    - if `type` is `BT_IO_ISO`
      - `socket(PF_BLUETOOTH, SOCK_SEQPACKET, BTPROTO_ISO)`
      - `iso_bind` binds the socket to `struct sockaddr_iso`
  - a connection is made
    - `l2cap_connect` connects to `struct sockaddr_l2`
    - `rfcomm_connect` connects to `struct sockaddr_rc`
    - `sco_connect` connects to `struct sockaddr_sco`
    - `iso_connect` connects to `struct sockaddr_iso`
  - `connect_add` adds a gio watch
    - it invokes the connect callback when `G_IO_OUT` is ready
- `bt_io_listen`
  - `parse_set_opts` is the same as in `bt_io_connect`
  - `create_io` creates a server `GIOChannel`
  - no connection is made
    - instead, `listen` is called
  - `server_add` adds a gio watch
    - it invokes the callbcks when `G_IO_IN` is ready and the connection is
      accepted

## BlueZ `bluetoothd`

- `/var/lib/bluetooth/nn:nn:nn:nn:nn:nn` contains info about a local
  controller
  - each device in the database is a subdirectory
    - with info about the device and whether it has been paired or not
- It has well-known name `org.bluez` on system bus
- For each HCI, an object on path `/org/bluez/<pid>/hciX` is created
- If an agent (`bluetooth-applet`) for an HCI is running, it will be `/org/bluez/agent/hciX`
- `main`
  - `connect_dbus` connects to the system bus and requests the well-known
    `org.bluez` name
  - `adapter_init` calls `mgmt_new_default`
    - a socket of bt protocol `BTPROTO_HCI` and of hci channel
      `HCI_CHANNEL_CONTROL` is created
    - `can_read_data` is called when there is incoming data
  - `btd_device_init` calls `btd_service_add_state_cb`
  - `btd_agent_init` registers `org.bluez.AgentManager1` for `/org/bluez`
  - `btd_profile_init` registers `org.bluez.ProfileManager1` for `/org/bluez`
  - `start_sdp_server`
    - `init_server` initializes the server
      - `l2cap_sock` is a listening socket of `BTPROTO_L2CAP`
      - the psm is `SDP_PSM`
      - `unix_sock` is set to -1 unless compat
    - `io_accept_event` is called when there is an incoming connection
      - it accepts the connection
      - `io_session_event` is called when there is incoming data on the
        connection
      - `handle_request` and `process_request` are called to handle the
        incoming requests
  - `plugin_init` adds all plugins on `__bluetooth_builtin` array and in
    `/usr/lib/bluetooth/plugins`
    - `BLUETOOTH_PLUGIN_DEFINE` is used to define a built-in plugin
      - it defines a global `struct bluetooth_plugin_desc __bluetooth_builtin_ ## name`
      - `src/genbuiltin` script generates `src/builtin.h` which defines the
        `__bluetooth_builtin` array
    - `add_plugin` adds a plugin to the `plugins` list
    - for each plugin, its `bluetooth_plugin_desc::init` is called
  - `rfkill_init` watches `/dev/rfkill`
    - on `G_IO_IN`, `rfkill_event` reads `struct rfkill_event` and checks if
      the type is `RFKILL_TYPE_BLUETOOTH` or `RFKILL_TYPE_ALL` and the op is
      `RFKILL_OP_CHANGE`
    - `get_adapter_id_for_rfkill` reads `/sys/class/rfkill/rfkill%u/name` to
      get the hci id
    - `adapter_find_by_id` returns the adapter with the given id
    - `btd_adapter_set_blocked` or `btd_adapter_restore_powered` to block or
      restre the adpter
  - `mainloop_run_with_signal` enters the mainloop

## Profiles

- my headset
  - it has these profiles
    - SPP (Serial Port Profile)
    - HSP (Headset Profile)
    - A2DP (Advanced Audio Distribution Profile) Sink
    - AVRCP (Audio/Video Remote Control Profile) Controller
    - AVRCP (Audio/Video Remote Control Profile) Target
    - HFP (Hands-Free Profile)
    - PnP Information
  - `pactl list cards` shows that pulseaudio talks to bluez directly, rather
    than talking to an ALSA card created by bluez
  - `lsmod` shows that uinput is loaded, which is used by bluez to create an
    input device
    - `AVRCP -> AVCTP -> uinput_create`
- my old gamepad
  - it has thse profiles
    - SDP (Service Discovery Protocol)
    - HIDP (Human Interface Device Profile)
    - PnP Information
  - `hidp` kernel module calls either `hid_allocate_device` to create a
    `hid_device`, or calls `input_allocate_device` to create a `input_dev`
    - this is decided by bluez depending what the device reports
    - in case of `hid_device`, a real `hid_driver` is needed to drive the
      `hid_device` and create `input_dev`s
  - for this gamepad, the `hid_driver` is `hid-generic`
- my new gamepad
  - it has thse profiles
    - HOGP (HID over GATT Profile)
  - bluez uses `uhid` to create `hid_device` from userspace and skips `hidp`

## Stack

- `RFCOMM`, `TCS`, and `SDP` are immediately above `L2CAP`
- `PPP`, `AT-COMMAND`, and `OBEX` are immediately above `RFCOMM`
- `SCO` and `ACL` are above baseband and below `Link Manager`
- `SCO` is for speech and `ACL` is for data.  That is, `L2CAP` is _NOT_ depends
  on `SCO`.

## Get Physical

- There are four physical channels
  - Two for communication between connected devices
  - One for discovering (inquiry scan channel)
  - One for connecting (page scan channel)
- Only one channel can be used at any given time
  - But it can switch between them
- All connected devices form a piconet
  - There is only one master in a piconet
  - There can be many slaves
- A physical link represents a baseband connection between devices.
  - is always associated with exactly one physical channel
- Logical transports
  - SCO
  - eSCO
  - ACL
  - ASB
  - PSB
- Each device has a BD_ADDR (bt device addr)
- Each slave active in a piconet is assigned a 3-bit LT_ADDR (logical transport
  addr)
- Logical links
  - LC, link control layer
  - ACL-C, link manager layer
  - ACL-U
  - SCO-S
  - eSCO-S
- `STANDBY`, `CONNECTION`, `PARK`
  - Vol2, PartB, Ch.8
  - Link Controller States
  - a inquirying device should enter `inquiry` state
  - a discoverable device should regularly enter `inquiry scan` state to respond
  - a paging device enters `page` state and a paged device enters `page scan`
    state.  The paging device is the master while the paged device is a slave.
    They will go on to enter `master response` and `slave response` states if
    everything goes well.  Finally, they will both enter `CONNECTION` state.

## `L2CAP`

- Stands for Logical Link Control and Adaption Protocol
- layered over link controller protocol 

## `l2ping`

- It opens a `socket(PF_BLUETOOTH, SOCK_RAW, BTPROTO_L2CAP)`
- It `connect`s to given remote
- It `send`s `L2CAP_ECHO_REQ`
- Note that a connection is made as can be seen from `hcitool conn`

## Connections

- `hcitool cc <bdaddr>` and `hcitool dc <bdaddr>`
- `hidd --connect <bdaddr>` and `hidd --unplug <bdaddr>`
- `rfcomm connect <dev> <bdaddr> <channel>` and `rfcomm release <dev>`
- the first one is in the link controller layer; the latters are much higher?

## `hidd --connect`

- It opens `ctl = socket(AF_BLUETOOTH, SOCK_RAW, BTPROTO_HIDP)`
- It opens a sdp over l2cap to get device info
- It makes several `sk = socket(PF_BLUETOOTH, SOCK_SEQPACKET, BTPROTO_L2CAP)` and
  connect to all of them.  They are HID control and interrupt channels.
- It optionally requests authentication and encryption.
- It `ioctl`s `HIDPCONNADD` with collected info.  An HID device is created by
  using the established connections/channels.

## `rfcomm connect`

- It opens `ctl = socket(AF_BLUETOOTH, SOCK_RAW, BTPROTO_RFCOMM)`
- It makes another `sk = socket(AF_BLUETOOTH, SOCK_STREAM, BTPROTO_RFCOMM)`,
  bind the remote address, and connect to it.
- After connection is established, `RFCOMMCREATEDEV` is called upon `sk` to
  create `/dev/rfcommX`.
- When disconnected, it simply leaves.  `ctl` is not used in normal situation.

## Debug

- `hcitool scan` to discover devices.  But devices can be configured undiscoverable.

    < HCI Command: Inquiry (0x01|0x0001) plen 5
    > HCI Event: Command Status (0x0f) plen 4
    > HCI Event: Inquiry Result with RSSI (0x22) plen 15
    ............
    > HCI Event: Inquiry Result with RSSI (0x22) plen 15
    > HCI Event: Inquiry Complete (0x01) plen 1
- `l2ping` to ping devices.  But devices can be configured not to echo back.

    < HCI Command: Create Connection (0x01|0x0005) plen 13
    > HCI Event: Command Status (0x0f) plen 4
    > HCI Event: Connect Complete (0x03) plen 11
    < HCI Command: Read Remote Supported Features (0x01|0x001b) plen 2
    > HCI Event: Max Slots Change (0x1b) plen 3
    > HCI Event: Command Status (0x0f) plen 4
    > HCI Event: Command Status (0x0f) plen 4
    < HCI Command: Remote Name Request (0x01|0x0019) plen 10
    > HCI Event: Read Remote Supported Features (0x0b) plen 11
    < ACL data: handle 42 flags 0x02 dlen 52
        L2CAP(s): Echo req: dlen 44
    > HCI Event: Command Status (0x0f) plen 4
    > ACL data: handle 42 flags 0x02 dlen 52
        L2CAP(s): Echo rsp: dlen 44
    > HCI Event: Remote Name Req Complete (0x07) plen 255
    > HCI Event: Number of Completed Packets (0x13) plen 5
    < ACL data: handle 42 flags 0x02 dlen 52
        L2CAP(s): Echo req: dlen 44
    > HCI Event: Number of Completed Packets (0x13) plen 5
    > ACL data: handle 42 flags 0x02 dlen 52
        L2CAP(s): Echo rsp: dlen 44
- E.g., a bt mouse might hide itself and stop echoing back when a hidd
  connection is made.
- `hcitoo cc` to a bt mouse (it is rejected, i guess)

    < HCI Command: Create Connection (0x01|0x0005) plen 13
    > HCI Event: Command Status (0x0f) plen 4
    > HCI Event: Connect Complete (0x03) plen 11
    < HCI Command: Read Remote Supported Features (0x01|0x001b) plen 2
    > HCI Event: Command Status (0x0f) plen 4
    < HCI Command: Remote Name Request (0x01|0x0019) plen 10
    > HCI Event: Page Scan Repetition Mode Change (0x20) plen 7
    > HCI Event: Command Status (0x0f) plen 4
    > HCI Event: Read Remote Supported Features (0x0b) plen 11
    > HCI Event: Remote Name Req Complete (0x07) plen 255
    > HCI Event: QoS Setup Complete (0x0d) plen 21
    < HCI Command: Disconnect (0x01|0x0006) plen 3
    > HCI Event: Command Status (0x0f) plen 4
    > HCI Event: Disconn Complete (0x05) plen 4
- `hidd --connect` to a bt mouse

    < HCI Command: Create Connection (0x01|0x0005) plen 13
    > HCI Event: Command Status (0x0f) plen 4
    > HCI Event: Connect Complete (0x03) plen 11
    < HCI Command: Read Remote Supported Features (0x01|0x001b) plen 2
    > HCI Event: Command Status (0x0f) plen 4
    < HCI Command: Remote Name Request (0x01|0x0019) plen 10
    > HCI Event: Page Scan Repetition Mode Change (0x20) plen 7
    > HCI Event: Command Status (0x0f) plen 4
    > HCI Event: Read Remote Supported Features (0x0b) plen 11
    < ACL data: handle 42 flags 0x02 dlen 10
        L2CAP(s): Info req: type 2
    > HCI Event: Number of Completed Packets (0x13) plen 5
    > ACL data: handle 42 flags 0x02 dlen 16
        L2CAP(s): Info rsp: type 2 result 0
          Extended feature mask 0x0000
    < ACL data: handle 42 flags 0x02 dlen 12
        L2CAP(s): Connect req: psm 1 scid 0x0040
    > HCI Event: Remote Name Req Complete (0x07) plen 255
    > HCI Event: Number of Completed Packets (0x13) plen 5
    > ACL data: handle 42 flags 0x02 dlen 16
        L2CAP(s): Connect rsp: dcid 0x005f scid 0x0040 result 1 status 2
          Connection pending - Authorization pending
    > HCI Event: QoS Setup Complete (0x0d) plen 21
    > ACL data: handle 42 flags 0x02 dlen 16
        L2CAP(s): Connect rsp: dcid 0x005f scid 0x0040 result 0 status 0
          Connection successful
    < ACL data: handle 42 flags 0x02 dlen 12
        L2CAP(s): Config req: dcid 0x005f flags 0x00 clen 0
    > HCI Event: Number of Completed Packets (0x13) plen 5
    > ACL data: handle 42 flags 0x02 dlen 14
        L2CAP(s): Config rsp: scid 0x0040 flags 0x00 result 0 clen 0
          Success
    > ACL data: handle 42 flags 0x02 dlen 16
        L2CAP(s): Config req: dcid 0x0040 flags 0x00 clen 4
          MTU 48


## Plugin, using input plugin as an example

- its `init` is called by `bluetoothd` to enable a plugin
- it calls `btd_register_adapter_driver` to be notified the come and leave of
  adapters
  - when an adapter comes, it calls `bt_io_listen` twice to listen on the
    adapter's control and interrupt channels.
  - for device initiated reconnection?
- it calls `btd_register_device_driver` twice to be notified the come and leave
  of devices supporting `HID_UUID` and `HSP_HS_UUID`
  - `struct btd_device_driver` has a memeber `uuids` to give uuids the driver
    suports.
  - every `struct btd_device` has a memeber `uuids` to give a list of supported
    uuids.
  - device and driver matching is done by `device_probe_drivers`.
  - Anyway, when a device supporting `HID_UUID` is found, the device is matched
    by input plugin.  It calls its `input_device_register` to put the device on
    the dbus with interface `org.bluez.Input`.
  - The interface has a `Connect` method.  When the method is called, a
    connection is made and `hidp_add_connection` is called to create a kernel
    input device.
- section 5.4.5 of HID spec for connection handling rules
  - Both `HID_Control` and `HID_Interrupt` channels need to be established that
    a HID connection is considered established.
  - A HID device might reconnect in the event of host reset
  - virtual cable: a HID device should exit page scan/page, inquiry scan/inquiry
    modes in the duration of a connection.  This makes sure that no second
    host can talk to the HID device.
- when a hid connection is added, a `struct hidp_session` is created in kernel
  using the given ctrl and intr connections.  `hidp_setup_input` is called to
  create a `struct input_dev`.  A kernel thread is created to run
  `hidp_session`.  Received ctrl frames and intr frames are handled by
  `hidp_recv_ctrl_frame` and `hidp_recv_intr_frame`.  They report input events
  using the standard mechanism.

## Connection

- Vol2, partC, Link Manager Protocol
- Bottom up: RF, LC, and LM
- After paging procedure, LC is in `CONNECT` state.  LMP procedure follows.
- Authentication involves verifier sending `LMP_au_rand` and claimant
  calculating a response based on the challenge, its `BD_ADDR` and a shared
  secret key.  Claimant sends `LMP_sres` if it has a link key.  If it doesn't,
  it sends `LMP_not_accepted` with error code `key missing`.
- Pairing or Secure Simple Pairing is used to create a initialization key
  `Kinit` if there is shared no link key.  When both devices have calculated
  `Kinit`, the link key is created and mutual auth is performed.
- Pairing starts with initiator sending `LMP_in_rand` and the repsonder replying
  `LMP_accepted`.  They then go on to calculate `Kinit` based on the `BD_ADDR`
  of the responder, and continue to create the link key.  This is the case the
  responder has a variable PIN.
- If the responder has a fixed PIN, it should send another `LMP_in_rand` to
  initiator and the initiator should `LMP_accepted` or `LMP_not_accepted`
  depending on whether it has a variable or fixed PIN.
- Both devices go on to create the link key through `LMP_comb_key` or
  `LMP_unit_key`.  The latter is deprecated though.
- After mutual auth, if encryption is enabled, it should start producing a
  encryption key from the link key.
- link, channel, and connection establishment
  - Vol2, partC, chapter7
  - link establishment is to establish a physical link between two devices.  It
    involves paging, authentication, encryption negotiation, etc.
  - channel establishment is to establish a L2CAP channel.  It is built above
    established link, and involves authentication and encryption negotiation.
  - connection establishment is to establish a connection.  It is built above
    established channel, and involves authentication and encryption negotiation.
- pairing and bonding

    The purpose of bonding is to create a relation between two Bluetooth devices
    based on a common link key (a bond). The link key is created and exchanged
    (pairing) during the bonding procedure and is expected to be stored by both
    Bluetooth devices, to be used for future authentication. In addition to
    pairing, the bonding procedure can involve higher layer initialization
    procedures.

## org.bluez.Headset

- It is provided by audio plugin.
  - In the same directory, an alsa module and a gstreamer module can be found
  - The alsa module allows
    - talking to bluez audio plugin through `\0/org/bluez/audio`.
    - it asks bluez to connect to a device.
    - it receives SCO socket from bluez (`SCM_RIGHTS`).
    - pcm data are written to SCO.
    - the audio path is audio player, alsa bt module, bt driver.  Totally cpu.
      - This is called SCO over HCI/UART.
  - for gsm, ideally, we want gsm, audio chipset, bt chipset.  No cpu
    intervention.
    - This is called SCO over PCM.
- In `audio_manager_init`,
  - Adapter driver `headset_server_driver` is registered to listen on HSP and
    HFP RFCOMM channels.
  - Device driver `audio_driver` is registered to add `org.bluez.Audio`
    interface to audio devices.  Devices are also probed and more interfaces
    like `org.bluez.Headset` are added.
- `Connect` causes `hs_connect` to be called.
  - The state must be `disconnected`.
  - It calls `rfcomm_connect` to establish ACL connection
    - The state is changed to `connecting`.
  - When ACL connection is made, `headset_connect_cb` is called.  It calls
    `sco_connect` to establish SCO connection.
    - The state is changed to `connected`.
  - When SCO connection is made, `sco_connect_cb` is called.
    - The state is changed to `playing`.

## HCI over UART

- hciattach opens the serial port
  - Makes it raw and sets the baud rate
  - Calls `uart_t->init` if exists
  - Changes ldisc and proto
    - `TIOCSETD` to use `N_HCI` ldisc.
    - `HCIUARTSETPROTO` to specify protocol (H4, BCSP, LL, etc.)
  - Calls `uart_t->post` if exists
- The `N_HCI` ldisc
  - does not allow read/write from the tty.
- When a proto is set through `HCIUARTSETPROTO`
  - the proto is `open`ed.
  - `hci_uart_register_dev` is called to register a hci device.
- When there are outgoing data, `hci_send_frame` is called
  - In uart case, `hci_uart_send_frame` is called.
  - `skb` is always enqueued to proto first.
  - They are dequeued from proto and written to the serial port.
- When there are incoming data, ldisc `receive_buf` is called.
  - It calls proto's `recv`, like `ll_recv`.
  - The buffer is parsed to form `skb`s.
  - `hci_recv_frame` is called to receive the `skb`s.

## HCI packet formats

- There are 4 types of packets
  - 0x01 for command
  - 0x02 for ACL
  - 0x03 for SCO
  - 0x04 for event
- All values are in little-endian
- A command packet
  - begins with 16bits opcode.
    - OGF: Opcode Group Field, higher 6bits
      - 0x01 for Link Control Command
      - 0x02 for Link Policy Command
      - 0x03 for Controller & Baseband Command
      - 0x04 for Informational Parameters
      - 0x05 for Status Parameters
      - 0x06 for Testing Command
      - ...
    - OCF: Opcode Command Field, lower 10bits
    - For example, `HCI_Reset` is OGF 0x03 and OCF 0x03.  It is `0x03 0x0c` in
      memory.
  - opcode is followed by 8bits parameter length
  - length is followed by parameters, 8bits each
- An event packet
  - begins with 8bits event code
  - followed by 8bits parameter total length
  - folowed by N parameters, 8bits for each
