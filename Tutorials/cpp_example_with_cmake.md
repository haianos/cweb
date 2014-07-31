---
layout: page
---
Contents
-----------
{:.no_toc}

* toc here
{:toc}

Example: C++ data in ubx
========================

Prerequisites
-------------

Generating a package
--------------------

### Design the model for the ubx package. 

A package contains a set of blocks and types, required in your application.


{% highlight lua %}
return pkg
{
  name="cpp_transfer",
  path="../",
      
  dependencies = {
    --    no dependencies...
  },
  
  types = {
-- define the data type you want to share, e.g. a struct cpp_data
    { name="cpp_data", dir="types" },
  },
  
  blocks = {
-- define the blocks of your package, e.g. a sender and a receiver block
    { name="cpp_sender", file="cpp_sender_block.lua", src_dir="src" },
    { name="cpp_receiver", file="cpp_receiver_block.lua", src_dir="src" },
  },
  
  libraries = {
-- define the libraries. Preferably 1 library per block
    { name="cpp_sender", blocks={"cpp_sender"} },
    { name="cpp_receiver", blocks={"cpp_receiver"} },
  },
}
{% endhighlight %}

### Design the model for the ubx blocks.

{% highlight lua %}
~/workspace/microblx_cmake$ vim cpp_sender_block.lua
return block
{
      name="cpp_sender",
      meta_data="A sender c-block which can handle cpp data",
      port_cache=true,

      types = {
	-- define types this blocks uses
	 { name="cpp_data", class='struct' },
      },

      configurations= {
	-- add configuration data here. e.g. parameters to set at init
	-- { name="image_width", type_name="int", len=1 },
      },

      ports = {
	-- define the ports for this block
	 { name="output", out_type_name="struct cpp_data", out_data_len=1, doc="output port for cpp data" },
      },
      -- define which operations this block implements
      operations = { start=true, stop=true, step=true },

      cpp = true,
}
{% endhighlight %}

{% highlight lua %}
~/workspace/microblx_cmake$ vim cpp_receiver_block.lua
return block
{
      name="cpp_receiver",
      meta_data="A receiver c-block which can handle cpp data",
      port_cache=true,

      types = {
	-- define types this blocks uses
	 { name="cpp_data", class='struct' },
      },

      configurations= {
	-- add configuration data here. e.g. parameters to set at init
	-- { name="image_width", type_name="int", len=1 },
      },

      ports = {
	-- define the ports for this block
	 { name="input", in_type_name="struct cpp_data", in_data_len=1, doc="input port for cpp data" },
      },
      -- define which operations this block implements
      operations = { start=true, stop=true, step=true },

      cpp = true,
}
{% endhighlight %}

Note: the flag "cpp = true" is used to generate .cpp and .hpp files instead of .c and .h files.

### Generate the package!

{% highlight lua %}
~/workspace/microblx_cmake$ ./generate_pkg.lua -s cpp_transfer_pkg.lua
missing output directory (-d), using default from model
    generating ..//cpp_transfer/src/cpp_sender.hpp
    generating ..//cpp_transfer/src/cpp_sender.cpp
    generating ..//cpp_transfer/src/cpp_sender.usc
    generating ..//cpp_transfer/src/cpp_receiver.hpp
    generating ..//cpp_transfer/src/cpp_receiver.cpp
    generating ..//cpp_transfer/src/cpp_receiver.usc
    export models in ..//cpp_transfer/models
{% endhighlight %}

Note: if the -d option is omitted, the script will use the directory defined in the model.

Implement ubx types and blocks
------------------------------

In this step you have to fill in the generated stubs for your types and blocks.


### Implement the types

First, have a look at the types directory and the generated header files (one for each type you defined).
In our example, two files have been generated:

{% highlight lua %}
~/workspace/microblx_cmake$ cd ../cpp_transfer/types/
jphilips@CIB-11-009:~/workspace/cpp_transfer/types$ ls
cpp_data.h  cpp_transfer_types.c
{% endhighlight %}

The cpp_data.h file defines the model implementation of the type you want to share between ubx blocks, while the cpp_transfer_types.c file defines ubx functions to register all the types in this directory. The former still requires some editing, while the latter should be fine in most cases.

{% highlight lua %}
~/workspace/cpp_transfer/types$ vim cpp_data.h 
/* generated type stub, extend this struct with real information */

struct cpp_data {
	/* This should be generated in the future */
        const char model[20]; // This should be a URL to the data model
	const char uid[10]; // This should be a unique id to identify this data type
	const char meta_model[20]; // This should be a URL to the meta model
	const char data[10]; // the actually data. In our example, just 10 random characters.
};
{% endhighlight %}

### Implement the blocks

In this very basic example, we will only edit the step function of the sender and receiver block and set the license at the top

{% highlight lua %}
UBX_MODULE_LICENSE_SPDX(GPL-2.0+)

...

/* step */
void cpp_sender_step(ubx_block_t *b)
{
        struct cpp_sender_info *inf = (struct cpp_sender_info*) b->private_data;
	/* get cpp object here and convert it to c data array */
	struct cpp_data data;
	strcpy(data.model, "image array");
	strcpy(data.uid,"image");
	strcpy(data.meta_model,"c array");
	strcpy(data.data,"test_data");

	write_output(inf->ports.output, &data);
}
{% endhighlight %}

