[![Build Status](https://travis-ci.org/Pelagicore/gdbus-codegen-glibmm.svg?branch=master)](https://travis-ci.org/Pelagicore/gdbus-codegen-glibmm)

This is a code generator for generating D-Bus stubs and proxies from XML
introspection files. The generated stubs and proxies are implemented using
glibmm and giomm.

This generator is based on the gdbus-codegen code generator which ships with
glib.

## Installing
`setup.py` is used for installing `gdbus-codegen-glibmm`. This is a Setuptools
script, and can be invoked according to the [developer's
guide](https://setuptools.readthedocs.io/en/latest/setuptools.html#developer-s-guide).
In a nutshell, `python3 ./setup.py install` with sufficient priveleges should 
get you going.

Once installed, the `gdbus-codegen-glibmm` command should be available in the
bin directory of your selected prefix, and the required python modules should
be installed in the python path.

## Running
The code generator is a command line executable, suitable for running either
manually from a terminal or via a build-system such as CMake (there's an
example of this below).

The following parameters are supported by the `gdbus-codegen-glibmm` code
generator executable:

* -h

  Show a short help, detailing the command line parameters of the code generator

* --interface-prefix=PREFIX1,PREFIX2

  Comma-separated list of prefix-strings to strip from the generated C++ functions and classes. The D-Bus names in the D-Bus introspection XML files are used to generate namespaces in the generated C++ code, roughly a dot (.) is translated to a double colon (::). As an example, the function `org.foo.Bar.Baz()`, in the name `org.foo.Bar` will be used to generate the `org::foo::Bar::Baz(void)` function in the `Bar` class of the `org::foo` namespace. This might be too verbose for long names, and `--interface-prefix` can be used to prune the `org.foo` part from the name, resulting in the shorter `Bar::Baz(void)` C++ function (and non-namespaced class). Beware that this may cause name collisions if several interfaces with functions of the same names are used in the same D-Bus introspection XML files.

* --cpp-namespace=NAMESPACE

  A string to prepend to the namespaces of the generated functions.

* --errors-namespace=ERROR\_NAMESPACE

  The namespace where error classes should be generated. Can contain `::`.

* --generate-cpp-code=OUTFILES

  Path and prefix of the file names to generate for the proxy and stub generated by the code generator. The filename prefix is suffixed by `_stub.[h|cpp]`, `_proxy.[h|cpp]` and `_common[.h|cpp]`. The file names generated usign this path are also used for inclusion of headers in the generated code, so it is not recommended to rename the files generated using the path supplied here.

* Following parameters

  List of D-Bus introspection XML files. These files are used to describe the D-Bus interfaces. Several files can be supplied. The interfaces from all files will be gathered and then emitted in the same output headers and cpp-files. See [the introspection chapter of the the D-Bus specification](http://dbus.freedesktop.org/doc/dbus-specification.html#introspection-format) by freedesktop for more information on the format of the D-Bus introspection XML files.

###  Example invocation
```bash
dbus-codegen-glibmm --generate-cpp-code=${HOME}/temperature-service-example/build/generated/temperature-service
                    ${HOME}/temperature-service-example/temperature-service.xml
```
This will create the following files in `${HOME}/temperature-service-example/build/generated/`:
```bash
temperature-service_common.cpp
temperature-service_common.h
temperature-service_proxy.cpp
temperature-service_proxy.h
temperature-service_stub.cpp
temperature-service_stub.h
```

### Generation of error classes
`gdbus-codegen-glibmm` can generate classes for error handling: these classes
will inherit from `Glib::Error`, and can be used both in the backend and in
the client library.

Errors are declared by inserting annotations as child elements of a D-Bus
interface:
```xml
<!DOCTYPE node PUBLIC "-//freedesktop//DTD D-BUS Object Introspection 1.0//EN"
                      "http://www.freedesktop.org/standards/dbus/1.0/introspect.dtd">
<node>
  <interface name="com.example.MyService">
    ...
    <!--
      Invalid parameters (this comment will also end up in the generated code)
    -->
    <annotation name="org.gdbus.glibmm.Error" value="com.example.Error.InvalidParams" />

    <!-- The backend ran out of memory -->
    <annotation name="org.gdbus.glibmm.Error" value="com.example.Error.OutOfMemory" />
    ...
```

## Implementing a stub
First, a D-Bus interface must be specified in XML. We will use the following:
```XML
<?xml version="1.0" encoding="UTF-8" ?>
<node name="/org/foo/Bar">
	<interface name="org.foo.Bar">
		<method name="Baz">
		</method>
	</interface>
</node>
```

In this tutorial we will keep the generated code in a subdirectory called `generated`, so go ahead and `mkdir generated`  before continuing. The following invocation will generate a stub (and also a proxy, but this is unused in this section):
`gdbus-codegen-glibmm --generate-cpp-code=generated/bar bar.xml`

The generated stub is a virtual class, designed to be implemented by a concrete
C++ class. In our example, the following will suffice:

```cpp
#include "bar_stub.h"

class BarImpl : public org::foo::Bar {
public:
    // Called wben org.foo.bar.Baz() is invoked
    void Baz (BarMessageHelper msg) override {
        // Return void
        msg.ret();
    }
};

int main(int argc, char **argv) {
    // Initialize Glib and Gio
    Glib::init();
    Gio::init();

    // Instantiate and run the main loop
    Glib::RefPtr<Glib::MainLoop> ml = Glib::MainLoop::create();

    // Connect to the system bus and acquire our name
    BarImpl bi;
    guint connection_id = Gio::DBus::own_name(
            Gio::DBus::BUS_TYPE_SESSION,
            "org.foo.Bar",
            [&](const Glib::RefPtr<Gio::DBus::Connection> &connection,
                const Glib::ustring & /* name */) {
                g_print("Connected to bus.\n");
                if (bi.register_object(connection, "/org/foo/Bar") == 0)
                    ml->quit();
            },
            [&](const Glib::RefPtr<Gio::DBus::Connection> & /* connection */,
                const Glib::ustring & /* name */) {
                g_print("Name acquired.\n");
            },
            [&](const Glib::RefPtr<Gio::DBus::Connection> & /* connection */,
                const Glib::ustring & /* name */) {
                g_print("Name lost.\n");
                ml->quit();
            });

    ml->run();

    Gio::DBus::unown_name(connection_id);

    return 0;
}
```

The example above doesn't do much in terms of functionality, but it shows how
to implement a very simple stub. Properties are supported in a similar fashion,
where the stub implements `_set()` and/or `_get()` functions (depending on the
accessibility of the property).

Signals are simply connected to, and no implementation code needs to be
written.

The above example can be compiled using the following command:
```bash
clang++ -I . -I generated `pkg-config --cflags --libs glibmm-2.4 giomm-2.4`
            generated/bar_common.cpp
            generated/bar_stub.cpp
            barimpl.cpp
```
Just update the filenames accordingly.

## Implementing a proxy
In order to implement a proxy, run the same generation commands as in the
previous example, and use the `_proxy.[h|cpp]` files instead of the
`_stub.[h|cpp]`. The output from the previous execution of the generator will
also work, so there is no need to re-run if the files are already available.

The following file shows the proxy corresponding to the stub above:

```cpp
#include "bar_proxy.h"

Glib::RefPtr<org::foo::Bar> proxy;

void on_baz_finished(const Glib::RefPtr<Gio::AsyncResult> &result) {
    proxy->Baz_finish(result);
}

void proxy_created(const Glib::RefPtr<Gio::AsyncResult> result) {
    proxy = org::foo::Bar::createForBusFinish(result);
    proxy->Baz(sigc::ptr_fun(&on_baz_finished));
}

int main(int argc, char **argv) {
    Glib::init();
    Gio::init();

    org::foo::Bar::createForBus(Gio::DBus::BUS_TYPE_SESSION,
                                Gio::DBus::PROXY_FLAGS_NONE,
                                "org.foo.Bar",
                                "/org/foo/Bar",
                                sigc::ptr_fun(&proxy_created));

    Glib::RefPtr<Glib::MainLoop> ml = Glib::MainLoop::create();
    ml->run();

    return 0;
}
```

It can be compiled in a similar fashion as the previous example.

## CMake integration
Running the code generator from CMake can be done using the following snippet:

```cmake
SET (CODEGEN gdbus-codegen-glibmm)
SET (INTROSPECTION_XML ${CMAKE_SOURCE_DIR}/bar.xml)

SET (GENERATED_STUB
    ${CMAKE_BINARY_DIR}/generated/bar_stub.cpp
    ${CMAKE_BINARY_DIR}/generated/bar_stub.h
    ${CMAKE_BINARY_DIR}/generated/bar_common.cpp
    ${CMAKE_BINARY_DIR}/generated/bar_common.h
)

ADD_CUSTOM_COMMAND (OUTPUT ${GENERATED_STUB}
                    COMMAND mkdir -p ${CMAKE_BINARY_DIR}/generated/
                    COMMAND ${CODEGEN} --generate-cpp-code=${CMAKE_BINARY_DIR}/generated/bar
                                        ${INTROSPECTION_XML}
                    DEPENDS ${INTROSPECTION_XML}
                    COMMENT "Generate the stub for the test program")
```

The usage of the `$GENERATED_STUB` files will trigger the execution of the code
generator.

## License and copyright
This README file is Copyright &copy; 2018 Luxoft Sweden AB

SPDX-License-Identifier: CC-BY-SA-4.0

[![License: CC BY-SA 4.0](https://img.shields.io/badge/LICENSE-CC--BY--SA--4.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc/4.0/)

The python code is 
- Copyright &copy; 2008-2011 Red Hat, Inc.
- Copyright &copy; 2018 Luxoft Sweden AB
Source code licensed under the LGPL 2.1 (please see source code headers for more.)
