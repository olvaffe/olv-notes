adapter, algorithm, i2c bus, client, and driver

Let's look adatpers and algorithms first.  Adapters are devices and algorithms
are their drivers.  Adapters belongs to platform bus and of class i2c-adapter.

There is also a single i2c bus.  Clients and drivers belong to i2c bus.
Clients are like devices and drivers are drivers.

That is from the view of device model.

Besides that view, when clients are registered (i2c_attach_client), the
corespondding adapter is informed.  When a new adapter is registered
(i2c_add_adapter), each driver is given a chance to scan the adapter.  When a
driver is added, it is given a chance to scan all adapters.

Take saa7134 for example.  It is a PCI device and its registers are controlled
through i2c.

saa7134-i2c.c registers an adapter and algorithm (well, and an internal
client).  ir-kbd-i2c.c registers the driver and the client (device).

It should be clear now.
