---
layout: post
title: "Compiling VMD with Python support"
author: Robin Betz
date: 2015-01-08 14:01:40 -0800
comments: true
categories: tutorial
published: true
---

Visual Molecular Dynamics (VMD) is great. Tcl isn't (IMO). Fortunately, VMD has support for a fully-featured Python interpreter!
Unfortunately, this support isn't built into the available binaries, and compiling it is kind of 
weird and confusing. Fortunately, I've done all the being confused for you and have written this
guide on compiling VMD from source.

*UPDATE* _6 April 2015_ Added all environment variables that need to be set to compile all plugins.

<!-- more -->

## Getting the source
Go to the [VMD Website](http://www.ks.uiuc.edu/Research/vmd/) and download the source code.
You will need to make a free account. Extract the source in a directory of your choosing. I would
recommend not putting it in /tmp because you might want it later if you are developing plugins.

## Compiling

### 1. Setting the compilation option flags

A full explanation of all compilation options is at the beginning of configure,
and many of them depend on what hardware you have.

I set the following compile options: 

* `LINUXAMD64` - for a Linux x86\_64 build
* `FLTK` - for the graphical user interace
* `FLTKOPENGL` - for a FLTK window with the OpenGL stuff
* `CUDA` - for GPU acceleration
* `MPI` - for parallelization
* `TK` - for user extensions to the GUI (you will need this for plugins)
* `NETCDF` - support for binary format trajectories
* `COLVARS` - collective variables module for NAMD/LAMMPS support
* `TCL` - support for plugins in TCL
* `PYTHON` - the whole reason we are compiling from source, enable python support
* `NUMPY` - also numpy support
* `NOSILENT` - print the entire make command during compilation

So the entire contents of my *configure.options* file is:

    LINUXAMD64 FLTKOPENGL FLTK CUDA MPI TK NETCDF COLVARS TCL PYTHON PTHREADS NUMPY NOSILENT

### 2. Specifying Environment Variables

Some plugins will only be compiled if certain environment variables are set,
even if the compiler arguments they reference are redundant.

It is simplest to set all the environment variables in a single script, 
which is sourced before running the configure script or compiling plugins.
Edit the following script as appropriate for your system and the plugins you wish to compile:

{ % codeblock lang:bash /etc/udev/rules.d/99-keyboard-leds.rules % }
\#!/bin/bash

\# Define directories and names of final binaries
export VMDINSTALLNAME="vmd-1.9.2-shared"
export VMDINSTALLBINDIR="/opt/$VMDINSTALLNAME"
export VMDINSTALLLIBRARYDIR="/opt/$VMDINSTALLNAME"
export PLUGINDIR="$VMDINSTALLLIBRARYDIR/plugins"

\# For reading and writing NetCDF format files
export NETCDFLIB="-L/usr/lib64"
export NETCDFINC="-L/usr/include"
export NETCDFLDFLAGS="-lnetcdf"

\# For GUI, vtfplugin, webpdbplugin
export TCLLIB="-L/usr/lib64"
export TCLINC="-I/usr/include"
export TCLLDFLAGS="-ltcl"

\# For dmsplugin (.dms files)
export SQLITEINC="-I/usr/include"
export SQLITELIB="-L/usr/lib64"
export SQLITELDFLAGS="-lsqlite3"

\# For hoomd plugin
export EXPATLIB="-L/usr/lib64"
export EXPATINC="-I/usr/linclude"
export EXPATLDFLAGS="-lexpat"

\# For python support (required if PYTHON in configure.options)
export PYTHON\_INCLUDE\_DIR="/usr/include/python2.7"
export PYTHON\_LIBRARY\_DIR="/usr/lib64/python2.7"
export NUMPY\_INCLUDE\_DIR="/usr/lib64/python2.7/site-packages/numpy/core/include"
export NUMPY\_LIBRARY\_DIR="/usr/lib64/python2.7/site-packages/numpy/core/lib"

{ % endcodeblock % }

Then, source it:

      source vmd_env_variables.sh

If they are not set, VMD will attempt to use its own versions, which is Python 2.5 and
whatever numpy it ships with.

Because I was compiling multiple times and didn't want to forget to set any environment variables,
I chose to edit the configure script and altered the following lines to
compile against my miniconda Python installation (which I recommend) that I have put in /opt.

    $stock_python_include_dir=$ENV{"PYTHON_INCLUDE_DIR"} || "/opt/miniconda/include/python2.7"
    $stock_python_library_dir=$ENV{"PYTHON_LIBRARY_DIR"} || "/opt/miniconda/lib/python2.7/config";
    $stock_numpy_include_dir=$ENV{"NUMPY_INCLUDE_DIR"} || "/opt/miniconda/lib/python2.7/site-packages/numpy/core/include";
    $stock_numpy_library_dir=$ENV{"NUMPY_LIBRARY_DIR"} || "/opt/miniconda/lib/python2.7/site-packages/numpy/core/lib";

Regardless of if you used environment variables or edited the configure script, you need to change
the linker command to link the appropriate version of Python, in my case 2.7.
    
    $python_libs = "-lpython2.7";

### 3. Specify installation location

At the top of the configure script the location of the installed binaries is specified. Change this
to where you want your stuff to go and the name of the binary. I would recommend picking a place
that you have permission to write to, or run all sudo commands as `sudo -E` to preserve environment
variables that you've set.

I have root on this machine so I will install vmd globally in /usr/local.

    $install_name = "vmd-1.9.2";
    $install_bin_dir = "/usr/local/bin";
    $install_library_dir = "/usr/local/lib/$install_name";

Alternatively, you can do this with environment variables by setting the following:

* VMDINSTALLNAME
* VMDINSTALLBINDIR
* VMDINSTALLLIBRARYDIR

Don't run the configure script yet. First, the plugins must be compiled.

### 4. Compile plugins

VMD ships with a variety of accessory programs that must first be compiled before compiling VMD itself.
In the directory where you extracted the source, cd to distrib/plugins.

The psfgen plugin, a dependency for many other plugins, will not build without the following environment
variables set to specify the location of the tcl library preceded by the correct flags for gcc (-I, -L, -l).

* TCLINC
* TCLLIB
* TCLLDFLAGS

You will also need to set the PLUGINDIR variable to specify where the final plugins will be located. In 
theory you can put them anywherer, but the result will be symlinked by $VMDINSTALLLIBRARYDIR/plugins, so
it is best to create that folder and then set it as PLUGINDIR. _Note that I am using sudo -E so that all
environment variables are inherited!_

    export PLUGINDIR=/usr/local/lib/vmd-1.9.2/plugins
    sudo -E mkdir -p $PLUGINDIR

Set these variables when compiling the plugins, and specify your architecture (again, probably LINUX64):

    gmake TCLINC=-I/usr/include TCLLIB=-L/usr/lib TCLLDFLAGS=-ltcl8.5 LINUX

Assuming compilation completes successfully, you can put the plugins in their directory. Again, use sudo -E
if you need permissions.

    sudo -E gmake distrib

Hopefully, your plugins now appear in $PLUGINDIR

### 5. Compile VMD

Now that the plugins are compiled and in the right place, we can run the configure script from the
VMD source directory.

    ./configure

Now compile.

    cd src
    make veryclean
    make
    sudo -E make install

Usually the last message is "No resource compiler required on this platform", which is not an error 
message. If you're not sure if make ran successfully (it can be difficult with all that output), try
`echo $?` right after the command finishes. If 0, it was successful. Otherwise, there was an error.

### 6. Test installation

In theory you should now have a working VMD installation with Python support enabled. Test this by
launching VMD.

    /usr/local/bin/vmd-1.9.2

Now check the following:

1. *Does the GUI appear?*
   You should see a grey box with menus in it. If not, check that you compiled with
   the FLTK and the TK options.

2. *Are the plugins loaded correctly?*
   Check that there are no error messages in the console about plugins not loading. Also check
   that the GUI menu named "Extensions" is populated. If not, check your setting of PLUGINDIR,
   check that psfgen was compiled ($PLUGINDIR/LINUX/tcl/psfgen1.6 should exist), and check that
   you distributed the plugins.

3. *Does python work?*
   In the console, type `gopython`. You should get a python shell. If not, check that you set
   PYTHON in configure.options, that the interpreter installation you built against is functional,
   and that you remembered to run make install and aren't looking at an older version of VMD (I've done
   this, oops).


