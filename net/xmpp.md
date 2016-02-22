core xmpp-connection
* read: block, partial some data
* write: block, partial some data
* poll before read/write to avoid blocking
* xmpp_connection_get_fd
* xmpp_connection_write
* xmpp_connection_read
* link to tls and sasl

core xmpp-stream
* XMPP XML stream
* control xmpp-connection for XMPP STARTTLS, USESASL
? resource binding
* I/O in stanza
* GtkTreeStore-like tiny XML constructor
* link to expat

core xmpp-policy
? enforce c2s or s2c policy

im session
* rosource binding -> connected resource
* <session/> -> active resource
* <presense/> -> available resource
* ???


<message type="chat|error|groupchat|headline|normal">
<subject/>
<message/>
<thread/>
</message>

<presence type="unavailable|subscribe|subscribed|unscribe|unscribed|probe|error">
<show/>
<status/>
<priority/>
</presence>

As specified in XMPP Core [1] and XMPP IM [2], an IQ stanza of type "get" sent
to a bare JID <localpart@domain.tld> is handled by the user's server on the
user's behalf, not delivered to one or more connected or available resources.

* node@domain/resource -> a connected resource, handled by client
* node@domain -> the data on the server, handled by the server
* domain -> the server
* e.g., iq sent to node@domain is handled by the server
* what if a resource got a unsupported iq, e.g. <query xmlns='jabber:iq:last'/>?
* return service-unavailable?

* Service Discovery
* Data form: <x/> SHOULD be direct child of <message/> or second-level child of <iq/>
* In order for an application to determine whether an entity supports this
  protocol, where possible it SHOULD use the dynamic, presence-based profile of
  service discovery defined in Entity Capabilities [4]. However, if an
  application has not received entity capabilities information from an entity,
  it SHOULD use explicit service discovery instead.
* <message 
    from='bernardo@shakespeare.lit/pda'
    to='francisco@shakespeare.lit/elsinore'
    type='chat'>
  <composing xmlns='http://jabber.org/protocol/chatstates'/>
</message>