In the receiver block we also include iostream

{% highlight lua %}
#include <iostream>
using namespace std;

UBX_MODULE_LICENSE_SPDX(GPL-2.0+)

...

/* step */
void cpp_receiver_step(ubx_block_t *b)
{
        struct cpp_receiver_info *inf = (struct cpp_receiver_info*) b->private_data;
	struct cpp_data dat;
	read_input(inf->ports.input, &dat);
	cout << "MODEL: " << dat.model << endl;
	cout << "UID: " << dat.uid << endl;
	cout << "META MODEL: " << dat.meta_model << endl;
	cout << "DATA: " << dat.data << endl;

}
{% endhighlight %}

Note: At the moment, there is still a problem with type registration and to avoid runtime errors remove in either cpp_sender.hpp or cpp_receiver.hpp the type definition:

From:

{% highlight lua %}
ubx_type_t types[] = {
        def_struct_type(struct cpp_data, &cpp_data_h),
        { NULL },
};
{% endhighlight %}

To:

{% highlight lua %}
ubx_type_t types[] = {};
{% endhighlight %}

### Build with CMAKE

Note: In ccmake, set the install path of your package libraries. E.g. ~/workspace/install

{% highlight bash %}
~/workspace/cpp_transfer$ mkdir build && cd build/
~/workspace/cpp_transfer/build$ cmake ../
~/workspace/cpp_transfer/build$ ccmake ../
~/workspace/cpp_transfer/build$ make
[ 25%] Built target gen_hexarr
[ 50%] Building CXX object CMakeFiles/cpp_receiver.dir/src/cpp_receiver.cpp.o
Linking CXX shared library cpp_receiver.so
[ 50%] Built target cpp_receiver
[ 75%] Building CXX object CMakeFiles/cpp_sender.dir/src/cpp_sender.cpp.o
Linking CXX shared library cpp_sender.so
[ 75%] Built target cpp_sender
Scanning dependencies of target cpp_transfer_types
[100%] Building C object CMakeFiles/cpp_transfer_types.dir/types/cpp_transfer_types.c.o
Linking C shared library cpp_transfer_types.so
[100%] Built target cpp_transfer_types
~/workspace/cpp_transfer/build$ make install
[ 25%] Built target gen_hexarr
[ 50%] Built target cpp_receiver
[ 75%] Built target cpp_sender
[100%] Built target cpp_transfer_types
Install the project...
-- Install configuration: ""
-- Installing: /home/jphilips/workspace/install/lib/microblx/types/cpp_transfer_types.so
-- Set runtime path of "/home/jphilips/workspace/install/lib/microblx/types/cpp_transfer_types.so" to "/home/jphilips/workspace/microblx/src"
-- Installing: /home/jphilips/workspace/install/include/microblx/types/cpp_data.h.hexarr
-- Installing: /home/jphilips/workspace/install/include/microblx/types/cpp_data.h
-- Installing: /home/jphilips/workspace/install/lib/microblx/blocks/cpp_sender.so
-- Set runtime path of "/home/jphilips/workspace/install/lib/microblx/blocks/cpp_sender.so" to "/home/jphilips/workspace/microblx/src"
-- Installing: /home/jphilips/workspace/install/share/microblx/cmake/cpp_sender-block.cmake
-- Installing: /home/jphilips/workspace/install/share/microblx/cmake/cpp_sender-block-noconfig.cmake
-- Installing: /home/jphilips/workspace/install/lib/microblx/blocks/cpp_receiver.so
-- Set runtime path of "/home/jphilips/workspace/install/lib/microblx/blocks/cpp_receiver.so" to "/home/jphilips/workspace/microblx/src"
-- Installing: /home/jphilips/workspace/install/share/microblx/cmake/cpp_receiver-block.cmake
-- Installing: /home/jphilips/workspace/install/share/microblx/cmake/cpp_receiver-block-noconfig.cmake
{% endhighlight %}

Running the example
-------------------

### Create USC script

First we need to create a script to load all the blocks and connect the correct ports
{% highlight lua %}
~/workspace/cpp_transfer$ vim cpp_transfer.usc
-- -*- mode: lua; -*-

return bd.system {
   imports = {
      "std_types/stdtypes/stdtypes.so",
      "std_blocks/ptrig/ptrig.so",
      "std_blocks/lfds_buffers/lfds_cyclic.so",
      "std_blocks/hexdump/hexdump.so",
      "blocks/cpp_sender.so",
      "blocks/cpp_receiver.so",
   },
   
   blocks = {
      {name="ptrig1", type="std_triggers/ptrig"},
      {name="fifo1", type="lfds_buffers/cyclic"},
      {name="cpp_sender1", type="cpp_sender"},
      {name="cpp_receiver1", type="cpp_receiver"},
   },
   
   connections = {
-- connect the cpp_sender block with the cpp_receiver block through a fifo block
      {src="cpp_sender1.output", tgt="fifo1"},
      {src="fifo1", tgt="cpp_receiver1.input"},
   },
   
   configurations = {
      { name="fifo1", config={type_name="struct cpp_data", buffer_len=1}},
      { name="ptrig1", config={ period={sec=1,usec=0}, 
                                trig_blocks={ {b="#cpp_sender1", num_steps=1, measure=0},
                                              {b="#cpp_receiver1", num_steps=1, measure=0} }}}
   },
}
{% endhighlight %}

