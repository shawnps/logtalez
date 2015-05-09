logtalez
========

logtalez, oo-ooo

# What Is This?
## Overview
logtalez is a minimal command line client (and API) for retrieving log streams from the
rsyslog logging daemon over zeromq. 

An rsyslog config emits logs over a zeromq publish socket, dynamically
constructing topics from the host and program name that generated the log.

The command line client can be used to subscribe to host / program name 
combinations and the logs are fed to stdout, where you can interact with 
them using whatever tools you wish.

Access to the zeromq socket is controlled via encryption certs, and all 
traffic between the server and client is encrypted.

# How Do I Build and Use This?
## Dependencies
### [libsodium](https://github.com/jedisct1/libsodium)
Version: 1.0.2 (or newer)

Sodium is a "new, easy-to-use software library for encryption, decryption, signatures, password hashing and more".  ZeroMQ uses sodium for the basis of the CurveZMQ security protocol.

```
git clone git@github.com:jedisct1/libsodium.git
cd libsodium
./autogen.sh; ./configure; make; make check
sudo make install
sudo ldconfig
```

### [ZeroMQ](http://zeromq.org/) 
Version: commit 6b4d9bca0c31fc8131749396fd996d17761c999f or newer

ZeroMQ is an embeddable [ZMTP](http://rfc.zeromq.org/spec:23) protocol library.

```
git clone git@github.com:zeromq/libzmq.git
cd libzmq
./autogen.sh; ./configure --with-libsodium; make; make check
sudo make install
sudo ldconfig
```

### [CZMQ](http://czmq.zeromq.org/)
Version: commit 7997d86bcac7a916535338e71f2d826a9913df28 or newer

CZMQ is a high-level C binding for ZeroMQ.  It provides an API for various services on top of ZeroMQ such as authentication, actors, service discovery, etc.

```
git clone git@github.com:zeromq/czmq.git
cd czmq
./autogen.sh; ./configure; make; make check
sudo make install
sudo ldconfig
```

### [GoCZMQ](http://https://github.com/zeromq/goczmq)
Version: commit 6b4d9bca0c31fc8131749396fd996d17761c999f or newer

GoCZMQ is a Go interface to the CZMQ API.

```
mkdir -p  $GOPATH/go/src/github.com/zeromq
cd $GOPATH/go/src/github.com/zeromq
git clone git@github.com:zeromq/goczmq.git
cd goczmq
go test
go build; go install
```


### [Rsyslog](http://www.rsyslog.com/)
Version: 8.9.0 or newer

Rsyslog is the "rocket fast system for log processing".

Build dependencies:
* libsodium
* libzmq
* czmq
* libjson-c
* libestr
* liblognorm
* liblogging

You will need to use the "--enable-omczmq" configure flag to build zeromq + curve support.

## Generating Certificates
logtalez uses CURVE security certificates generated by the [zcert](http://api.zeromq.org/czmq3-0:zcert) API.  They are stored in [ZPL](http://rfc.zeromq.org/spec:4) format.  Logtalez includes a simple cert generation tool (curvecertgen) for convenience.

To generate a public / private key pair:

```
$ ./curvecertgen bogus_cert
Name: Brian
Email: bogus@whatever.com
Organization: Bogus Org
Version: 1
```

The above would generate a bogus_cert and bogus_cert_secret file.

## Configuring Your Rsyslog Server

The following rsyslog configuration snippet consists of:
* A template that dynamically sets a "topic" on a message consisting of hostname.syslogtag + an "@cee" cookie and JSON message payload
* A rule snippet that attempts to parse a syslog message as JSON, then outputs it over a zeromq publish socket using the template

```
module(load="mmjsonparse")
module(load="omczmq")

template(name="pubsub_host_tag" type="list") {
  property(name="hostname")
  constant(value=".")
  property(name="syslogtag" position.from="1" position.to="32")
  constant(value="@cee:")
  constant(value="{")
  constant(value="\"@timestamp\":\"")
  property(name="timereported" dateFormat="rfc3339" format="json")
  constant(value="\",\"host\":\"")
  property(name="hostname")
  constant(value="\",\"severity\":\"")
  property(name="syslogseverity-text")
  constant(value="\",\"facility\":\"")
  property(name="syslogfacility-text")
  constant(value="\",\"syslogtag\":\"")
  property(name="syslogtag" format="json")
  constant(value="\",")
  property(name="$!all-json" position.from="2")
} 

ruleset(name="zmq_pubsub_out") {
  action(
    name="zmq_pubsub"
    template="pubsub_host_tag"
    type="omczmq"
    endpoints="tcp://*:24444"
    socktype="PUB"
    authtype="CURVESERVER"
    clientcertpath="/etc/curve.d/"
    servercertpath="/etc/curve.d/my_server_cert"
  )
}

action(type="mmjsonprase")
if $parsesuccess == "OK" then {
  call zmq_pubsub_out
} 
```

## Usage

`````go
	import "github.com/digitalocean/logtalez"

	func main() {
		zmqconns := "tcp://127.0.0.1:24444,tcp://example.com:24444"
		endpoints := logtalez.MakeEndpointList(zmqconns)

		hosts := "host1,host2,host3"
		programs := "nginx"
		topics := logtalez.MakeTopicList(hosts, programs)

		serverCert := "/home/me/.curve/server_public_cert"
		clientCert := "/home/me/.curve/client_public_cert"

		lt, err := logtalez.New(endpoints, topics, serverCert, clientCert)
		if err != nil {
			panic(err)
		}

		sigChan := make(chan os.Signal, 1)
		signal.Notify(sigChan, os.Interrupt, os.Kill)

		for {
			select {
			case msg := <-lt.TailChan:
				logline := strings.Split(string(msg[0]), "@cee:")[1]
				fmt.Println(logline)
			case <-sigChan:
				lt.Destroy()
				os.Exit(0)
			}
		}
	}
`````

## Todo
* Support for custom topic formats and custom json delimiters
* Dockerized rsyslog "appliance" with support baked in

## Tools That Work Well with Logtalez
* [jq](https://stedolan.github.io/jq/) JSON processor
* [humanlog](https://github.com/aybabtme/humanlog) "Logs for humans to read."
* Anything that can read stdout!

## GoDoc

[godoc](https://godoc.org/github.com/digitalocean/logtalez)

## License

This project uses the MPL v2 license, see LICENSE
