---
layout: page
title: "My first cblock in C ( with tools! )"
---

## Contents
{:.no_toc}

* toc here
{:toc}

Goals
-------------
The tutorial shows how to implement a cblock written in __C__ with the support of an autogenerated tool. The advantage of this approach is to produce faster, error-prone cblock implementations.
The workflow consists on:

  1. Define a package model, that is a bundle which will contain your software
  2. Define the model of the cblocks we would like to implement
  3. Generate all the sources skeleton and setup the environment to compile the sources
  4. Complete the step function of the cblocks in the source
  5. Compile and run it!

Thanks for the "modeling first" approach, it is possible to verify beforehand generation the correctness of the model.

Before to start, we should have in mind what we would like to generate.
In this tutorial, the goal is to generate a basic application composed by two cblock: a _sender_ and a _receiver_ , thus we want to generate a package which contains:

* _sender_ and _receiver_ implementation
* a custom datatype exchanged between the two cblocks
* compile the modules (three in total: one for the new datatype, and two for the blocks)
* write a .usc file to run the application

This tutorial fully apply to cblocks written in __C++__ too.

Prerequisites
-------------

To run and understand this tutorial, be sure that:

* you already [installed Microblx](/quickstart/)
* you already [installed microblx_cmake]() tool
* you read the [documentation]() (or you would like to check it while doing the tutorial)
* (very) basic knowledge about [CMake framework](http://www.cmake.org/)

Model Design: how to generate a package
--------------------

### The package model

The first step is to create a package model (extension .pkg) which describes the future contents of your package. Using your favorite text editor, we create a new file named _cblock_tutorial_auto.pkg_ with the following informations
{% highlight lua %}
return pkg
{
  name="cblock_tutorial_auto",
  path="../",
  
  types = {
    { name="my_data", dir="types" },
  },
  
  blocks = {
    { name="sender", file="sender_block.blx", src_dir="src" },
    { name="receiver", file="receiver_block.blx", src_dir="src" },
  },
  
  modules = {
    { name="sendermod", blocks={"sender"} },
    { name="receivermod", blocks={"receiver"} },
  },
}
{% endhighlight %}

In short, in this description you are saying that the package name is _cblock_tutorial_auto_ and it will contains:

+ a new datatype definition, called _cpp_data_ (sources stored in the folder _types_ of your package)
+ two new blocks sources, stored in the foulder _src_ . The skeleton of the blocks is indicated in the block models (you have to create those!)
+ once the sources have been compiled, you will obtain two modules for each block.

> Please, notice that the latest is a pure design decision: you could prefer to have your blocks in one single module.
{% highlight lua %}
  modules = {
    { name="cblock_tutorial", blocks={"sender","receiver"} },
  },
}
{% endhighlight %}

> Note: a module name must not have the same name of a block

Further details about the package model can be found in the [Documentation]().

### The block model

Another important step is to define the model of the cblocks we want to generate.
We create two files: _sender_block.blx_ and _receiver_block.blx_ with the following contents.

___sender_block.blx___
{% highlight lua %}
return block
{
      name="sender",
      meta_data="A sender cblock which can handle my data",
      port_cache=true,

      types = {
        -- define types this blocks uses
	 { name="my_data", class='struct' },
      },

      ports = {
	-- define the ports for this block
	 { name="output", out_type_name="struct my_data", out_data_len=1, doc="output port for my data" },
      },
      
      -- define which operations this block implements
      operations = { start=true, stop=true, step=true },

      cpp = false,
}
{% endhighlight %}
___receiver_block.blx___
{% highlight lua %}
return block
{
      name="receiver",
      meta_data="A receiver cblock which can handle my data",
      port_cache=true,

      types = {
	-- define types this blocks uses
	 { name="my_data", class='struct' },
      },

      ports = {
	-- define the ports for this block
	 { name="input", in_type_name="struct my_data", in_data_len=1, doc="input port for my data" },
      },
      
      -- define which operations this block implements
      operations = { start=true, stop=true, step=true },

      cpp = false,
}
{% endhighlight %}

> The flag "cpp = false" is used to generate .c and .h files instead of .cpp and .hpp files. Apart of that, the rest of the tutorial applies to __C++__ cblocks too.

> The "types" field assumes a different meaning in this context. While in the package model is the new datatype you want to define, in the block model is the list of datatypes the block is going to use.

### Generate the package!

Run the ubx_pkg_gen targeting the .pkg specification you just created
{% highlight bash %}
$ ubx_pkg_gen -s cblock_tutorial_auto.pkg -d ../cblock_tutorial_auto
{% endhighlight %}

