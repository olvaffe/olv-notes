UPnP Protocol
-------------

Addressing
* every device must support DHCP
* if no DHCP server, device must assign itself an address
* if it obtains a domain name during DHCP, the name is used
* otherwise, ip is used to identify the device

Discovery
* a UPnP device advertises its services once it obtains an address
* a UPnP control point discovers services in the network
* SSDP is used

Description
* control point could ask device for its description
* the URL is as obtained from discovery stage
* description is in XML
* consists of a list of embedded devices or services,
* a list of actions
* a list of variables, reflecting service states

Control
* having retrieved the description, control points could send actions to devices
* control messages are in SOAP (successor to XML-RPC)

Event notification
* on variable changes, device could send event notifications
* control points should subscribe to interested variables
* event notifications are in XML, over GENA
* GENA: [subscriber] <- [subscription arbiters] <- [notifying resource]
* GENA: HTTP over TCP/IP and multicast UDP

Presentation
* device could provide a presentation URL for use by browser


UPnP AV standards (DLNA)
-----------------

Components
* UPnP MediaServer DCP
* UPnP MediaServer ControlPoint
* UPnP MediaRenderer DCP
* UPnP RenderingControl DCP
* UPnP Remote User Interface (RUI) client/server 
* QoS (Quality of Service)


Alternatives
------------
* DPWS by MS
* NAT-PMP by Apple
