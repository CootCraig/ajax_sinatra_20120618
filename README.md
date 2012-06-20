# Linked Servers Distributing AJAX events from multiple sources

A presentation to Not Just Ruby by the "Coot" Craig Anderson.

## Project Overview

The call center I work at wanted to distribute call progress events to
various pieces of software on computers throughout the company intranet.
There is both inbound and outbound work.

A windows DLL is available to get inbound call events from the phone switch.

No centralized source was available for outbound calls, but messages
are sent to the workstations of outbound agents logged into the dialer.

The shop infracture is Windows OS, MSSQL and Firefox for the browser.

I'm a happy Ruby programmer and management was fine with that so a
developing I went.

![System Diagram](https://github.com/CootCraig/ajax_sinatra_20120618/blob/master/system_diagram.png)

<img src="https://github.com/CootCraig/ajax_sinatra_20120618/blob/master/system_diagram.png" alt="System Diagram"></img>

## The Plan such as it was

1. I've used Sinatra as an HTTP application before, so that's decided.
1. Distributing events sounds like AJAX. I've never worked with AJAX.  Does it work with Sinatra.  I'll find out.
1. There's this DLL for getting events from the phone switch; sure hope I can get this to work.

## Access the Phone Switch DLL from Ruby
There's a gem for that: Here's one that will connect DLLs to Ruby

```ruby
require 'ffi'
```

I know the name of the DLL, cvlancli.dll, and have a C header file with the interface, asai_str.h.

The C functions look like this:

```C
DECLDLL	long asai_close(const void *libPtr, int fd);
DECLDLL	long asai_errval(const void *libPtr, char mes_buf[]);
DECLDLL	long asai_get_env(const void *libPtr, int fd, long characteristic, get_type *arg);
DECLDLL	long asai_rcv(const void *libPtr, int fd, char *buf, long length);
DECLDLL	long asai_send(const void *libPtr, int fd, const char buf[], const long length);
DECLDLL	long asai_set_env(const void *libPtr, int fd, long characteristic, set_type *arg);
DECLDLL	int asai_open(const void *libPtr, char path[], int flags);
```

The Ruby wrapper to the DLL functions.

```ruby
require 'ffi'
module CvlanLib
  extend FFI::Library

  module CvlanDll
    extend FFI::Library

    ffi_lib File.expand_path('../cvlancli.dll',__FILE__)

    attach_function :asai_open, [:string, :int], :int
    attach_function :asai_set_env, [:int, :int, :string], :int
    attach_function :asai_send, [:int, :pointer, :int], :int
    attach_function :asai_rcv, [:int, :pointer, :int], :int
  end
end
```

A sample struct.

```ruby
typedef struct {
	capability_t	capability;
	primitive_t	primitive_type;
	long	cluster_id;
	long	reserved;
} asai_common_t;

typedef struct {
	asai_common_t	asai_common;
	long			domain_type;
	char			*domain_ext;
	char			pool[C_DATSZ];
} cv_info_t;
```

Look at this, There are enums passed back and forth.  In C.

```C
/* service_type values indicating capability */
typedef enum {
	C_3PCC,			/*  0 */
...
	C_EN_REQ,		/* 23 */
...
	C_3PSSC_CONF		/* 58 */
} capability_t;
```

In FFI / Ruby.

```C
  CAPABILITY_T = enum(
    :C_3PCC ,  0,
...
    :C_EN_REQ ,  23,
...
    :C_3PSSC_CONF ,  58
  )
```

The event interface is handled by 2 ruby functions with the help of FFI.

```ruby
module CvlanLib
  def self.asai_rcv(fd)
    raw_buf = CvlanLib::RAW_BUF.new
    stat = CvlanLib::CvlanDll.asai_rcv(fd,raw_buf.to_ptr,raw_buf.size)
    {stat: stat,
      fd: fd,
      buf: (stat>0) ? raw_buf : nil,
      time: Time.now}
  end
  def self.asai_open_vdn_events(mapd_ip, vdn, cvlan_node)
    puts "asai_open_vdn_events mapd_ip #{mapd_ip}, vdn #{vdn}, cvlan_node #{cvlan_node}"
    fd = CvlanLib::CvlanDll.asai_open(mapd_ip, 0)
    puts "asai_open fd=#{fd}"
    stat = CvlanLib::CvlanDll.asai_set_env(fd,CvlanLib::ENVIRONMENT_CHARACTERISTICS[:C_NODE_ID],cvlan_node)
    cv_info = CvlanLib::CV_INFO_T.new
    cv_info[:capability] = CvlanLib::CAPABILITY_T[:C_EN_REQ]
    cv_info[:primitive_type] = CvlanLib::PRIMITIVE_T[:C_REQUEST]
    cv_info[:cluster_id] = 0
    cv_info[:domain_ext] = cv_info[:pool].to_ptr
    cv_info[:domain_type] = CvlanLib::REQUEST_NOTIFICATION_DOMAIN_TYPE[:C_CALL_VECTOR]
    cv_info[:pool].to_ptr.put_string(0,vdn)
    stat = CvlanLib::CvlanDll.asai_send(fd,cv_info.to_ptr,cv_info.size)
    fd
  end
end
```

Whew! that wasn't easy, but FFI made it straight forward.

## Collecting the Inbound events

The phone switch is programmed so that the inbound phone numbers, say 800-123-4567, are referenced by the last 4 digits, 4567, known as a "Vector Directory Number" (VDN).

After a lot of experimentation I find that VDN is accessed through the DLL on separate network sockets.

```ruby
asai_fd = CvlanLib::asai_open_vdn_events(APP_CONFIG['cvlan_ip'],vdn,APP_CONFIG['cvlan_node'])
```

The active VDNs are listed in a config file.

This means that the call events for each VDN are read on separate file descriptors(FDs).

```ruby
rcv = CvlanLib.asai_rcv(@fd)
```

The events could be collected in an evented manner or a threaded manner.  I chose to do blocking reads on each FD in separate threads.

There must be a gem for that.  I picked Celluloid.

## Celluloid

[Celluloid - Painless multithreaded programming for Ruby](http://celluloid.io)

Celluloid is a concurrent object oriented programming framework for Ruby which lets you build multithreaded programs out of concurrent objects just as easily as you build sequential programs out of regular objects

The author, Tony Acrieri has distilled a lot of coding experience and made them accessable in the celluloid.io gems.  The online documentation is very informative.

[Basic usage](https://github.com/celluloid/celluloid/wiki/Basic-usage)

A little example shows how easy the basics are:

```ruby
class Sheen
  include Celluloid

  def initialize(name)
    @name = name
  end

  def set_status(status)
    @status = status
  end

  def report
    "#{@name} is #{@status}"
  end
end
```

Let's take a look at this on the wiki.

## Event pipeline using Celluloid

Note the use of the Celluloid actor registry.

```ruby
Celluloid::Actor[:connected_calls] = CvlanCallEventSourceApp::ConnectedCallsActor.new(@@zmq_publish_socket_port)
writer_target = Proc.new do |call_event|
  Celluloid::Actor[:connected_calls].publish!(call_event)
end

inject_target = Proc.new do |call_event|
  Celluloid::Actor[:connected_calls].broadcast_event!(call_event)
end
Celluloid::Actor[:inject_events] =
              CvlanCallEventSourceApp::InjectEventActor.new(@@zeromq_inject_event_uri, inject_target)

Celluloid::Actor[:asai_parse] = CvlanCallEventSourceApp::AsaiParseEventActor.new(writer_target)

rcv_target = Proc.new do |call_event|
  Celluloid::Actor[:asai_parse].parse!(call_event)
end

APP_CONFIG['vdns'].each do |vdn|
  actor_symbol = "rcv_vdn_#{vdn}".to_sym
  asai_fd = CvlanLib::asai_open_vdn_events(APP_CONFIG['cvlan_ip'],vdn,APP_CONFIG['cvlan_node'])
  @@logger.info "Starting AsaiRcvActor #{vdn} fd #{asai_fd}"
  Celluloid::Actor[actor_symbol] = CvlanCallEventSourceApp::AsaiRcvActor.new(vdn,asai_fd,rcv_target)
  Celluloid::Actor[actor_symbol].run!
end
```
4 types of actor form the event pipeline.

```
AsaiRcvActor ---|
AsaiRcvActor ---|----> AsaiParseEventActor ---> ConnectedCallsActor
 ...            |                                 |
AsaiRcvActor ---|                                 |
                                                  |
InjectEventActor ---------------------------------|
```

2 of the actors do network IO.

+ The InjectEventActor thread receives outbound call events from outbound agent workstations via a 0MQ request-reply socket pair.
+ The ConnectedCallsActor thread writes the call events to a 0MQ pub socket

## Omq Overview

[The Intelligent Transport Layer](http://www.zeromq.org/)

[The excellent guide for 0mQ](http://zguide.zeromq.org/page:all)

### From the Guide - ØMQ in a Hundred Words

ØMQ (ZeroMQ, 0MQ, zmq) looks like an embeddable networking library but
acts like a concurrency framework. It gives you sockets that carry whole
messages across various transports like in-process, inter-process, TCP,
and multicast. You can connect sockets N-to-N with patterns like fanout,
pub-sub, task distribution, and request-reply. It's fast enough to be
the fabric for clustered products. Its asynchronous I/O model gives you
scalable multicore applications, built as asynchronous message-processing
tasks. It has a score of language APIs and runs on most operating
systems. ØMQ is from iMatix and is LGPL open source.

I heard of 0MQ because DCell, the distributed extension to Celluloid
uses 0MQ for remote messaging.

## VB.net 0mq Event Injector

I wrote a VB.net console application to send arbitrary events
to a 0mq

```
    Sub Main(ByVal args As String())
            Dim zmq_context As New ZMQ.Context
            Dim zsock As ZMQ.Socket = Nothing
            Dim msg As New Dictionary(Of String, String)

            zsock = zmq_context.Socket(SocketType.REQ)
            zsock.Connect(My.Settings.InjectEventUri)

            For Each arg As String In args
                If key Is Nothing Then
                    key = arg
                Else
                    val = arg
                    msg.Add(key, val)
                    key = Nothing
                    val = Nothing
                End If
            Next arg
            Dim sw As New StringWriter
            sw.Write("{")
            Dim not_first As Boolean = False
            For Each kvp As KeyValuePair(Of String, String) In msg
                If not_first Then
                    sw.Write(",")
                End If
                sw.Write(String.Format("""{0}"" : ""{1}""", kvp.Key, kvp.Value))
                not_first = True
            Next
            sw.Write("}")
            Dim json As String = sw.ToString

            logger.Info(String.Format("v{0} :: {1} :: {2}", My.Settings.version, My.Settings.InjectEventUri, json))

            Dim timeoutAlarm As New System.Threading.Timer(AddressOf sendTimeout, "Send", 10 * 1000, System.Threading.Timeout.Infinite)

            zsock.Send(json, System.Text.Encoding.UTF8)
            Dim response = zsock.Recv(System.Text.Encoding.UTF8)
            logger.Info(String.Format("Response [{0}]", response))
        Catch ex As Exception
            logger.Fatal(String.Format("Exception: {0}", ex.Message))
            Environment.Exit(2)
        End Try
        Environment.Exit(0)
    End Sub
```

## Distributing the call events with AJAX

### Events are read on a 0MQ port

The setup.

```ruby
distribution_target = Proc.new do |call_event|
  Celluloid::Actor[:EventDistribution].event! call_event
end
Celluloid::Actor[:CvlanCallEvents] =
   CvlanAjaxEvents::CvlanCallEvents.new("tcp://127.0.0.1:#{@@zmq_publish_socket_port}",distribution_target)
Celluloid::Actor[:CvlanCallEvents].run!
```

The implementation.

```ruby
require 'celluloid/zmq'
require 'json'

module CvlanAjaxEvents
  class CvlanCallEvents
    include Celluloid::ZMQ
    def initialize(zmq_url,target)
      @zmq_url = zmq_url
      @target = target
      @zmq_socket = SubSocket.new
      @zmq_socket.setsockopt(::ZMQ::SUBSCRIBE,"")
      @zmq_socket.connect(@zmq_url)
    end
    def run
      while true
        msg = @zmq_socket.read
        @logger.debug "read event #{msg}"
        event = JSON.parse msg
        @target.call(event)
      end
    end
  end
end
```

### Sinatra HTTP endpoint

## JavaScript for the browser