...and you should obtain the following output
{% highlight bash %}
    generating ../cblock_tutorial_auto/src/sender.h
    generating ../cblock_tutorial_auto/src/sender.c
    generating ../cblock_tutorial_auto/src/sender.usc
    generating ../cblock_tutorial_auto/src/receiver.h
    generating ../cblock_tutorial_auto/src/receiver.c
    generating ../cblock_tutorial_auto/src/receiver.usc
    generating ../cblock_tutorial_auto/modules/sendermod_module.h
    generating ../cblock_tutorial_auto/modules/sendermod_module.c
    generating ../cblock_tutorial_auto/modules/receivermod_module.h
    generating ../cblock_tutorial_auto/modules/receivermod_module.c
    export models in cblock_tutorial_auto/models
{% endhighlight %}

that is, in your current folder you should find a new directory, with the name of the package _cblock_tutorial_auto_ , 
contains all the sources you need to implement your blocks!

Implement ubx types and blocks
------------------------------

In this step you have to fill in the generated stubs for your types and blocks.

### Implementing the types

First, have a look at the types directory and the generated header files (one for each type you defined).
In our example, two files have been generated:

{% highlight bash %}
$ cd cblock_tutorial_auto/types/
~cblock_tutorial_auto/types$ ls
my_data.h  cblock_tutorial_auto_types.c
{% endhighlight %}

The _my_data.h_ file defines the implementation of the type you want to share between ubx blocks, while the _cblock_tutorial_auto_types.c_ file defines ubx functions to register the types into a specific module.. The former still requires some editing, while the latter should be modified only if an extra type is added manually.

You procede to complete the _my_data.h_ source, by editing as follow:
{% highlight c %}
struct my_data {
	/* This should be generated in the future */
        char model[20]; // This should be a URL to the data model
	char uid[10]; // This should be a unique id to identify this data type
	char meta_model[20]; // This should be a URL to the meta model
	char data[10]; // the actually data. In our example, just 10 random characters.
};
{% endhighlight %}

### Implementing the blocks

Finally, you are ready to implement the functions realized by the cblocks by editing their step functions!
In this example, you would like to fill in some data in the type

To do so, open the sources _sender.c_ and _receiver.c_ in the subfolder _src_, search for their step function and complete them 
as follow:

In _src/sender.c_
{% highlight c %}
/* step */
void sender_step(ubx_block_t *b)
{
  struct sender_info *inf = (struct sender_info*) b->private_data;
  struct my_data data;
  strcpy(data.model, "image array");
  strcpy(data.uid,"image");
  strcpy(data.meta_model,"c array");
  strcpy(data.data,"test_data");

  write_output(inf->ports.output, &data);
}
{% endhighlight %}

In _src/receiver.c_
{% highlight c %}
/* step */
void receiver_step(ubx_block_t *b)
{
  struct receiver_info *inf = (struct receiver_info*) b->private_data;
  struct my_data dat;
  read_input(inf->ports.input, &dat);
  printf("MODEL: %s\n",dat.model);
  printf("UID: %s\n",dat.uid);
  printf("META MODEL: %s\n", dat.meta_model);
  printf("DATA: %s\n",  dat.data);
}
{% endhighlight %}

In this example, we don't have to complete the other functions (_init_, _start_, _stop_,_cleanup_).
For further details about these, check the [Anatomy of a cblock](/Documentation/cblock_explained).

### Build and Install with CMAKE

We are finally ready to compile your blocks!
All the steps documented here are pretty standard within __CMAKE__ framework. However, you might be not familiar to it.
Don't worry about that, because there are only few things to do!

The steps are:

* create a subfolder (usually named _build_); all temporary files due to compilation will be stored there
* Note: if you want to use a versioning tool for your code (git, svn, ...), remember to don't commit the _build_ folder!
* run cmake to compile.
* if you want to install your modules, then set up the installation path setting the cmake variable __CMAKE_INSTALL_PREFIX__

In short, type:
{% highlight bash %}
~cblock_tutorial_auto$ mkdir build && cd build/
~cblock_tutorial_auto/build$ cmake .. -DCMAKE_INSTALL_PREFIX=<your-installation-path>
~cblock_tutorial_auto/build$ make
{% endhighlight %}

To Install, run:
{% highlight bash %}
~cblock_tutorial_auto/build$ make install
{% endhighlight %}

Running the example
-------------------

### Create USC script

