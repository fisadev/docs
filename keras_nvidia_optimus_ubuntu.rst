How to run Keras (Theano backend) with a Nvidia Optimus gpu using CUDA, under Ubuntu 16.04 
==========================================================================================

I broke so many things trying to get this to work, that I need to write it down for my future self and any others who might need this.

This tutorial explains how to get a Keras neural network to train using your gpu, or any Theano related code for that matter, but for the special scenario of having a Nvidia Optimus enabled gpu running under Ubuntu 16.04. 
Other scenarios:

* Under Ubuntu 16.04 and you have a Nvidia gpu which is not Optimus enabled? then some things might work, but the "Nvidia gpu drivers" section won't be useful to you.
* Under a different distro or Ubuntu version? I would **not** recommend trying this. 
* Have an non-Nvidia gpu? this **won't work** at all.

Enough presentations, lets get to work.

0. The example
==============

We will need a Keras example to check if things are working well. 
Use the code from `this repo <https://github.com/fisadev/keras_experiments>`_. 

In that repo you will find a keras experiment (in one of the ipython notebooks), and a small script to check for gpu usage from Theano. 
Both will be useful.

**Remember** to create a virtualenv and **install its dependencies** (from the ``requirements.txt``).
You can use wheels to speed up the process. And it requires **python 3**.

1. Nvidia gpu drivers
=====================

We need Ubuntu to be able to use our gpu. 
To achieve that, we need both the gpu drivers and the Optimus drivers.
Thing is, Optimus drivers are kind of broken right now under Ubuntu 16.04.

But we can get things working, with some small annoyances:

1. Disconnect any external/secondary monitors.
2. Install video and optimus drivers: ``sudo apt install nvidia-361 nvidia-prime``
3. Reboot your machine and login.
4. Open the "Nvidia X Server Settings" app, go to the "PRIME Profiles" section and choose "NVIDIA".
5. Close the settings app.
6. Logout and login again, so the profile change can take effect.

**A small annoyance**: while the "NVIDIA" profile is active you cannot (at the moment) use a secondary screen. 
It will crash your X server.

**Important**: remember to go back to the "Intel" profile whenever you don't want to use your gpu anymore, logout and login back for it to take effect.
Otherwise, your gpu will be using lots of power for nothing.

2. CUDA
=======

We need to be able to do general purpose computation on our Nvidia gpu.
To achieve that, we need the CUDA toolkit.
Thing is, CUDA has no Ubuntu 16.04 package at the moment. 

But we can make things work. This is Linux, after all (thanks to Martin Thoma's `answer <http://askubuntu.com/questions/799184/how-can-i-install-cuda-on-ubuntu-16-04>`_):

1. Download CUDA from the `Nvidia website <https://developer.nvidia.com/cuda-downloads>`_. Choose the Ubuntu 15.04 "runfile (local)" version.
2. Check the md5 sum with: ``md5sum cuda_7.5.18_linux.run``. Only continue if it is correct.
3. Run the installer and follow the instructions: ``sudo sh cuda_7.5.18_linux.run --override``. **Make sure** that you say **y** for the symbolic link, and **n** to the video drivers installation.

Don't worry about the warning related to the non-installation of the video drivers. 
The script is somewhat dumb and doesn't detect the drivers we installed on the previous section.

3. Gcc and G++ versions
=======================

Guess what? Yes, more problems.

CUDA requires gcc to be a version up to 4.9. 
Later versions won't work.
Ubuntu 16.04 ships with gcc 5.4.

But we can have both versions side by side, and trick CUDA to use the older versions with just some simple symlinks (thanks to `charlie <https://twitter.com/_zzzoom_/status/765720104868904964>`_ for the tip, the old solution was far more complicated).

.. code:: bash

    sudo apt-get install gcc-4.9 g++-4.9
    sudo ln -s /usr/bin/gcc-4.9  /usr/local/cuda/bin/gcc
    sudo ln -s /usr/bin/g++-4.9  /usr/local/cuda/bin/g++

    # Work around a glibc bug (according to the theano docs)
    echo -e "\n[nvcc]\nflags=-D_FORCE_INLINES\n" >> ~/.theanorc


4. Testing
==========

Done! You should be able to run Theano things (including Keras neural networks) using your gpu.

A simple way to check if that's true, is to run the ``test_gpu.py`` script from the example repo.
But don't just "run" it.
You need to tell Theano "hey, I want you to use my shiny cuda gpu". 
To achieve that, run the script like this (inside your virtualenv):

.. code:: bash

    THEANO_FLAGS="mode=FAST_RUN,device=gpu,floatX=float32,cuda.root=/usr/local/cuda/" python test_gpu.py


If everything is working, it should quickly run and output something which ends with: ``Used the gpu``.
If instead it takes a long time (~1 minute) and ends with ``Used the cpu``, then something is not working.

If that worked, then you can try the full example and play a little with it (should be able to run all lines without errors):

.. code:: bash

    THEANO_FLAGS="mode=FAST_RUN,device=gpu,floatX=float32,cuda.root=/usr/local/cuda/" ipython notebook
