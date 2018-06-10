
# The Event definition

The event definition and streaming have a central place in a DQM online setup. It defines what kind of event you will collect and analyze during data taking, as well as how you read and write your data in binary format. DQM4hep defines a simple interface to hold your data: the **EventBase< T >** class. This class also holds basic properties you may want to use for your event definition. These properties are very common in high energy physics experiments:

- **Event type** (enumerator), the event type on data taking workflow. Possible values are:
    - **UNKNOWN_EVENT**: the default flag when an event is created
    - **RAW_DATA_EVENT**: data coming from a single device, e.g one DIF or one particular sensor
    - **RECONSTRUCTED_EVENT**: a reconstructed event after going through the event builder. This kind of event is generally a combination of *RAW_DATA_EVENT* events
    - **PHYSICS_EVENT**: the *RAW_DATA_EVENT* and *RECONSTRUCTED_EVENT* events usually contain raw data (binary buffers very specific to readout chips). The *PHYSICS_EVENT* type is defined for events containing data that have been translated into "physics readable" data, e.g calorimeter hits, tracker hits or reconstructed particles.
    - **CUSTOM_EVENT**: an additional flag in case none of the previous types fits in your case
- **Source** (string), the name of source (e.g device) that have created the event,
- **Time stamp**, the time stamp of the event creation. The type of this property is [std::chrono::system_clock::time_point](http://en.cppreference.com/w/cpp/chrono/system_clock),
- **Event size** (uint32_t), a user defined measurement of the event size. E.g, it can be the number of sub-elements hold by the event or the size in bytes of the event after serialization,
- **Run number** (uint32_t), the run number during which the event was created,
- **Event number** (uint32_t), the event number.

Our definition of the event does not provide a specific model but defines an interface to hold a user specific event. As usual, an example is always better :

```cpp
// This is an external event defined in your lovely framework
MyEvent* event = new MyEvent();

// This is the DQM4hep event
// Note the polymorphism with the Event class
dqm4hep::core::Event* dqmEvent = new dqm4hep::core::EventBase<MyEvent>(event);

// Set the properties
dqmEvent->setTimeStamp( dqm4hep::core::now() );
dqmEvent->setEventNumber( 1 );
dqmEvent->setRunNumber( 42 );
dqmEvent->setSource( "UserManual" );
dqmEvent->setEventType( dqm4hep::core::CUSTOM_EVENT );

// The handled event can be retrieved
MyEvent *eventget = dqmEvent->getEvent<MyEvent>();
assert( event == eventget );
```

Note the getEvent< T >() method use on last lines. This method casts the *Event* object to an *EventBase< T >* object and access the event stored in this class, something like :

```cpp
template <typename T>
T *Event::getEvent() {
  return dynamic_cast< EventBase<T>* >(this)->getEvent();
}
```

<div class="warning-msg">
  <i class="fa fa-warning"></i>
  As c++ is a strongly typed language, you have to make sure that the template parameter type T in the <span style="font-style: italic">"getEvent< T >"</span> method is the same as the one in the new call <span style="font-style: italic">"new EventBase< MyEvent >(event)"</span>.
</div>

# The Event streaming facility

The genericity of DQM4hep relies on ability to stream (read/write from/to binary) any kind of event by combining two things :

- an event streamer interface definition
- the [plugin system](plugin-system.md)

This combination allows to implement a streamer for any event type (interface definition), to compile it as a [plugin](plugin-system.md) in a shared library and load it at runtime (plugin system). This smart mechanism allowed to shape a framework that does not make any assumption on the event format and the way it reads or writes the data.

The class **EventStreamerPlugin** is the main interface for streaming event and simply provide three virtual methods to implement :

```cpp
// factory method to create an event with the correct type
virtual EventPtr createEvent() const = 0;
// write an event
virtual StatusCode write(const EventPtr &event, xdrstream::IODevice* device) = 0;
// read an event
virtual StatusCode read(EventPtr &event, xdrstream::IODevice* device) = 0;
```

All the streaming functionalities are implemented in the [xdrstream](https://github.com/DQM4HEP/xdrstream) package installed with DQM4hep packages while building the software. The class **IODevice** is the main interface for reading/writing raw data from/to a buffer. The different sub-classes of **IODevice** implement *how* the data are streamed, e.g in a buffer, in a file, etc ... Most of the time, the real implementation is a buffer.

The best way to show how this whole mechanism works and also to show how this IODevice class works, is to provide a simple example. 

Let say you have setup a device that provides measurements of temperature and pressure every seconds. We first have to define the event model for this setup. This is straight forward :

```cpp
// Defined in DeviceEvent.h

class DeviceEvent {
public:
  float m_temperature = {0.f};
  float m_pressure = {0.f};
};
```

The second step is to implement the streamer. Let's start with the class definition:

```cpp
// DeviceEventStreamer.cc
#include <dqm4hep/EventStreamer.h>
#include <DeviceEvent.h> // our event definition

using namespace dqm4hep::core;

class DeviceEventStreamer : public EventStreamerPlugin {
public:
  ~DeviceEventStreamer() {}
  EventPtr createEvent() const;
  StatusCode write(const EventPtr &event, xdrstream::IODevice* device);
  StatusCode read(EventPtr &event, xdrstream::IODevice* device);
};

// next is the implementation
```

We first include our event definition and the event streamer from DQM4hep. The class definition is really simple. Let's write the implementation, step by step.

```cpp
EventPtr DeviceEventStreamer::createEvent() const {
  return Event::create<DeviceEvent>();
}
```

The first line creates our event and the second wraps it into a DQM4hep event instance. Simple.

```cpp
StatusCode DeviceEventStreamer::write(const EventPtr &event, xdrstream::IODevice* device) {
  const DeviceEvent* devEvt = event->getEvent<DeviceEvent>();
  if(nullptr == devEvt) {
    return STATUS_CODE_INVALID_PARAMETER;
  }    
  // write temperature
  if(not XDR_TESTBIT( device->write<float>(&devEvt->m_temperature), xdrstream::XDR_SUCCESS ) ) {
    return STATUS_CODE_FAILURE;
  }  
  // write pressure
  if(not XDR_TESTBIT( device->write<float>(&devEvt->m_pressure), xdrstream::XDR_SUCCESS ) ) {
    return STATUS_CODE_FAILURE;
  }
  return STATUS_CODE_SUCCESS;
}
```

The first lines simply check if the event type is correctly passed to the streamer method. 
The next two lines simply write the temperature and pressure using the IO device. That's all ! 

For the read method, similar operations are performed.

```cpp
StatusCode DeviceEventStreamer::read(EventPtr& event, xdrstream::IODevice* device) {
  DeviceEvent* devEvt = event->getEvent<DeviceEvent>();
  if(nullptr == devEvt) {
    return STATUS_CODE_INVALID_PARAMETER;
  }  
  // read temperature
  if(not XDR_TESTBIT( pDevice->read<float>(&devEvt->m_temperature), xdrstream::XDR_SUCCESS ) ) {
    return STATUS_CODE_FAILURE;
  }    
  // read pressure
  if(not XDR_TESTBIT( pDevice->read<float>(&devEvt->m_pressure), xdrstream::XDR_SUCCESS ) ) {
    return STATUS_CODE_FAILURE;
  }
  return STATUS_CODE_SUCCESS;
}
```

The read and write are finally in their syntaxes.

That's all ! 

Only remains the plugin declaration: don't forget it !

```cpp
DQM_PLUGIN_DECL( DeviceEventStreamer, "DeviceEventStreamer" );
```

Here after is the full file:


```cpp
// DeviceEventStreamer.cc
#include <dqm4hep/EventStreamer.h>
#include <DeviceEvent.h> // our event definition

using namespace dqm4hep::core;

class DeviceEventStreamer : public EventStreamerPlugin {
public:
  ~DeviceEventStreamer() {}
  EventPtr createEvent() const;
  StatusCode write(const EventPtr &event, xdrstream::IODevice* device);
  StatusCode read(EventPtr &event, xdrstream::IODevice* device);
};

//------------------------------------------------------------------------------
//------------------------------------------------------------------------------

EventPtr DeviceEventStreamer::createEvent() const {
  return Event::create<DeviceEvent>();
}

//------------------------------------------------------------------------------

StatusCode DeviceEventStreamer::write(const EventPtr &event, xdrstream::IODevice* device) {
  const DeviceEvent* devEvt = event->getEvent<DeviceEvent>();
  if(nullptr == devEvt) {
    return STATUS_CODE_INVALID_PARAMETER;
  }    
  // write temperature
  if(not XDR_TESTBIT( device->write<float>(&devEvt->m_temperature), xdrstream::XDR_SUCCESS ) ) {
    return STATUS_CODE_FAILURE;
  }  
  // write pressure
  if(not XDR_TESTBIT( device->write<float>(&devEvt->m_pressure), xdrstream::XDR_SUCCESS ) ) {
    return STATUS_CODE_FAILURE;
  }
  return STATUS_CODE_SUCCESS;
}

//------------------------------------------------------------------------------

StatusCode DeviceEventStreamer::read(EventPtr& event, xdrstream::IODevice* device) {
  DeviceEvent* devEvt = event->getEvent<DeviceEvent>();
  if(nullptr == devEvt) {
    return STATUS_CODE_INVALID_PARAMETER;
  }  
  // read temperature
  if(not XDR_TESTBIT( pDevice->read<float>(&devEvt->m_temperature), xdrstream::XDR_SUCCESS ) ) {
    return STATUS_CODE_FAILURE;
  }    
  // read pressure
  if(not XDR_TESTBIT( pDevice->read<float>(&devEvt->m_pressure), xdrstream::XDR_SUCCESS ) ) {
    return STATUS_CODE_FAILURE;
  }
  return STATUS_CODE_SUCCESS;
}

//------------------------------------------------------------------------------

DQM_PLUGIN_DECL( DeviceEventStreamer, "DeviceEventStreamer" );
```

Finally, you can use this streamer using the *EventStreamer* class:

```cpp
// Create an event
EventPtr event = Event::create<DeviceEvent>();
// Set the event streamer name to use in the next steps 
event->setStreamerName("DeviceEventStreamer");
//
// ... work with event ...
//
// Our actual streamer toy
EventStreamer writeStreamer;
// A buffer with 1 Mo to start
xdrstream::BufferDevice writeBuffer(1024*1024);
// Write the event in the buffer
writeStreamer.writeEvent(event, &writeBuffer);

// This reads back our event from the same raw buffer
EventStreamer readStreamer;
xdrstream::BufferDevice readBuffer(writeBuffer.getBuffer(), writeBuffer.getPosition(), true);
EventPtr eventBack = nullptr;
readStreamer.readEvent(eventBack, &readBuffer);
```

Note that for the last line, the event streamer name is read on-the-fly in the buffer in order to find the correct plugin to use for reading out event. 

For a more complete documentation on xdrstream, please look on the [project page](https://github.com/DQM4HEP/xdrstream).

# Reading events from files

As the DQM4hep framework is "event implementation agnostic", there is no implementation of file reader. Instead, we provide an interface that, again, can be used together with the [plugin system](plugin-system.md). The base class of all file readers is *dqm4hep::core::EventReader*. The following virtual methods have to re-implemented in order to use an instance of file reader:

```cpp
virtual core::StatusCode open(const std::string &fname) = 0;
virtual core::StatusCode skipNEvents(int nEvents) = 0;
virtual core::StatusCode runInfo(core::Run &run) = 0;
virtual core::StatusCode readNextEvent() = 0;
virtual core::StatusCode close() = 0;
```

The `open()` function, as it suggests, instruct your reader to open a new file, and the `close()` method to close it. The `skipNEvents()` asks your reader to skip the N next events from the current position in your file. If your internal event reader does not provide such a function, it is still possible to read the N next event and throw them: 

```cpp
StatusCode MyReader::skipNEvents(int nevents) {
  for(int e=0 ; e<nevents ; e++) {
    // hypotetical function to read an event from your file
    MyEventType* event = this->performReaderEvent(); 
  }
  return STATUS_CODE_SUCCESS;
}
```  

Very often, event data model libraries provides a mechanism to efficiently skip N events from a file without having to unpack events one by one as it done in the example above.
The `runInfo()` instruct the reader to find the run information related to the file currently being read. If such an information is not available, it is still possible to provide dummy information.

```cpp
StatusCode MyReader::runInfo(Run &runInfo) {
  runInfo.setRunNumber( this->fileRunNumber() );
  runInfo.setDetectorName( this->fileDetectorName() );
  runInfo.setStartTime( core::fromTime_t(this->fileStartOfRunTime()) );
  return STATUS_CODE_SUCCESS;
}
```

The `readNextEvent()` function is used for read a single event from the file. The read event has to be sent using the [signal](core-tools.md) `onEventRead()`. When the end of file is reached, the status code *STATUS_CODE_OUT_OF_RANGE* has to be returned.

Example:

```cpp
StatusCode MyReader::readNextEvent() {
  // hypotetical function to read an event from your file
  MyEventType* event = this->performReaderEvent();
  if(nullptr == event) {
    // Out of range means end of file is reached !
    return STATUS_CODE_OUT_OF_RANGE;
  }
  // wrap your event into a DQM4hep event
  EventPtr eventPtr = Event::create<MyEventType>(event);
  // send the signal to all listeners
  onEventRead().emit(eventPtr);
  return STATUS_CODE_SUCCESS;
}
```

Here after is a typical use of the event reader class:

```cpp
// Our event consumer function
void printEvent(EventPtr event) {
  dqm_info( "Read event no {0}", event->getEventNumber() );
}

auto eventReader = PluginManager::instance()->create<EventReader>("DatEventReader");
// open a file
eventReader->open("superfile.dat");
// Read run info
Run runInfo;
eventReader->runInfo(runInfo);
dqm_info( "Read run info : {0}", dqm4hep::core::typeToString(runInfo) );
// optional: skip 2 first events
eventReader->skipNEvents(2);
eventReader->onEventRead().connect(&printEvent);

while(1) {
  auto code = eventReader->readNextEvent();
  if(STATUS_CODE_OUT_OF_RANGE == code) {
    dqm_info( "Reached end of file" );
    break;
  }
  if(STATUS_CODE_SUCCESS != code) {
    dqm_error( "Error while reading file: {0}", statusCodeToString(code) );
    break;
  }
}
eventReader->close();
```

# A builtin event type: GenericEvent

In case you are working on a small setup and you don't want to write your own event streamer, you can use our built-in event implementation (**GenericEvent**) and its associated streamer (**GenericEventStreamer**). The event holds maps of different types: integer, float, double and string.

As usual, an example is better than a long text:

```cpp
GenericEvent event;

std::vector<float> temperature = {21.5};
std::vector<float> pressure = {1.2};

// store values
event.setValues( "T", temperature );
event.setValues( "P", pressure );

// get values
std::vector<float> temp, pres;
event.getValues( "T", temp );
event.getValues( "P", pres );

dqm_info( "Temperature is {0}", temp[0] );
dqm_info( "Pressure is {0}", pres[0] );
```

The associated streamer plugin is declared with the name *"GenericEventStreamer"*.
