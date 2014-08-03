---
layout: post
title: Implementing an IOS Push Notification client with Golang
---
So you want to send push notifications to your IOS app. Starting out the 
whole process can seem a bit daunting. The APNS service from Apple has a lot
of little quirks that can make the process a bit difficult to grasp. Here 
we\'ll walk through the basics of connecting to Apple\'s gateway service and
how to correctly send push notifications. **Disclaimer** For the 
most part, you probably should use a third party service like 
[Urban Airship](http://urbanairship.com) if you\'re going to be sending a lot
of push notifications. There are good reasons to implement this on your own, 
those being cost, control, performance, etc... but they do a great job of 
minimizing risk to you and your infrastructure and chances are it may 
actually be cheaper.

 ##Overview
 Apple\'s push notification service (APNS) is setup for high throughput 
 communication and this is clearly evident in the way you communicate with
 their servers (a binary format over a socket). In contrast, Google and 
 Microsoft do their push notifications using HTTP. While simpler, HTTP is
 much slower. 

 The basic setup goes like this:
 * Create a TLS socket to Apple using your push keys
 * To send a push, write payload information in binary to socket
 * If an error occurs Apple will write information to socket about the error
    * **Note** any pushes sent after the error occured will need to be resent

####Note on error handling
Due to this format being about throughput first, error handling becomes a bit 
more complicated. Essentially Apple doesn\'t tell you that a push notification
was sent successfully, they only tell you that a push notification wasn\'t 
sent successfully and due to the fact that you don\'t get an immediate
response you may (probably) will have to replay some of the messages that were
sent after the message that caused the error. This can be tricky to 
synchronize but golang channels help out a lot here.

##go-libapns
[go-libapns](http://joekarl.github.io/go-libapns/) is a full featured 
implementation of the ideas below and is a good place to start to look at the 
intricacies of building out all of the features required of a compliant APNS
client. If you don\'t wish to delve into the internals below, check out the lib
and feel free to use, modify, clone, etc...

##Step 1. Creating a socket with your pem files
Basically we just need to create a socket and then do a TLS handshake with it.

{% highlight go %}
package main
import (
    "crypto/tls"
    "ioutil"
    "net"
)

func createSocket() net.Conn {
    certBytes := ioutil.ReadFile("path/to/cert.pem")
    keyBytes := ioutil.Readfile("path/to/key.pem")

    x509Cert, err := tls.X509KeyPair(certBytes, keyBytes)
    if err != nil {
        //failed to validate key pair
        panic(err)
    }

    tlsConf := &tls.Config{
        Certificates: []tls.Certificate{x509Cert},
        ServerName:   "gateway.push.apple.com",
    }

    tcpSocket, err := net.Dial("tcp", "gateway.push.apple.com:2195")
    if err != nil {
        //failed to connect to gateway
        panic(err)
    }

    tlsSocket := tls.Client(tcpSocket, tlsConf)
    err = tlsSocket.Handshake()
    if err != nil {
        //failed to handshake with tls information
        panic(err)
    }

    return tlsSocket
}

func main() {
    socket := createSocket()
}

{% endhighlight %}

At this point, you have a socket ready to go, now we just need to setup reading
and writing to it.

##Step 2. Writing to the socket
We\'ll create a goroutine who\'s sole purpose is to manage writes to our socket.
To do this, we\'ll also need a channel to talk to that goroutine. The channel 
will accept push notification objects (simplified for this example). Those 
push notification objects will be converted to binary frames to be written to 
Apple.

{% highlight go %}
package main
import (
    "bytes"
    "crypto/tls"
    "encoding/binary"
    "encoding/hex"
    "ioutil"
    "net"
)

type PushNotification struct {
    AlertText   string
    Token       string
    id          uint32
}

func (pn *PushNotification) toBytes() []byte {
    buffer := new(bytes.Buffer)
    frameBuffer := new(bytes.Buffer)
    token, err := hex.DecodeString(pn.Token)
    if err != nil {
        //Failed to decode token
        panic(err)
    }
    payloadBytes := []byte("{\"aps\":{\"alert\":" + pn.AlertText + "}}")

    //write token
    binary.Write(buffer, binary.BigEndian, uint8(1))
    binary.Write(buffer, binary.BigEndian, uint16(32))
    binary.Write(buffer, binary.BigEndian, token)

    //write payload
    binary.Write(buffer, binary.BigEndian, uint8(2))
    binary.Write(buffer, binary.BigEndian, uint16(len(payloadBytes)))
    binary.Write(buffer, binary.BigEndian, payloadBytes)

    //write push notification id
    binary.Write(buffer, binary.BigEndian, uint8(3))
    binary.Write(buffer, binary.BigEndian, uint16(4))
    binary.Write(buffer, binary.BigEndian, pn.id)

    //write header info and item info for frame
    binary.Write(frameBuffer, binary.BigEndian, uint8(2))
    binary.Write(frameBuffer, binary.BigEndian, uint32(buffer.Len()))

    return frameBuffer.Bytes()
}

func socketWriter(sendChan chan *PushNotification, socket net.Conn) {
    nextId := uint32(0)
    for pn := range sendChan {
        pn.id = nextId
        socket.Write(pn.toBytes())
        nextId++
    }
}

func createSocket() net.Conn {...}

func main() {
    socket := createSocket()
    sendChan := make(chan *PushNotification)

    go socketWriter(sendChan, socket)
}

{% endhighlight %}

So pretty simple, there\'s a function that turns our push notification objects 
into a byte array as per [Apple\'s guidelines](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Chapters/CommunicatingWIthAPS.html). We then write those bytes to the socket. **Note**
you may have noticed the notification id, this is for error handling a bit later...

##Step 3. Listening for socket close/errors
To listen for errors, we need another goroutine. That goroutine will attempt to
read from the socket and will write to a channel when the connection is closed 
or a response is received from Apple. This also means a slight change to our
socket writer to kill it if the socket is closed. We also need a new type to 
indicate why the connection was closed.

{% highlight go %}
package main
import (
    "bytes"
    "crypto/tls"
    "encoding/binary"
    "encoding/hex"
    "ioutil"
    "net"
)

type SocketClosed struct {
    //Internal ID of the message that caused the error
    MessageID uint32
    //Error code returned by Apple
    ErrorCode uint8
}

type PushNotification struct {...}

func (pn *PushNotification) toBytes() []byte {...}

func socketReader(errChan chan *SocketClosed, socket net.Conn) {
    buffer := make([]byte, 6, 6)
    _, err := socket.Read(buffer)
    if err != nil {
        //the socket was closed but nothing was read
        errChan <- &SocketClosed{
            ErrorCode:   10,
            MessageID:   0,
        }
    } else {
        //apple sent us a response
        messageId := binary.BigEndian.Uint32(buffer[2:])
        errChan <- &SocketClosed{
            ErrorCode:   uint8(buffer[1]),
            MessageID:   messageId,
        }
    }
}

func socketWriter(sendChan chan *PushNotification, errChan chan *SocketClosed, socket net.Conn) {
    nextId := uint32(0)
    shouldClose := false
    for {
        if shouldClose {
            break
        }
        select {
            case pn := <- sendChan:
                pn.id = nextId
                socket.Write(pn.toBytes())
                nextId++
                break
            case err := <- errChan:
                shouldClose = true
                break
        }
    }
}

func createSocket() net.Conn {...}

func main() {
    socket := createSocket()
    sendChan := make(chan *PushNotification)
    errChan := make(chan *SocketClosed)

    go socketReader(errChan, socket)
    go socketWriter(sendChan, errChan, socket)
}

{% endhighlight %}

##Step 4. Handling replaying messages
At this point, we just need a little bit of tidying up to be able to handle 
replaying messages. First we need to keep track of the messages that are \"in 
flight\". Second, on error we need to search for any messages that were sent 
after the error message so that we can do something with them.

{% highlight go %}
package main
import (
    "bytes"
    "container/list"
    "crypto/tls"
    "encoding/binary"
    "encoding/hex"
    "ioutil"
    "net"
)

type SocketClosed struct {...}

type PushNotification struct {...}

type WriterClosed struct {
    //what caused the writer to close
    SocketClosedObj     *SocketClosed
    //any unsent notifications
    UnsentNotifications *list.List
}

func (pn *PushNotification) toBytes() []byte {...}

func socketReader(errChan chan *SocketClosed, socket net.Conn) {...}

func socketWriter(sendChan chan *PushNotification, errChan chan *SocketClosed, 
        writerClosedChan chan *WriterClosed, socket net.Conn) {
    inFlightNotifications = list.New()
    nextId := uint32(1)
    shouldClose := false
    var socketClosed *SocketClosed
    for {
        if shouldClose {
            break
        }
        select {
            case pn := <- sendChan:
                pn.id = nextId

                inFlightNotifications.PushFront(pn)
                //check to see if we've overrun our buffer
                //if so, remove one from the buffer
                if inFlightNotifications.Len() > 1000 {
                    inFlightNotifications.Remove(inFlightNotifications.Back())
                }

                socket.Write(pn.toBytes())
                nextId++
                break
            case socketClosed = <- errChan:
                shouldClose = true
                break
        }
    }

    unsentNotifications := list.New()
    if socketClosed.MessageID > 0 {
        //we received error
        for e := inFlightNotifications.Front(); e != nil; e = e.Next() {
            pn := e.Value.(*PushNotification)
            if pn.id == socketClosed.MessageID {
                break
            }
            unsentNotifications.PushFront(pn)
        }
    }

    writerClosedChan <- &WriterClosed {
        SocketClosedObj: socketClosed,
        UnsentNotifications: unsentNotifications,
    }
}

func createSocket() net.Conn {...}

func main() {
    socket := createSocket()
    sendChan := make(chan *PushNotification)
    errChan := make(chan *SocketClosed)
    writerClosedChan := make(chan *WriterClosed)

    go socketReader(errChan, socket)
    go socketWriter(sendChan, errChan, writerClosedChan, socket)
}

{% endhighlight %}

##Step 5. Putting it all together
So we\'ve got the basics covered, socket connection, writing to the socket, 
listening for socket close or error, and getting a list of unsent notifications
when the connection closes. Now we just need to send some push notifications! To
do this all we need to do is put some push notification objects onto our send 
channel. We also need to listen for the writer to close.

{% highlight go %}
package main
import (
    "bytes"
    "container/list"
    "crypto/tls"
    "encoding/binary"
    "encoding/hex"
    "ioutil"
    "net"
)

type SocketClosed struct {...}

type PushNotification struct {...}

type WriterClosed struct {...}

func (pn *PushNotification) toBytes() []byte {...}

func socketReader(errChan chan *SocketClosed, socket net.Conn) {...}

func socketWriter(sendChan chan *PushNotification, errChan chan *SocketClosed, 
        writerClosedChan chan *WriterClosed, socket net.Conn) {...}

func createSocket() net.Conn {...}

func main() {
    socket := createSocket()
    sendChan := make(chan *PushNotification)
    errChan := make(chan *SocketClosed)
    writerClosedChan := make(chan *WriterClosed)

    go socketReader(errChan, socket)
    go socketWriter(sendChan, errChan, writerClosedChan, socket)

    //write some push notifications
    shouldStopWriting := false
    for i := 0; i < 10; i++ {
        if shouldStopWriting {
            break
        }
        select {
            case sendChan <- &PushNotification {
                    AlertText: "Hello World!",
                    Token: <a token>,
                }:
                break
            case writerClosed := <- writerClosedChan:
                shouldStopWriting := true
                //do something with unsent notifications
                break
        }
    }
}

{% endhighlight %}

##Step 6. Doing it right
Whew, so that was a lot of code and a lot of things to think about. Unfortunately
the work isn\'t done yet and raises some more questions.

* How do we wrap this up into a nice library?
* What do we do with list of unsent notifications?
* Do you wait for a period after sending to see if an error comes in?
* How do we handle Nagling in the socket layer?
* How do we implement all the other fields allowed by the APNS service?

Basically these tend to be implementation specific and can\'t be covered in a 
general way and are left to the reader to implement.

##Why does this have to be so complicated?
The main motivation for Apple to build a system like this is to create a stable, 
robust communication spec that is high throughput and efficient for Apple and 
people implementing the APNS spec. This means the use cases of sending 5-10-100
push notifications have been glossed over for the use cases of sending 100,000
push notifications at a time. 

##What have we learned?
The APNS spec is a bit tricky, but when it comes down to it, the spec isn\'t 
really that hard to implement. With a little work we\'ve got a good start 
towards being able to build a robust and pretty efficient APNS client. And 
hopefully this gives some insight into how one might go about building a similar
client in another language.