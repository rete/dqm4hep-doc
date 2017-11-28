
# The plugin system in a nutshell

The plugin system component of the DQM4hep core package has a central place in the software design. The genericity of DQM4hep relies on plugins, that can be loaded at runtime. Parts of the framework can thus easily changed at runtime by specifying a different set of plugins to use.

This system is composed of only two classes and a registration macro :

- **dqm4hep::core::PluginManager**, a singleton class holding the list of registered plugins. It is the main user interface, responsible for :
    * loading shared libraries (.so or .dylib).
    * owning the plugin prototypes loaded in the shared libraries.
    * creating instances of plugins on user query.
- **dqm4hep::core::Plugin**, an internal class defining the interface for user plugins.
- The **DQM_PLUGIN_DECL()** registration macro, called in user code to register plugins in shared libraries.

In DQM4hep, a lot of components are implemented as plugins, such as [event streamer](event-streaming.md) and [quality test](quality-test.md) classes.

# Writing a plugin

A plugin is a simple c++ class that is declared in a compiled component, e.g shared library or executable. No inheritance is needed to declare a plugin. The macro **DQM_PLUGIN_DECL()** takes care of everything for you. As always, an example is better than a long documentation :

Header file: 
```cpp
#include <iostream>

class MyPlugin {
public:
  MyPlugin() {};
  void helloWorld(); 
};
```

Source file:
```cpp
#include <MyPlugin.h>
#include <dqm4hep/PluginManager.h>

MyPlugin::helloWorld() { 
  std::cout << "Hello world !" << std::endl; 
}

// This is where the magic happens !
DQM_PLUGIN_DECL( MyPlugin, "MyPlugin" );
```

Most of the code is just a simple c++ class that does nothing else than saying "Hello world !". In the source file, the *PluginManager.h* include is needed to declare your plugin in the registry. The last line of the source actually does the plugin declaration. The first argument is the class name itself and the second, the plugin name that can be different from the class name. 

<div class="warning-msg">
  <i class="fa fa-warning"></i>
  The registration macro performs operations that require a simple class name, for example without namespace specifier in the argument. You can use <span style="font-style: italic">'typedef'</span> or <span style="font-style: italic">'using'</span> keywords to fix the problem.
</div>

The source file can be compiled in a shared library or within a binary and loaded at runtime (see next section).

# The PluginManager API

This section explains how the declared plugins are loaded and accessed from the **PluginManager** class.

First of all, we need to load the shared libraries in which the plugin are implemented. This can be done in different ways :

```cpp
#include <dqm4hep/PluginManager.h>

using namespace dqm4hep::core;

int main() {
  
  // get the pointer on the plugin manager
  auto mgr = PluginManager::instance();
  
  // load a single library
  THROW_RESULT_IF( STATUS_CODE_SUCCESS, != , mgr->loadLibrary( "libMyPlugin.so" ) );
  
  // load a set of libraries
  THROW_RESULT_IF( STATUS_CODE_SUCCESS, != , mgr->loadLibraries( "libMyPlugin.so:libMyPlugin2.so:libSuperman.so" ) );
  
  // load libraries from the environment variable DQM4HEP_PLUGIN_DLL
  THROW_RESULT_IF( STATUS_CODE_SUCCESS, != , mgr->loadLibraries() );
  
  return 0;
}
```

<div class="info-msg">
  <i class="fa fa-info-circle"></i>
  Most of the binaries provided the DQM4hep packages load the shared libraries using the third option, with the environment variable <span style="font-style: italic">DQM4HEP_PLUGIN_DLL</span>, at the beginning of the main. 
</div>

Before running a DQM4hep program it is usually a good idea to export <span style="font-style: italic">DQM4HEP_PLUGIN_DLL</span> with your plugin libraries inside :

```shell
export DQM4HEP_PLUGIN_DLL=libSpiderman.so:libThor.so:libIronman.so
```

The following piece of code assume you have implemented the "Hello world" example above and compiled it in a library called libMyPlugin.so.


