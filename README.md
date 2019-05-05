# PHP client library for XMPP (Jabber) protocol

[![Latest Stable Version](https://poser.pugx.org/norgul/xmpp-php/v/stable)](https://packagist.org/packages/norgul/xmpp-php)
[![Total Downloads](https://poser.pugx.org/norgul/xmpp-php/downloads)](https://packagist.org/packages/norgul/xmpp-php)
[![Latest Unstable Version](https://poser.pugx.org/norgul/xmpp-php/v/unstable)](https://packagist.org/packages/norgul/xmpp-php)
[![License](https://poser.pugx.org/norgul/xmpp-php/license)](https://packagist.org/packages/norgul/xmpp-php)

This is low level socket implementation for enabling PHP to 
communicate with XMPP due to lack of such libraries online (at least ones I 
could find that had decent documentation). 

XMPP core documentation can be found [here](https://xmpp.org/rfcs/rfc6120.html).

# Installation requirements and example

Project requirements are given in `composer.json` (
[Composer website](https://getcomposer.org)):

You can use this library in your project by running:

```
composer require norgul/xmpp-php
```

You can see usage example in `Example.php` file by changing credentials to 
point to your XMPP server and from project root run `php Example.php`.

# Library usage
## Brief introduction and v2.0 changelog
Version 2.0 onward is done with major refactoring in readability department. I was
trying to get all methods more readable and understandable to the first time code reader.

`XmppClient.php` class has been stripped out of unnecessary functionalities as I 
imagined the class being only a user friendly interface for interacting with XMPP server
without the need to know the logic behind it.

Since XMPP is all about 3 major stanzas, (**IQ, Message and Presence**), I've 
created separate classes which are dependant on socket implementation so that you 
can directly send XML by calling a stanza method. 

Also what was before a: `$client->sendMessage()` is now a `$client->message->send()`. 
The major change is that another layer is added to the equation so that `XmppClient` is 
basically a stanza wrapper with only client relevant functions left inside the class
(i.e. `connect()` and `disconnect()`).

Current logic thus is `$client->STANZA->METHOD()`.

I would appreciate all the feedback I can get as I most probably broke something in the
process :). 

## Init
In order to start using the library you first need to instantiate a new `Options` 
class. Host, username and password are mandatory fields, while port number, if omitted,
will default to `5222` which is XMPP default. 

Username can be either bare `JID` or in `JID/resource` form. If you are using a bare `JID`
the resource will be added automatically. You can override this by explicitly setting a 
resource with `$client->iq->setResource()`. In the second case the username will be automatically
parsed to `username` and `resource` variables. In case of `JID/resource/xyz` format, everything
after second slash will be ignored. If both `JID/resource` is present as well as using the
`$client->iq->setResource()` method, second one will take precedence. 

```
$options = new Options();

$options
    ->setHost($host)            // required
    ->setPort($port)            // not required, defaults to 5222
    ->setUsername($username)    // required
    ->setPassword($password);   // required
```

`Options` object is required for establishing the connection and every other subsequent
request, so once set it should not be changed. 

Once this is set you can instantiate a new `XmppClient` object and pass the `Options` object in.

## Connecting to server
```
$client = new XmppClient();
$client->connect($options);
```

`$client->connect()` method does a few things:
1. Connects to the socket which was initialized in `XmppClient` constructor
2. Opens an XML stream to exchange with the XMPP server
3. Tries to authenticate with the server based on provided credentials
4. Starts the initial communication with the server, bare minimum to get you started

Current version supports `PLAIN` and `DIGEST-MD5` auth methods. 

## TLS

TLS is now supported by default. If server has support for TLS, library will 
automatically try to connect with TLS and make the connection secure. 

If you'd like to explicitly disable this functionality, you can use `setUseTls(false)` 
function on the `Options` instance so that TLS communication is disabled. 

## Sending messages

`$client->message->send()` takes 3 parameters of which the last one is optional. First parameter
is the actual message you'd like to send, second one is recipient of the message and third
one is type of message to be sent. This defaults to `chat`.

## Receiving messages and other responses

Server responses (or server side continuous XML session to be exact) can be retrieved with 
`$client->getResponse()`.

~~This method also takes an optional boolean parameter `echoOutput` which is a 
flag indicating whether the response should be echoed out. This is useful for testing from
terminal so when you set the flag to `true` you will be able to actually see the response
which will be returned. For all other purposes this flag should be set to `false`.~~

If you would like to see the output of the received response in the console you can call the
`$client->prettyPrint($response)` method. 

In case you are not interested in complete response which comes from server, you may also use 
`$client->getMessages()` which will match message tags with regex and return array of matched 
messages. In case you'd like to see the response in the terminal, you can do something like this:

```
do {
    $response = $client->getMessages();
    if($response)
        echo print_r($response);
} while (true);
```

## Roster

`$client->iq->getRoster()` takes no arguments and fetches current authenticated user roster. 

## Priority

`$client->presence->setPriority()` sets priority for given resource. First argument is an integer 
`-128 <> 127`. If no second argument is given, priority will be set for currently used resource.
Other resource can be provided as a second argument whereas the priority will be set for that
specific resource. 

## Presence

`$client->presence->subscribe()` takes JID as an argument and asks that user for presence.

`$client->presence->acceptSubscription()` takes JID as an argument and accepts presence from that user.

`$client->presence->declineSubscription()` takes JID as an argument and declines presence from that user.

## Group

`$client->iq->setGroup()` puts a given user in group you provide. Method takes two arguments: 
first one being the group name which you will attach to given user, and other 
being JID of that user. 

## More options (not required)

`Options` object can take more options which may be chained but are not required. These are explained
and commented in the code directly in the `Options` class:

```
$options
    ->setProtocol($protocol)  // defaults to TCP
    ->setResource($resource)  // defaults to 'norgul_machine' string + timestamp
    ->setLogger($logger)      // PSR-4 logger instance (logging explained below)
    ->setAuthType($authType)  // Takes on classes which implement Authenticable
```

## Socket options
Most of the socket options are set by default so there is no need to temper
with this class, however you can additionally change the timeout for the period 
the socket will be alive when doing a `socket_read()`, and you can do that with
`$socket->setTimeout()`.

## Logging

Upon new established session the library is creating 3 files in `logs/` folder:

- `request.xml` -> containing XML's sent to the server
- `response.xml` -> containing XML's received from the server
- `xmpp.log` -> will contain all other logs (currently empty)

In order to circumvent this,  you can manually set logger when instantiating 
`Options` with `setLogger($logger)` where you can put any of the loggers from
 `Loggers/` folder (defaults to `SimpleLogger`).
  
- `SimpleLogger` - creates only 3 files and overwrites them upon each established 
session. While this is good for testing a single session, concurrent ones will 
render those files unusable as they will get merged request/response from multiple 
sessions. 
- `FullLogger` - creates 3 files on each established session. These files are 
prefixed by time in `YMD` format, plus time and unique ID. It may be cumbersome 
to have 3 files created on each session, but it is a great way to easily see 
the communication between server and client this way and debug if something's 
wrong.

- `NoLogger` - doesn't log anything.

# Other

`Example.php` has a `sendRawXML()` method which can be helpful with debugging. Method works in a way
that you can provide hand-written XML and send it to the server. On the other hand you can 
also trigger a method by providing method name instead of XML. 

```
Enter XML: <xml>foo</xml>           <-- will send XML
Enter XML: getRoster                <-- will run getRoster() method
Enter XML: requestPresence x@x.com  <-- will run with argument requestPresence(x@x.com)
```

Some valid XMPP XML will be declined (like sending `<presence/>`) because `simplexml_load_string()` 
is not able to parse it as being a valid XML. In cases you need to do some custom stuff like 
that and you are sure it is a XMPP valid XML, you can remove the parsing line and just let the
`send()` method do its magic.
 
 **Be aware! Be very aware!** sending an invalid XML to the server
will probably invalidate currently open XML session and you will probably need to restart the 
script.