To deploy our application, we procede on creating a Ubx System Composition file (USC) named _cblock_tutorial_auto.usc_
For our tutorial, we need to fill in the following informations.
{% highlight lua %}
return bd.system {
   imports = {
      "std_types/stdtypes/stdtypes.so",
      "std_blocks/ptrig/ptrig.so",
      "std_blocks/lfds_buffers/lfds_cyclic.so",
      "std_blocks/hexdump/hexdump.so",
      "types/cblock_tutorial_auto_types.so",
      "blocks/sendermod.so",
      "blocks/receivermod.so",
   },
   
   blocks = {
      {name="ptrig1", type="std_triggers/ptrig"},
      {name="fifo1", type="lfds_buffers/cyclic"},
      {name="sender1", type="sender"},
      {name="receiver1", type="receiver"},
   },
   
   connections = {
-- connect the sender block with the receiver block through a fifo block
      {src="sender1.output", tgt="fifo1"},
      {src="fifo1", tgt="receiver1.input"},
   },
   
   configurations = {
      { name="fifo1", config={type_name="struct my_data", buffer_len=1} },
      { name="ptrig1", config={ period={sec=1,usec=0}, 
                                trig_blocks={ {b="#sender1", num_steps=1, measure=0},
                                              {b="#receiver1", num_steps=1, measure=0} } } }
   },
}
{% endhighlight %}

Thanks to these, we specify which modules will be loaded, which blocks will be created, their configuration and the connections between them (through iblocks).
Further informations about the USC files can be found in [USC documentation](/Documentation/usc_explained).

### Create launch script (optional)

Sometimes it is convenient create a script to wrap up the command line to execute you application. We can do it by creating a script _run.sh_ as follow:

{% highlight bash %}
#!/bin/bash
exec $UBX_ROOT/tools/ubx_launch -webif 8888 -c cblock_tutorial_auto.usc
{% endhighlight %}

Remember to give executable rights to your script
{% highlight bash %}
$ chmod +x run.sh
{% endhighlight %}

### Launch the application!

Now, start your application by typing:

{% highlight bash %}
~/workspace/cpp_transfer$ ./run.sh
{% endhighlight %}

If everything goes fine, you should obtain a similar output at your terminal

![Pins](runusc.png )

If not, usefull error message will help you to understand what went wrong.

If not specified, your blocks are not running at the startup.
You can have fun and do it manually by using the [webinterface]().

1. Browse to [http://localhost:8888](http://localhost:8888)
![Pins](webint1.png)
You should see both __sender__ and __receiver__ in ___init___ mode. 
We can also check the current status of the whole application thanks to the [http://localhost:8888/node](node graph)
![Pins](node.png)
The ptrig1, representing the ___sblock___ of the application, triggers first the __sender__, then the __receiver__, in sequence.
Both __sender__ and __receiver__ are in ___init___ mode (blue color), while the __ptrig1__ is stopped.
2. By clicking  ___init___ buttons, we are initializing the __sender__ and the __receiver__ .
3. By clicking __start___ buttons, we starting the cblocks.
4. Finally, we start the ptrig1 block, bridge block responsible to attach a periodic activity to your algorithm.

The results should be as follow:
![Pins](webint2.png)

While, if everything goes is well, we should receive following output on our terminal:
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

That means the cblock __sender__ is properly setting some data, while the _receiver_ is reading it!

(___NOTE___: further details over the webinterface can be found [here](/Tutorials/ubx_webinterface) ).

Generate pure C application (from ubx system composition file!)
-----------------------------------
(
(___note___: still under development. Steps like checkout of dev branch, manually file copying and CMakeFile editing will be automated later on)


Then, generate the source file for the application with
{% highlight bash %}
./ubx_capp_gen -o my_app -f cblock_tutorial_auto.usc --webif
{% endhighlight %}

and we will find the source as "my_app.c".

As last step, it is necessary to compile the source, by copying the source in one project and adding the following to your CMakeFiles.txt
{% highlight bash %}
add_executable(my_app src_bin/m_app.c)
target_link_libraries(my_app ${UBX_LIBRARIES})
add_dependencies(my_app gen_hexarr)
{% endhighlight %}
Of course, it is possible to do the same if you are using a plain Makefile system.

Compile everything and we should obtain an executable "my_app"

By launch it, we obtain
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
and we can navigate, as usual, to the [web interface](http://localhost:8888).

Get the code!
--------------

You can find the sources obtained by following the tutorial [here](https://github.com/UbxTeam/ubx_tutorials/tree/master/cblock_tutorial_auto).