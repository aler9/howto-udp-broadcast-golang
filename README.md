
# Howto: UDP broadcast in Go

Sample code and instructions on how to send and receive UDP broadcast packets in the Go programming language.

## Introduction

A common method to trasmit informations to multiple devices consists in using the UDP transport layer and sending the packets to the broadcast address associated with the current LAN. The router will take care of propagating the packets to each connected device. There are at least two advantages with respect to normal network communication:
* transmission is much more efficient, as a single stream of data is used for communicating with multiple devices
* it is not necessary to know in advance the addresses of the recipients

The Go programming languages offers at least 3 ways to send UDP broadcast packets, the situation is not very clear and that's the reason why this guide is being written.

## Sending

Let's assume we want to transmit the ascii-encoded phrase "data to transmit" though port `8829` in a network with IP `192.168.7.2` and broadcast address `192.168.7.255`.

The first (and preferred) method consists in using the net.ListenPacket() function:
```go
package main

import (
  "net"
  "fmt"
)

func main() {
  pc, err := net.ListenPacket("udp4", ":8829")
  if err != nil {
    panic(err)
  }
  defer pc.Close()

  addr,err := net.ResolveUDPAddr("udp4", "192.168.7.255:8829")
  if err != nil {
    panic(err)
  }

  _,err := pc.WriteTo([]byte("data to transmit"), addr)
  if err != nil {
    panic(err)
  }
}
```

A second method consists in using the net.ListenUDP() function:
```go
package main

import (
  "net"
)

func main() {
  listenAddr,err := net.ResolveUDPAddr("udp4", ":8829")
  if err != nil {
    panic(err)
  }
  list, err := net.ListenUDP("udp4", listenAddr)
  if err != nil {
    panic(err)
  }
  defer list.Close()

  addr,err := net.ResolveUDPAddr("udp4", "192.168.7.255:8829")
  if err != nil {
    panic(err)
  }

  _,err := list.WriteTo([]byte("data to transmit"), addr)
  if err != nil {
    panic(err)
  }
}
```


A third method consists in using the net.DialUDP() function:
```go
package main

import (
  "net"
)

func main() {
  local,err := net.ResolveUDPAddr("udp4", ":8829")
  if err != nil {
    panic(err)
  }
  remote,err := net.ResolveUDPAddr("udp4", "192.168.7.255:8829")
  if err != nil {
    panic(err)
  }
  list, err := net.DialUDP("udp4", local, remote)
  if err != nil {
    panic(err)
  }
  defer list.Close()

  _,err := list.Write([]byte("data to transmit"))
  if err != nil {
    panic(err)
  }
}
```

All three method works and their result is indistinguishable. By looking at the Go source code, it is possible to assert that:
* [net.ListenPacket()](https://golang.org/src/net/dial.go?s=19117:19219#L625) and [net.ListenUDP()](https://golang.org/src/net/udpsock.go?s=6961:7025#L221) use a nearly identical identical procedure, as they both call the same system functions, the only difference is that net.ListenPacket() converts the desired listen address (`:8829`) into an UDPAddr structure, while net.ListenUDP() requires directly an UDPAddr structure;
* [net.DialUDP()](https://golang.org/src/net/udpsock.go?s=5929:5998#L195) uses a different route and also provides a Write() function to write directly to the broadcast address. This could be confusing, as Go always work with WriteTo() and ReadFrom() when dealing with UDP connections.

Conclusion: i use with the ListenPacket() solution as it is the simpler one.

## Receiving

Data can be received with a standard UDP server listening on the provided port:
```go
package main

import (
  "fmt"
  "net"
)

func main() {
  pc,err := net.ListenPacket("udp4", ":8829")
  if err != nil {
    panic(err)
  }
  defer pc.Close()

  buf := make([]byte, 1024)
  n,addr,err := pc.ReadFrom(buf)
  if err != nil {
    panic(err)
  }
  
  fmt.Printf("%s sent this: %s\n", addr, buf[:n])
}
```

