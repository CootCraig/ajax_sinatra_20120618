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

The phone switch is programmed so that the inbound phone numbers, say
800-123-4567, are referenced by the last 4 digits, 4567, known as a
"Vector Directory Number" (VDN).

After a lot of experimentation I find that a VDN is accessed through
the DLL on separate network sockets.  The active VDNs are listed in a
config file.

```ruby
APP_CONFIG['vdns'].each do |vdn|
  actor_symbol = "rcv_vdn_#{vdn}".to_sym
  asai_fd = CvlanLib::asai_open_vdn_events(APP_CONFIG['cvlan_ip'],vdn,APP_CONFIG['cvlan_node'])
  Celluloid::Actor[actor_symbol] = CvlanCallEventSourceApp::AsaiRcvActor.new(vdn,asai_fd,rcv_target)
  Celluloid::Actor[actor_symbol].run!
end
```
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

## Daily restart the JRuby / Celluloid way

The call center has an overnight downtime.  This is convenient, the software can be restarted every night. The following code
forces an exit nightly.  The Windows Task Scheduler has a task to restart it.

~~~ruby
Celluloid::Actor[:Shutdown] = CvlanCallEventSourceApp::Shutdown.new
~~~

~~~ruby
class Shutdown
  include Celluloid
  attr_reader :shutdown_time, :start_time, :timer

  Runtime = java.lang.Runtime.getRuntime()

  def initialize
    @start_time = Time.now
    day_later = @start_time + (24 * 60 * 60)
    @shutdown_time = Time.new day_later.year, day_later.month, day_later.day, 1, 8
    delay_in_seconds = (@shutdown_time - @start_time).to_i

    if APP_CONFIG['debug_quick_shutdown']
      quick_delay = 5 * 60
      @shutdown_time = @start_time + quick_delay
      delay_in_seconds = (@shutdown_time - @start_time).to_i
    end
    @timer = after(delay_in_seconds) do
      Runtime.halt(1)
    end
  end
end
~~~

Note the Java called from JRuby

~~~ruby
Runtime = java.lang.Runtime.getRuntime()
  ...
Runtime.halt(1)
~~~

It turns out that a graceful shutdown of all these threads takes
some effort.  For this project the drastic kill approach above works
fine. Celluloid is a fairly new library and better shutdown support will
be added.

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

### Actor CvlanCallEvents reads events on a 0MQ port and passes them along.

The setup. Note the ! at the end of the method. In Celluloid this means asynchronous call.

```ruby
distribution_target = Proc.new do |call_event|
  Celluloid::Actor[:EventDistribution].event! call_event
end
Celluloid::Actor[:CvlanCallEvents] =
   CvlanAjaxEvents::CvlanCallEvents.new("tcp://127.0.0.1:#{@@zmq_publish_socket_port}",distribution_target)
Celluloid::Actor[:CvlanCallEvents].run!
```

The implementation.  The run method loops forever doing blocking reads on the 0MQ subscribe socket.  This works because the method was invoked asynchronously.

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

### Sinatra HTTP AJAX endpoint

The function of the call is to get the next event for an extension number.

```ruby
module CvlanAjaxEvents
  class Server < Sinatra::Base
    helpers Sinatra::Jsonp
...
    get '/event' do
      extension = params[:extension]
      last_event_id = params[:eventid]
      puts "extension #{extension} last_event_id #{last_event_id}"
      response['Access-Control-Allow-Origin'] = '*'
      if extension && (extension.length>0)
        last_event_id = '' unless last_event_id && (last_event_id.length>0)
        JSONP Celluloid::Actor[:EventDistribution].get_event_for_extension(extension,last_event_id)
      else
        JSONP( { :error => 'Invalid parameters' } )
      end
    end
...
    configure do
      CvlanAjaxEvents::App.start
    end
  end
end
```

This response header is needed for cross-site use from the browser.

```ruby
response['Access-Control-Allow-Origin'] = '*'
```

Every request is handled by a separate thread.  The AJAX requests may wait for the requested event. Here is the Celluloid method call to get an event.  Note that there is no ! suffix on the call, so it is synchronous.

