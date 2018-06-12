

# The logging tools

The logging tools are currently based on the super-fast logging library [spdlog](https://github.com/gabime/spdlog). It is faster than most of the c++ logging library, header-only, threadsafe and really easy to use.

DQM4hep defines simple macros that allows you to log from any piece of code, with different log levels. You just need to include the file *dqm4hep/Logging.h* to use them. Here is a simple example demonstrating its use :

```cpp
#include <dqm4hep/Logging.h>

int main() {
  
  dqm_debug( "This is a debug message" );
  dqm_trace( "This is a trace message" );
  dqm_info( "This is an info message" );
  dqm_warning( "This is a warning message" );
  dqm_error( "This is a error message" );
  dqm_critical( "This is a critical message" );
  
  return 0;
}
```

The spdlog library uses the [fmt library](https://github.com/fmtlib/fmt) to format optional argument in the logging macros :

```cpp
#include <dqm4hep/Logging.h>
#include <typeinfo>

int main() {
  
  int nErrors = 42;
  dqm_error( "This is a error message that occured {0} times", nErrors );
  
  try {
    throw std::runtime_error( "Invalid pointer !" );
  }
  catch(std::exception &e) {
    dqm_warning( "Caught exception of type {0}: {1}", typeid(e).name(), e.what() );
  }
  
  return 0;
}
```

The DQM4hep logging component is generally configured at the begining of the each main. Different possibilities are provided by *spdlog* to log messages :

- **Simple file** : log messages in a simple file.
- **Rotating file** : log messages into rotating files. A maximum file size and number of files are defined. When a file reaches its maximum size, a new file is opened. When the maximum number of files is reached, the first file is opened and overwritten.
- **Daily file** : log messages into rotating files. A new file is opened every N hours and N minutes.
- **Console** : log message into the console.
- **Colored console** : log message into the console. The message are colored according to the log level in use.

The *dqm4hep::core::Logger* class defines simple static methods to create a logger with different configuration :

```cpp
#include <dqm4hep/Logging.h>
#include <dqm4hep/Logger.h>
#include <typeinfo>

using namespace dqm4hep::core;

int main() {
  
  // this creates a logger to log into the
  // console and into a simple file
  Logger::createLogger("my-logger", {
    Logger::coloredConsole(),
    Logger::simpleFile("logfile.log")
  });
  
  Logger::setMainLogger("my-logger");
  
  dqm_info( "This line will appear in the console and in the file logfile.log" );
  
  return 0;
}
```

In addition to the logging macros demonstrated in the first piece of code, other ones have been defined to directly log using a user specific logger :

```cpp
#include <dqm4hep/Logging.h>
#include <dqm4hep/Logger.h>

using namespace dqm4hep::core;

int main() {
  
  // this creates a logger to log into the
  // console and into a simple file
  auto logger = Logger::createLogger("my-logger", {
    Logger::coloredConsole(),
    Logger::simpleFile("logfile.log")
  });
  
  // log in the created logger
  dqm_logger_info( logger, "This line will appear in the console and in the file logfile.log" );
  
  // log into the default logger
  dqm_info( "This line will appear only in the console (main logger)" );
  
  // change logger and log again
  Logger::setMainLogger("my-logger");
  dqm_warning( "This line will appear (again) in the console and in the file logfile.log" );
  
  // switch back to the main logger
  Logger::setMainLogger("main");
  dqm_warning( "This line will appear (again) only in the console (main logger)" );
  
  return 0;
}
```

It is also possible to log with a general macro an by passing the log level by argument :


```cpp
#include <dqm4hep/Logging.h>
#include <dqm4hep/Logger.h>

using namespace dqm4hep::core;

int main() {
  
  // this creates a logger to log into the
  // console and into a simple file
  auto logger = Logger::createLogger("my-logger", {
    Logger::coloredConsole(),
    Logger::simpleFile("logfile.log")
  });
  
  // log in the created logger
  dqm_logger_log( logger, spdlog::level::info, "This is an info message in the 'my-logger' logger !" );
  
  // log into the default logger
  dqm_log( spdlog::level::error, "This is an error message in the main logger !" );
    
  return 0;
}
```

By default, in most of the binaries provided by the DQM4hep packages, the main logger is configured to be a colored console.

# Signal/slot mechanism

The DQM4hep core library provides a *dqm4hep::core::Signal* class that implements the observer pattern with an API similar to the Qt signal/slot mechanism. In contay to the Qt framework, the signal/slot mechanism described here is more likely a tool than a base framework functionality. The idea is to connect a function, or a set of functions, to process when a signal is emitted. These functions can be global functions or class methods. The following code illustrates how to use of the *dqm4hep::core::Signal* class with simple sender/receiver classes:


```cpp
#include <dqm4hep/Logging.h>
#include <dqm4hep/Signal.h>

using namespace dqm4hep::core;

/// A simple sender class
/// Send user-defined or random data using signal
class Sender {
public:
  void sendRandom() {
    int randomNumber = rand();
    std::stringstream ss; ss << "A random number : " << randomNumber;
    // Emit signal will notify all listeners 
    m_signal.emit( ss.str() );
  }
  
  void send(const std::string &data) {
    // Emit signal will notify all listeners 
    m_signal.emit( data );
  }
  
  Signal<const std::string &>& onSend() {
    return m_signal;
  }
  
private:
  Signal<const std::string &>    m_signal;
};

// Example with a global function
void print(const std::string &data) {
  dqm_info( "print function received: " + data );
}

/// A simple receiver class
class Receiver {
public:
  void logData(const std::string &data) {
    dqm_info( "Received data: {0}", data );
  }
};


int main() {
  // our main objects
  Sender sender;
  Receiver receiver;
  
  // connect the sender signal to the logData() method of the Receiver class
  sender.onSend().connect( &receiver, &Receiver::logData );
  // connect the sender signal to the global print() function
  sender.onSend().connect( &print );
  // send data will call signal.emit() and notify the receiver and the global function
  sender.sendRandom();
  sender.send( "Punk is not dead !" );
    
  return 0;
}
```

Note in this example the signature of the template parameter of the *dqm4hep::core::Signal* class :

```cpp
Signal<const std::string &>  m_signal;
``` 

As c++ is a strongly typed language, the signal template parameters signature has to be the same as the one of the method to connect. In this example, the function *logData()* has a *const std::string &* signature in argument. This example would have failed if the signature would have been different.

Note also that the *dqm4hep::core::Signal* class allows to :

- connect methods from any c++ class,
- connect global functions,
- disconnect these functions if needed,
- connect methods or functions with unlimited number of arguments. Nevertheless, the parameter types have to be specified in the template arguments of the *dqm4hep::core::Signal* object : 

```cpp
  Signal<std::string, int, int, double, float, int> aLongSignal;
``` 

# Status code and useful pre-processor macros

If you navigate through the DQM4hep code, you will notice the use of the *StatusCode* enum, the *StatusCodeException* and pre-processor macros such as *RETURN_RESULT_IF()*.
The *StatusCode* enum contains the folowing values:

```cpp
enum StatusCode {
  STATUS_CODE_SUCCESS,
  STATUS_CODE_FAILURE,
  STATUS_CODE_NOT_FOUND,
  STATUS_CODE_NOT_INITIALIZED,
  STATUS_CODE_ALREADY_INITIALIZED,
  STATUS_CODE_ALREADY_PRESENT,
  STATUS_CODE_OUT_OF_RANGE,
  STATUS_CODE_NOT_ALLOWED,
  STATUS_CODE_INVALID_PARAMETER,
  STATUS_CODE_UNCHANGED,
  STATUS_CODE_INVALID_PTR
};
```

The *StatusCodeException* exception class handle one of these status codes on creation. Together with the StatusCode enum are also defined the following pre-processor macros:

- *RETURN_RESULT_IF(code, Operator, Command)*: return the status code returned by the command if the comparison code with *code* using the operator *Operator* fails. See example below.
- *RETURN_RESULT_IF_AND_IF(code, code2, Operator, Command)*: return the status code returned by the command if the comparison code with *code* and *code2* using the operator *Operator* fails.
- *THROW_RESULT_IF(code, Operator, Command)*: throw a *StatusCodeException* with the status code returned by the command if the comparison code with *code* using the operator *Operator* fails.
- *THROW_RESULT_IF_AND_IF(code, code2, Operator, Command)*: throw a *StatusCodeException* with the status code returned by the command if the comparison code with *code* and *code2* using the operator *Operator* fails.
 
Example:

```cpp
StatusCode myFunction(int i) {
  if(i > 0) {
    return STATUS_CODE_SUCCESS;
  }
  else {
    return STATUS_CODE_INVALID_PARAMETER;
  }
}

void doIt() {
  try {
    THROW_RESULT_IF(STATUS_CODE_SUCCESS, !=, myFunction(42));
    THROW_RESULT_IF(STATUS_CODE_SUCCESS, !=, myFunction(-1)); // this will throw an exception
  }
  catch(StatusCodeException &exception) {
    dqm_error( "Caught a StatusCodeException: {0}", exception.toString() );
  }
}
```

The following message will be printed-out in the console:

```shell
myFunction(42) return STATUS_CODE_INVALID_PARAMETER
    in function: doIt
    in file:     /path/to/toto.cc line#: 75
```

These macros are particuliarly useful to debug a huge stack trace by following the function call by function name, file and line call.

# The XML parser

One of the nice components of the DQM4hep core library is the XML parser. It uses the [tinyxml](http://www.grinninglizard.com/tinyxml/) library to internally parse a XML document. On top of that comes a certain number of features that modifies the XML tree in memory after parsing listed below.

## XML parsing features

### XML file includes

The element `<include>` allows to include an other XML file in-place of the element. The attribute `ref` of the XML element points on a XML using either an absolute or relative path. If a relative path is used, it is relative to the file containing the XML include element. By default, the XML also process nested include elements, meaning that *file1.xml* can include *file2.xml* that can also includes *file3.xml*.

<div class="warning-msg">
  <i class="fa fa-warning"></i>
  No check is perform to protect against infinite recursive includes. It is users responsability to check the file consistency before using the XML file parser.
</div>

The `<include>` elements can be placed anywhere in the XML tree.
Example:

```xml
<!-- Main XML file: main.xml -->
<dqm4hep>
  <!-- Any user custom element -->
  <custom>
    <include ref="file2.xml" />
  </custom>
</dqm4hep>
```

```xml
<!-- file2.xml -->
<dqm4hep>
  <entry id="toto"/>
  <entry id="titi"/>
  <entry id="tata"/>
</dqm4hep>
```

This will result to:

```xml
<!-- Main XML file after processing -->
<dqm4hep>
  <!-- Any user custom element -->
  <custom>
    <entry id="toto"/>
    <entry id="titi"/>
    <entry id="tata"/>
  </custom>
</dqm4hep>
```

<div class="info-msg">
  <i class="fa fa-info"></i>
  Note that in the included file file1.xml the root element has to be `<dqm4hep>`. Every element contained in this root element will be **literally** copied in-place of the `<include>` element. Note also that the nested elements are processed on include directly.
</div>

### XML constants

The DQM4hep XML parser offers the possibility to define constants in the XML file and to re-use them. These constants are defined in the root element `<dqm4hep>` in dedicated section called `<constants>`. Each constant is defined in a single XML element `<constant>` labeled with a `name` and a `value`.

Example:

```xml
<dqm4hep>
  <constants>
    <constant name="MyInteger" value="42"/>
    <constant name="ATitle" value="This is a title"/>
  </constants>
</dqm4hep>
```

A constant can't be defined twice and an exception will be thrown in this case. These constants can be re-used further in the XML file inside element attributes or inside XML text using the syntax `${name}` where `name` is a constant name.

Example:


```xml
<dqm4hep>
  
  <constants>
    <constant name="MyInteger" value="42"/>
    <constant name="ATitle" value="This is a title"/>
    <!-- Constants can be used in constants definition ! -->
    <constant name="Answer" value="The answer of everything is ${MyInteger}"/>
  </constants>
  
  <!-- Constant in an element attribute -->
  <custom start="${MyInteger}"/>
  
  <!-- Constant in a text -->
  <quote> This is movie reference : ${Answer} </quote>
  
</dqm4hep>
```

### MySQL database parameter select

To be done - requires documentation on MySQL parameter database and tables ...

### Use of environment variables

Constants are a nice feature of the DQM4hep XML parser but can sometimes be a problem when users are dealing with passwords and other private sensitive data such API tokens. Environment variables are often use to deal with this problem as they are defined locally in your shell and thus not accessible you opening and reading the XML file in an editor. The XML parser can use environment variables by using a similar syntax as for constants (see above) with `$ENV{var}` where `var` is an environment variable.

Example:

```xml
<dqm4hep>
  <!-- Oops ! Password hardcoded... -->
  <login user="Arthur" password="Cuillere"/>
</dqm4hep>
```

then use an environment variable instead:

```xml
<dqm4hep>
  <login user="Arthur" password="$ENV{LOGIN_PASS}"/>
</dqm4hep>
```

<div class="warning-msg">
  <i class="fa fa-warning"></i>
  Don't forget to export the referenced environment variables before using the XML parser as you will get an exception if they are not defined !
</div>

### XML for loops

The element `<for>` allows to run a simple for loop and duplicate XML elements. To keep it simple, the loop is run over integer id (`id`) from a lower value (`begin`) to an upper value (`end`) with a user defined increment (`increment`). A for loop is run over the whole content of the `<for>` element, look for the string "$FOR{*id*}" and replace it by the current loop value. The replacement operates on:

- the XML comments content
- all the XML element attributes
- the XML texts

Example:
```xml
<dqm4hep>
  <for id="count" begin="0" end="3" increment="1">
    <!-- The current value of the counter is $FOR{count} -->
    <counter value="$FOR{count}"/>
  </for>
</dqm4hep>
```

will be replaced by:
```xml
<dqm4hep>
  <!-- The current value of the counter is 0 -->
  <counter value="0"/>
  <!-- The current value of the counter is 1 -->
  <counter value="1"/>
  <!-- The current value of the counter is 2 -->
  <counter value="2"/>
  <!-- The current value of the counter is 3 -->
  <counter value="3"/>
</dqm4hep>
```

By construction, it is also possible the run nested for loops by changing the loop id:

```xml
<dqm4hep>
  <for id="x" begin="0" end="2" increment="1">
    <for id="y" begin="0" end="2" increment="1">
      <entry x="$FOR{x}" y="$FOR{y}"/>
    </for>
  </for>
</dqm4hep>
```

will be replaced by:

```xml
<dqm4hep>
  <entry x="0" y="0"/>
  <entry x="0" y="1"/>
  <entry x="0" y="2"/>
  <entry x="1" y="0"/>
  <entry x="1" y="1"/>
  <entry x="1" y="2"/>
  <entry x="2" y="0"/>
  <entry x="2" y="1"/>
  <entry x="2" y="2"/>
</dqm4hep>
```

The `<for>` element attributes possible values are summurized in table below. 
 
Attribute | Type    | Optional ? | Default value
--------- | ------- | ---------- | -------------
id        | string  | false      | - 
begin     | integer | false      | - 
end       | integer | false      | -
increment | integer | true       | 1

## XML parsing ordering

The XML file is processed in the given order:

- Includes resolution
- Constants and environment variables
- Database connections and parameter select
- For loops

This has the following effects:

1. Constants and environment variables can be used in database connections and for loops
2. As includes are processed before constants, the `ref` attribute of the `<include>` element can not refer to any constant. In this case the constant will not replaced and the file will not been included.
3. As constants are processed after includes, it possible to define constants in an included file. The `<constants>` section in the included file has to be defined in the root element and the `<include>` element in the main file has to be defined in the root element.

## Using the XMLParser class

The `XMLParser` class is part of the DQMCore distribution and can be included using the directive:

```cpp
#include <dqm4hep/XMLParser.h>
using XMLParser = dqm4hep::core::XMLParser;
```

To parse an XML file, use the method `XMLParser::parse()`:

```cpp
XMLParser parser;
try {
  parser.parse("myfile.xml");
}
catch(StatusCodeException &exception) {
  dqm_error( "Caught StatusCodeException while parsing XML file: {0}", exception.toString() );
}
```

The XML document itself can be accessed with the method `XMLParser::document()`:

```cpp
auto document = parser.document();
```

All of the features listed above are by default active but can be de-activated using dedicated method. The example below shows how to de-activate all the features listed above. 

```cpp
XMLParser parser;

// disable all includes resolution
parser.setProcessIncludes(false);

// disable nested includes
// only useful if we keep the option setProcessIncludes to true
parser.setAllowNestedIncludes(false);

// disable constants parsing
parser.setProcessConstants(false);

// disable database connect and parameters select resolution
parser.setProcessDatabase(false);

// disable environment variable resolution
parser.setAllowEnvVariables(false);

// disable for loops resolution
parser.setProcessForLoops(false);
```

If the constants parsing has been activated before parsing the XML file, the parsed constants can be accessed after parsing:

```
XMLParser parser:
parser.parse("myfile.xml");

// access all constants
auto constants = parser.constants();

for(auto &constant : constants) {
  std::cout << "constant: name=" << constant.first << ", value=" << constant.second << std::endl;
}

// access single constant
// the second argument is a fallback value if the constant doesn't exists
auto integer = parser.constant<int>("MyInteger", 42);

std::cout << "Constant MyInteger, value: " << integer << std::endl;
```