### Create launch script (optional)

{% highlight bash %}
~/workspace/cpp_transfer$ vim run.sh
#!/bin/bash
exec $UBX_ROOT/tools/ubx_launch -webif 8888 -c cpp_transfer.usc
{% endhighlight %}

### Launch the application!

Now, start ubx!

{% highlight bash %}
~/workspace/cpp_transfer$ ./run.sh
LuaJIT 2.0.2 -- Copyright (C) 2005-2013 Mike Pall. http://luajit.org/
Environment setup...
    UBX_ROOT: /home/jphilips/workspace/microblx
    UBX_MODULES: /home/jphilips/workspace/install/lib/microblx
importing 6 modules... 
    /home/jphilips/workspace/microblx/std_types/stdtypes/stdtypes.so
    /home/jphilips/workspace/microblx/std_blocks/ptrig/ptrig.so
    /home/jphilips/workspace/microblx/std_blocks/lfds_buffers/lfds_cyclic.so
    /home/jphilips/workspace/microblx/std_blocks/hexdump/hexdump.so
    /home/jphilips/workspace/install/lib/microblx/blocks/cpp_sender.so
    /home/jphilips/workspace/install/lib/microblx/blocks/cpp_receiver.so
importing modules completed
instantiating 4 blocks... 
    ptrig1 [std_triggers/ptrig]
    fifo1 [lfds_buffers/cyclic]
    cpp_sender1 [cpp_sender]
    cpp_receiver1 [cpp_receiver]
instantiating blocks completed
launching block diagram system in node node-20140414_111110
    preprocessing configs
    resolved block #cpp_sender1
    resolved block #cpp_receiver1
configuring 2 blocks... 
    fifo1 with {buffer_len=1,type_name="struct cpp_data"}
    ptrig1 with {trig_blocks={ 
                   {b=cpp_sender1 (type=cblock, state=preinit, prototype=cpp_sender),num_steps=1,measure=0},
                   {b=cpp_receiver1 (type=cblock, state=preinit, prototype=cpp_receiver),num_steps=1,measure=0} }, 
                 period={usec=0,sec=1} })
ptrig_handle_config: ptrig1 config: period=1s:0us, policy=SCHED_OTHER, prio=0, stacksize=0 (0=default size)
configuring blocks completed
creating 2 connections... 
    cpp_sender1.output -> fifo1 (iblock)
    fifo1 (iblock) -> cpp_receiver1.input
creating connections completed
starting up webinterface block (port: 8888)
loaded request_handler()
JIT: ON CMOV SSE2 SSE3 SSE4.1 fold cse dce fwd dse narrow loop abc sink fuse
> 
{% endhighlight %}


- Browse to [http://localhost:8888](http://localhost:8888) to start the blocks...
- Init and start the cpp_receiver1 and cpp_sender1 block
- Init and start the ptrig1 block

If all is well, you should receive following output:
{% highlight bash %}
...
MODEL: image array
UID: image
META MODEL: c array
DATA: test_data
MODEL: image array
UID: image
META MODEL: c array
DATA: test_data
MODEL: image array
UID: image
META MODEL: c array
DATA: test_data
...
{% endhighlight %}

Generate pure C application (from ubx system composition file!)
-----------------------------------
(note: still under development. Steps like checkout of dev branch, manually file copying and CMakeFile editing will be automated later on)

First, checkout dev branch of microblx_cmake package
{% highlight bash %}
git checkout dev
{% endhighlight %}

Then, generate the source file for the application with
{% highlight bash %}
./generate_capp -o cpp_transfer_app -f <path-to-cpp_transfer>/cpp_transfer.usc --webif
{% endhighlight %}

and you will find the source as "cpp_transfer_app.c".

As last step, it is necessary to compile the source, by copying the source in one project and adding the following to your CMakeFiles.txt
{% highlight bash %}
add_executable(cpp_transfer_app src_bin/cpp_transfer_app.c)
target_link_libraries(cpp_transfer_app ${UBX_LIBRARIES})
add_dependencies(cpp_transfer_app gen_hexarr)
{% endhighlight %}
Of course, it is possible to do the same if you are using a plain Makefile system.

Compile everything and you should obtain an executable "cpp_transfer_app"

By launch it, you obtain
{% highlight bash %}
All modules have been loaded!
All modules have been created!
ptrig_handle_config: ptrig1_1 config: period=0s:0us, policy=SCHED_OTHER, prio=0, stacksize=0 (0=default size)
All blocks have been initialized!
ptrig_start:  
All blocks have been started!
loaded request_handler()
webif block lauched on port 8888
Everything is up and running! hit enter to quit
{% endhighlight %}
and you can navigate, as usual, to the web interface.