```ruby
JSONP Celluloid::Actor[:EventDistribution].get_event_for_extension(extension,last_event_id)
```

### Actor EventDistribution receives events and responds to AJAX requests for events

```ruby
Celluloid::Actor[:EventDistribution] = CvlanAjaxEvents::EventDistribution.new
```

#### Here is the method that is called synchonously for the next event.

```ruby
def get_event_for_extension(extension,call_unique_id)
  @get_event_count += 1
  symbol = extension_symbol extension
  evt = @events_by_extension[symbol]
  if !evt
    evt = { :connect_num => extension, :call_unique_id => UUIDTools::UUID.parse_int(0).to_s }
    evt[:event_id] = next_serial_number
    @events_by_extension[symbol] = evt
  elsif call_unique_id.to_s == evt[:event_id].to_s
    evt = wait symbol
  end
  @logger.debug "returning event for extension #{extension} event #{evt}"
  evt
end
```

The wait call is where the method suspends.

```ruby
evt = wait symbol
```

[Signaling](https://github.com/celluloid/celluloid/wiki/Signaling) is an advanced technique similar to condition variables in typical multithreaded programming

#### Method event stores the events and signals them

```ruby
def event(wire_evt)
  begin
    evt = {}
    wire_evt.each { |key,val| evt[key.to_sym] = val }
    if evt[:connect_num]
      wire_evt = nil
      evt[:event_id] = next_serial_number
      symbol = extension_symbol evt[:connect_num]
      @events_by_extension[symbol] = evt
      signal symbol, evt
    else
      @logger.error "event has no connect_num #{evt}"
    end
  rescue => ex
    @logger.error ex.to_s
  end
end
```

Whew! Took me a while to get here, but it's all working.  All those nice gems made it straight forward.

## JavaScript for the browser

Wait, Wait, how does a browser get the events?

The browser code is packaged in 2 files,

### My sample page

```haml
!!! 5
%html
  %head
    %title Demo Ajax Events
    %link{:rel => 'stylesheet', :href => '/stylesheets/style.css', :type => 'text/css'}
    %script{:src => "/js/json2.min.js", :type => "text/javascript"}
    %script{:src => "/js/jquery-1.7.1.min.js", :type => "text/javascript"}
    %script{:src => "/js/cvlan_ajax_events_parms.js", :type => "text/javascript"}
    %script{:src => "/js/cvlan_ajax_events.js", :type => "text/javascript"}
    %script{:src => "/js/demo.js", :type => "text/javascript"}
  %body
    %h1 Demo Ajax Events for an Extension - add parms
    .ext_form_div
      %input#ext_input{ :name => 'ext_input', :type => 'text' }
      %br
      %button#ext_button
        Show Events for Extension:
      %br
      %input#ext_log{ :name => 'ext_log', :type => 'text', :readonly => 'readonly' }
```

Note the 2 script files

### cvlan_ajax_events_parms.js

This file sets the specifics for a page. A template file is provided.

```javascript
/*
var APP_CALL_EVENTS_PARMS = {};
APP_CALL_EVENTS_PARMS.event_url = 'http://$host:$port/event';
APP_CALL_EVENTS_PARMS.call_start_func = function(event) {
};
APP_CALL_EVENTS_PARMS.call_end_func = function(event) {
};
 */
```

### cvlan_ajax_events.js

There is a default for our production HTTP server.  The default action is to append events to the page.

```javascript
/*
 * Setting up:
 * APP_CALL_EVENTS_PARMS.event_url = {};
 * APP_CALL_EVENTS_PARMS.event_url = 'http://host:port/event';
 * APP_CALL_EVENTS_PARMS.call_start_func = function() {};
 * APP_CALL_EVENTS_PARMS.call_end_func = function() {};
*/
var call_event_info = (function() {
  var app_parms;
  try {
    app_parms = APP_CALL_EVENTS_PARMS;
  } catch(e) {
    app_parms = {};
  }
  var interface = {};
  var event_url = app_parms.event_url || 'http://gcs1:38008/event';
  var call_start_func = app_parms.call_start_func || function(event) {
    var msg = "Call start. ";
    try {
      msg = msg + "event_name_s ";
      msg = msg + event.event_name_s;

      msg = msg + ". calling_num ";
      msg = msg + event.calling_num;

      msg = msg + ". connect_num ";
      msg = msg + event.connect_num;

      msg = msg + ". event_id ";
      msg = msg + event.event_id;

      msg = msg + ". vdn ";
      msg = msg + event.vdn;

      $('<h2/>').text(msg).appendTo('body');
    } catch(e) {
    }
  };
  var call_end_func = app_parms.call_end_func || function(event) {
    var msg = "Call end. ";
    try {
      msg = msg + "event_name_s ";
      msg = msg + event.event_name_s;

      msg = msg + ". calling_num ";
      msg = msg + event.calling_num;

      msg = msg + ". connect_num ";
      msg = msg + event.connect_num;

      msg = msg + ". event_id ";
      msg = msg + event.event_id;

      msg = msg + ". vdn ";
      msg = msg + event.vdn;

      $('<h2/>').text(msg).appendTo('body');
    } catch(e) {
    }
  };
  var SUCCESS_DELAY = 300;
  var ERROR_DELAY = 8 * 1000;
  var NO_EXTENSION = 'x';
  var NO_EVENT_ID = 'i';
  var extension = NO_EXTENSION;
  var last_event_id = NO_EVENT_ID;
  var get_last_event_id = function(){ return last_event_id; };
  var set_last_event_id = function(event_id){ last_event_id = event_id; };
  var ajax_call_delay = SUCCESS_DELAY;
  var last_jqxhr = null;

  var ajax_success = function(call_event_string,status,xhr) {
    var call_event,event_id,event_name;
    try {
      call_event = JSON.parse(call_event_string);
      try {
        event_id = call_event.event_id || NO_EVENT_ID;
      } catch (ex2) {
        event_id = NO_EVENT_ID;
      }
      set_last_event_id(event_id);
      event_name = call_event.event_name_s;
      if ((/c_drop|c_callend/i).test(event_name)) {
        call_end_func(call_event);
      } else if (/c_connected/i.test(event_name)) {
        call_start_func(call_event);
      }
      ajax_call_delay = SUCCESS_DELAY;
    } catch (ex1) {
      ajax_call_delay = ERROR_DELAY;
    }
  };
  var ajax_error = function(xhr,status) {
    ajax_call_delay = ERROR_DELAY;
  };
  var ajax_complete = function(xhr,status) {
    setTimeout(function(){
      ajax_call(extension,last_event_id);
    }, ajax_call_delay);
  };
  var ajax_call = function(extension, last_event_id) {
    var call_data = {};
    call_data.extension = extension;
    call_data.eventid = last_event_id;

    last_jqxhr = $.ajax({
      url : event_url,
      data : call_data,
      type : 'GET',
      //
      // code to run if the request succeeds;
      // the response is passed to the function
      success : ajax_success,

      // code to run if the request fails;
      // the raw request and status codes are
      // passed to the function
      error : ajax_error,

      // code to run regardless of success or failure
      complete : ajax_complete
    });
  };

  interface.set_extension = function(aExtension) {
    extension = aExtension;
  };
  interface.start_call_events = function() {
    // todo: cancel existing ajax call
    setTimeout(function(){
      ajax_call(extension,last_event_id);
    }, 500 );
  };
  return interface
})();
```

## JRuby / Trinidad configs for production

One last teeny tiny problem.  When this was first put into production the
events stopped after an hour or 2.  It turns out the default thread pool
was being maxed out (one thread per request).  A little configuration
magic fixed it right up.

### run.bat

```
C:\jruby\jruby-1.6.8.dev\bin\jruby.exe
 --server
 -J-Xss256k
 -S trinidad
 --threadsafe
 --config c:\GCS\apps\cvlan_ajax_events\sinatra_base\config\trinidad.yml
```

### trinidad.yml

```ruby
---
  port: 38008
  rackup: config.ru
  http:
    maxThreads: 6000
    connectionTimeout: 60000
```

