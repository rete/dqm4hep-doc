
# The Event definition

Event streaming has a central place in online setup. It defines what kind of event you will collect and analyze during data taking, as well as how you read and write your data in binary format. DQM4hep defines a simple interface to hold your data: the **EventBase< T >** class. This class holds basic properties you may want to use for your event definition. These properties are particularly redundant in High Energy Physics experiments. The list of these properties are list here after :

- **Event type** (enumerator), the event type on data taking workflow. Possible values are:
    - **UNKNOWN_EVENT**: the default flag when an event is created
    - **RAW_DATA_EVENT**: data coming from a single device, e.g one DIF or one particular sensor
    - **RECONSTRUCTED_EVENT**: a reconstructed event after going through the event builder farm. This kind of event is generally a combination of *RAW_DATA_EVENT* events
    - **PHYSICS_EVENT**: the *RAW_DATA_EVENT* and *RECONSTRUCTED_EVENT* events usually contain raw data (binary buffers very specific to readout chips). The *PHYSICS_EVENT* type is defined for events containing data that have been translated into "physicist readable" data, e.g calorimeter hits, tracker hits or reconstructed particles.
    - **CUSTOM_EVENT**: an additional flag in case none of the previous types fits in your case
- **Source** (string), the name of source (e.g device) that have created the event.
- **Time stamp**, the time stamp of the event creation. The type of this property is *std::chrono::system_clock::time_point*.
- **Event size** (uint32_t), the size in byte of the event being serialized. See next sections for more information
- **Run number** (uint32_t), the run number in which the event was created.
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

// The handled event can be retreived
MyEvent *eventget = dqmEvent->getEvent<MyEvent>();
assert( event == eventget );
```

Note the getEvent< T >() method use in last place. This method casts the *Event* object to an *EventBase< T >* object and access the event stored in this class, something like :

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

# The event streaming facility

The genericity of DQM4hep relies on ability to stream (read and write event into binary) any kind of event by combining two things :

- an event streamer interface definition
- the [plugin system](plugin-system.md)

This combination allows to implement a streamer for any event type (interface definition), to compile it as a [plugin](plugin-system.md) in a shared library and load it at runtime (plugin system). This smart mechanism allowed to shape a framework that do not put any assumption on the event format and the way it read or write the data.

The class dqm4hep::core::EventStreamer is the main interface for streaming event and simply provide three virtual methods to implement :

```cpp
// factory method to create an event with the correct type
virtual Event *createEvent() const = 0;
// write an event
virtual StatusCode write(const Event *const pEvent, xdrstream::IODevice *pDevice) = 0;
// read an event
virtual StatusCode read(Event *&pEvent, xdrstream::IODevice *pDevice) = 0;
```

All the streaming functionalities are implemented in the [xdrstream](https://github.com/DQM4HEP/xdrstream) package installed with DQM4hep packages while building the software. The class **xdrstream::IODevice** is the main interface for reading/writing raw data from/to a buffer. The different sub-classes of IODevice implement *how* the data are streamed, e.g in a buffer or in a file.







