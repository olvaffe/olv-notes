A connection described a set of (user) settings

A manager, which manages network devices
device is typed, wired, wireless, gsm, ...
When a manager created, it calls initial_get_connections to get the settings

nm-applet then calls nm_manager_activate_device(connection, device) to relate
connection to device and activate the device


MANAGER STATES:
typedef enum NMState
{
	NM_STATE_UNKNOWN = 0,
	NM_STATE_ASLEEP, /* deactivate and bring down all devices */
	NM_STATE_CONNECTING,
	NM_STATE_CONNECTED, /* if any device is activated */
	NM_STATE_DISCONNECTED
} NMState;

typedef enum
{
	NM_DEVICE_STATE_UNKNOWN = 0,
	NM_DEVICE_STATE_DOWN,
	NM_DEVICE_STATE_DISCONNECTED,

	/* 5 stages of activation */
	NM_DEVICE_STATE_PREPARE,
	NM_DEVICE_STATE_CONFIG,
	NM_DEVICE_STATE_NEED_AUTH,
	NM_DEVICE_STATE_IP_CONFIG,
	NM_DEVICE_STATE_ACTIVATED,

	NM_DEVICE_STATE_FAILED,
	NM_DEVICE_STATE_CANCELLED,
} NMDeviceState;

static gboolean nm_dbus_signal_device_status_change (gpointer user_data)
	if (!(message = dbus_message_new_signal (NM_DBUS_PATH, NM_DBUS_INTERFACE, sig)))
	{ DEVICE_NO_LONGER_ACTIVE, 	"DeviceNoLongerActive"	},
	{ DEVICE_NOW_ACTIVE,		"DeviceNowActive"		},
	{ DEVICE_ACTIVATING,		"DeviceActivating"		},
	{ DEVICE_ACTIVATION_FAILED,	"DeviceActivationFailed"	},
	{ DEVICE_ADDED,			"DeviceAdded"			},
	{ DEVICE_REMOVED,			"DeviceRemoved"		},
	{ DEVICE_CARRIER_ON,		"DeviceCarrierOn"		},
	{ DEVICE_CARRIER_OFF,		"DeviceCarrierOff"		},
	{ DEVICE_STATUS_INVALID,		NULL					}

void nm_dbus_signal_state_change (DBusConnection *connection, NMData *data)
	if (!(message = dbus_message_new_signal (NM_DBUS_PATH, NM_DBUS_INTERFACE, NM_DBUS_SIGNAL_STATE_CHANGE)))

void nm_dbus_signal_wireless_network_change (DBusConnection *connection, NMDevice80211Wireless *dev, NMAccessPoint *ap, NMNetworkStatus status, gint strength)
			sig = "WirelessNetworkDisappeared";
			sig = "WirelessNetworkAppeared";
			sig = "WirelessNetworkStrengthChanged";
	if (!(message = dbus_message_new_signal (NM_DBUS_PATH, NM_DBUS_INTERFACE, sig)))

void nm_dbus_signal_device_strength_change (DBusConnection *connection, NMDevice80211Wireless *dev, gint strength)
	if (!(message = dbus_message_new_signal (NM_DBUS_PATH, NM_DBUS_INTERFACE, "DeviceStrengthChanged")))

void nm_act_request_set_stage (NMActRequest *req, NMActStage stage)
	if (!(message = dbus_message_new_signal (NM_DBUS_PATH, NM_DBUS_INTERFACE, "DeviceActivationStage")))

{
	NMDbusMethodList	*list = nm_dbus_method_list_new (NULL);

	nm_dbus_method_list_add_method (list, "getDevices",			nm_dbus_nm_get_devices);
	nm_dbus_method_list_add_method (list, "getDialup",			nm_dbus_nm_get_dialup);
	nm_dbus_method_list_add_method (list, "activateDialup",		nm_dbus_nm_activate_dialup);
	nm_dbus_method_list_add_method (list, "deactivateDialup",		nm_dbus_nm_deactivate_dialup);
	nm_dbus_method_list_add_method (list, "setActiveDevice",		nm_dbus_nm_set_active_device);
	nm_dbus_method_list_add_method (list, "createWirelessNetwork",	nm_dbus_nm_create_wireless_network);
	nm_dbus_method_list_add_method (list, "setWirelessEnabled",		nm_dbus_nm_set_wireless_enabled);
	nm_dbus_method_list_add_method (list, "getWirelessEnabled",		nm_dbus_nm_get_wireless_enabled);
	nm_dbus_method_list_add_method (list, "sleep",				nm_dbus_nm_sleep);
	nm_dbus_method_list_add_method (list, "wake",				nm_dbus_nm_wake);
	nm_dbus_method_list_add_method (list, "state",				nm_dbus_nm_get_state);
	nm_dbus_method_list_add_method (list, "createTestDevice",		nm_dbus_nm_create_test_device);
	nm_dbus_method_list_add_method (list, "removeTestDevice",		nm_dbus_nm_remove_test_device);

	return (list);
}
