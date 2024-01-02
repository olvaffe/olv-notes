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
    - it invokes one of the callbcks when `G_IO_IN` is ready and the
      connection is accepted
    - there is no real difference between the `confirm` and the `connect`
      callbacks
- `bt_io_accept`
  - this is normally called from the `confirm` callback of `bt_io_listen`
    - in the confirm callback, `btd_request_authorization` starts auth and the
      auth callback normally calls `bt_io_accept`
  - it reads 1 byte if necessary (a workaround?)
  - it invokes the connect callback when `G_IO_OUT` is ready
- `bt_io_set` sets socket options
- `bt_io_get` gets socket options

## BlueZ `bluetoothd`

- `/var/lib/bluetooth/nn:nn:nn:nn:nn:nn` contains info about a local
  controller
  - each device in the database is a subdirectory
    - with info about the device and whether it has been paired or not
- `main`
  - `connect_dbus` connects to the system bus and requests the well-known
    `org.bluez` name
  - `adapter_init` initializes adapters
    - `mgmt_new_default` creates a socket of bt protocol `BTPROTO_HCI` and of
      hci channel `HCI_CHANNEL_CONTROL`
      - `can_read_data` is called when there is incoming data
    - `mgmt_send(MGMT_OP_READ_VERSION)` to start initialization
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
- adapter enumeration
  - `adapter_init` sends `MGMT_OP_READ_VERSION`
  - `read_version_complete` parses `mgmt_rp_read_version` and...
    - sends `MGMT_OP_READ_COMMANDS` to get supported commands and events
    - registers `MGMT_EV_INDEX_ADDED` and `MGMT_EV_INDEX_REMOVED` for adapter
      addition and removal notifications
    - sends `MGMT_OP_READ_INDEX_LIST` to enumerate adapters
  - `read_index_list_complete` parses `mgmt_rp_read_index_list` and calls
    `index_added` for each adapters
  - `index_added` calls `btd_adapter_new` to create a new adapter and calls
    `read_info` to send `MGMT_OP_READ_INFO`
  - `read_info_complete` parses `mgmt_rp_read_info` and does a lot of adapter
    initialization
    - e.g. it registers `connected_callback` for `MGMT_EV_DEVICE_CONNECTED`
  - `adapter_register` registers the new adapter
    - it creates `/org/bluez/hci%d` with `org.bluez.Adapter1` interface
    - `load_drivers` calls `btd_adapter_driver::probe` for each driver
      - e.g., `hostname` plugin calls `btd_register_adapter_driver` to
        register `hostname_driver`
      - `hostname_probe` calls `adapter_set_name` to update the name of the
        adapter
    - `probe_profile` calls `btd_profile::adapter_probe` on the adapter and
      updates `adapter->profiles` if the profile supports the adapter
      - e.g., the `input` plugin calls `btd_profile_register` to register
        `input_profile`
      - `hid_server_probe` starts an input server on
    - `load_devices` loads all devices from the database
      - `create_filename` prefixes `/var/lib/bluetooth`
      - `device_create_from_storage` creates the device
        - it creates `/org/bluez/hci%d/dev_%s` with `org.bluez.Device1`
          interface
      - `adapter_add_device` adds the device to `adapter->devices`
        - it also calls `btd_adapter_driver::device_added` for each driver
      - `probe_devices` calls `device_probe_profiles` and
        `device_resolved_drivers` on all added devices
        - `dev_probe` calls `probe_service` which calls
          `btd_profile::device_probe`
          - e.g., it points to `input_device_register` for the input profile 
    - `load_connections` sends `MGMT_OP_GET_CONNECTIONS` to get connected
      devices
      - for each connection, it looks up or creates a device of the addr
      - `adapter_add_connection` calls `device_add_connection` and adds the
        device to `adapter->connections`
    - it does a lot more stuff
- device connection
  - `dev_connect` of `device_methods` initiates the connection
    - `create_pending_list` adds all services to `dev->pending`
    - `connect_next` calls `btd_service_connect` on the next pending service
    - `btd_service_connect` calls `btd_profile::connect`
  - the dongle initiates the connection
    - take `input` plugin for example
    - the hid dongle is connected
      - `can_read_data` receives `MGMT_EV_DEVICE_CONNECTED`
      - `connected_callback` calls `adapter_add_connection` to manage the
        connection
    - the hid dongle sends HID events
      - btio accepts the ctrl channel and calls `connect_event_cb`
      - btio accepts the intr channel and calls `confirm_event_cb`
        - this also leads to `connect_event_cb` after auth
    - `confirm_event_cb` calls `btd_request_authorization` to auth
      - `adapter_authorize` adds a `service_auth` to `adapter->auths`
      - `process_auth_queue` authorizes the connection
        - if `btd_device_is_trusted` returns true, it is authorized
        - otherwise, `agent_get` and `agent_authorize_service` authorize the
          connection
      - `auth_callback` accepts the connection and calls `connect_event_cb`
    - `connect_event_cb` calls `input_device_set_channel`
      - `input_device_set_channel` calls `input_device_connadd` after both
        ctrl and intr channels are set up
      - `input_device_connadd` calls `input_device_connected`
      - `input_device_connected` calls `hidp_add_connection` which calls
        `ioctl_connadd` to make `HIDPCONNADD` ioctl to create the hid device

## BlueZ `bluetoothctl`

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
- `main`
  - it connects to the system bus
  - it creates a client for `/org/bluez` of `org.bluez`
  - it registers proxy handlers
    - `proxy_added`
    - `proxy_removed`
    - `property_changed`
- `cmd_agent` calls `agent_register`
  - it registers `/org/bluez/agent` with `org.bluez.Agent1`
  - it invokes `org.bluez.AgentManager1.RegisterAgent` with `/org/bluez/agent`
    - this calls `register_agent` in `bluetoothd`
  - this provides a cmdline-based auth agent
- `cmd_default_agent` calls `agent_default`
  - it invokes `org.bluez.AgentManager1.RequestDefaultAgent`
    - this calls `request_default` in `bluetoothd`
    - it moves the agent to the head of `default_agents` queue, which will be
      returned by `agent_get(NULL)`
- `cmd_connect`
  - `find_proxy_by_address` returns the proxy for the device
  - it invokes `org.bluez.Device1.Connect` on the device
    - this calls `dev_connect` in `bluetoothd`

## BlueZ Plugin, using input plugin as an example

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

## BlueZ Debug

- `btmon` to sniff traffic
- `l2ping` to ping devices.  But devices can be configured not to echo back.
  - It opens a `socket(PF_BLUETOOTH, SOCK_RAW, BTPROTO_L2CAP)`
  - It `connect`s to given remote
  - It `send`s `L2CAP_ECHO_REQ`

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

- `L2CAP` stands for Logical Link Control and Adaption Protocol
  - layered over link controller protocol 
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