```cpp
#include <dqm4hep/PluginManager.h>
#include <MyPlugin.h>

using namespace dqm4hep::core;

int main() {
  
  auto mgr = PluginManager::instance();
  THROW_RESULT_IF( STATUS_CODE_SUCCESS, != , mgr->loadLibraries() );

  // check for registered plugins
  if ( mgr->isPluginRegistered( "MyPlugin" ) ) {

    // Create an instance of your plugin
    MyPlugin* plugin = mgr->create<MyPlugin>( "MyPlugin" );

    // Use it !!
    plugin->helloWorld();
    
    // The ownership belongs to the caller
    delete plugin;    
  }

  // Get all registered plugin names (string vector)
  auto pluginNames = mgr->getPluginNames();
  
  // Dump all registered plugins in the console
  mgr->dump();

  return 0;
}
```

Note that in this simple example, the file MyPlugin.h has to be included to make this working. The following code shows a more complete example of how to define a simple interface in one library and implement different plugins in an other library.

```cpp
// Shape.h
class Shape {
  virtual ~Shape() {}
  virtual void draw() = 0;
};
```

This is our interface definition. A simple **Shape** class with an abstract draw() method. This file is part of our library.

```cpp
// Square.cc 
#include <dqm4hep/Logging.h>
#include <dqm4hep/PluginManager.h>
#include <Shape.h>

class Square : public Shape {
  void draw();
};

void Square::draw() {
  dqm_info( "Drawing a square !!" );
}

DQM_PLUGIN_DECL( Square , "SquareShapePlugin" );
```

```cpp
// Circle.cc 
#include <dqm4hep/Logging.h>
#include <dqm4hep/PluginManager.h>
#include <Shape.h>

class Circle : public Shape {
  void draw();
};

void Circle::draw() {
  dqm_info( "Drawing a circle !!" );
}

DQM_PLUGIN_DECL( Circle , "CircleShapePlugin" );
```

These two files implement our **Shape** interface and just make a print out. They are both compiled in the same library called *libShapePlugins.so*

<div class="info-msg">
  <i class="fa fa-info-circle"></i>
  Note that it is not needed to define a header file to implement a plugin !
</div>

Now, we define our main in this way : 

```cpp
#include <dqm4hep/PluginManager.h>
#include <Shape.h>

using namespace dqm4hep::core;

int main(int argc, char** argv) {
  
  std::string pluginName( argv[1] );
  
  auto mgr = PluginManager::instance();
  THROW_RESULT_IF( STATUS_CODE_SUCCESS, != , mgr->loadLibraries() );

  Shape* shape = mgr->create<Shape>( pluginName );
  shape->draw();
  delete shape;

  return 0;
}
```

Compile it as **shape_example** and run it like this : 

```shell
export DQM4HEP_PLUGIN_DLL=libShapePlugins.so
./shape_example CircleShapePlugin # to draw a circle
./shape_example SquareShapePlugin # to draw a square
```

Note that in this example, your executable is not aware of the existence of the two plugin implementations at compile time. The classes are loaded and hooked via the plugin manager **at runtime**.

# Known plugin interfaces in the framework

A set of interfaces have been pre-defined in DQM4hep and can be implemented as plugins:

- **dqm4hep::core::QualityTest** (*dqm4hep-core* package)
- **dqm4hep::core::EventStreamer** (*dqm4hep-core* package)
- **dqm4hep::online::AnalysisModule** (*dqm4hep-online* package)
- **dqm4hep::online::StandaloneModule** (*dqm4hep-online* package)

# Command line tools

A binary is provided to dump the registered plugins in the console, called *dqm4hep-dump-plugins*. By default, it dumps only the plugins registered in the DQMCore library, that is linked to this binary. You can try to run it simply as :

```shell
# print only the plugins registered in libDQMCore.so 
dqm4hep-dump-plugins
```

or with your own libraries to see what are the compiled plugins :

```shell
# show our two 'shape' plugins
export DQM4HEP_PLUGIN_DLL=libShapePlugins.so
dqm4hep-dump-plugins
```














