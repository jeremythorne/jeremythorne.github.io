This is a set of notes on trying to get headless OpenGL-ES2 rendering working on a rapsberry-pi. 

By headless I mean, creating a GPU context without having a display or xserver running e.g. communicating with the pi purely over the network.

Using [DRM render nodes](https://en.wikipedia.org/wiki/Direct_Rendering_Manager) to create an EGL context, I now have simple code that successfully creates a glFramebuffer, uploads pixel data with glTexImage2D, renders a textured quad and then uses glReadPixels to inspect the results. That is, the basics of some mixed CPU/GPU computation e.g. computer vision, image processing, AI ...


![my first rendered quad](first_quad.png)

Some benchmarking 10000 loops of upload, render, read

| upload size | render size | read size | raspberry pi 3 (time ms) | 2005 intel macbook running xubuntu(time ms) |
|-------------|-------------|-----------|--------------------------|---------------------------------------------|
| 4x4         | 4x4         | 4x4       | 869                      | 986                                         |
| 4x4         | 256x256     | 4x4       | 3498                     | 3727                                        |
| 256x256     | 4x4         | 4x4       | 19822                    | 1772                                        |
| 4x4         | 256x256     | 256x256   | 39582                    | 9340                                        |
| 256x256     | 256x256     | 256x256   | 45664                    | 11836                                       |

the first row is essentially just overhead - the cost of doing business. The raspberry pi has similar overhead and rendering performance to the macbook, but it's texture upload performance is ~ 1/10th and it's read performance 1/7th that of the macbook.
So to do anything useful you'd want to try and keep activity in the GPU and return minimal results (e.g. a count of non zero pixels).

TODOs:
* pull request for ModernGL to use render nodes so we can develop in python.
* actually do some useful computation to compare against processing just on the CPU and to wrestle with storing results as pixels.

---------------------------------

Oldest comments at the top.


1st July 2017

The original plan was to use python3 ModernGL to speed development, but this doesn't yet have support for creating a context without having a display or xserver running...

Some notes on headless rendering on a raspberry pi

Setup a micro SD card with a minimal raspbian image

Follow [caffinc's guide](https://caffinc.github.io/2016/12/raspberry-pi-3-headless/) to set up your pi without a screen or keyboard

setup your Pi to use the open source desktop OpenGL driver
```bash
raspi-config
```
choose advanced options, GL driver, full KMS


install python3 pilow and ModernGL

```bash
sudo apt-get update
sudo apt-get install python3-pip
```
and then  [ModernGL](https://github.com/cprogrammer1994/ModernGL#installing-on-ubuntu-from-source)
and [pre-requisites for pillow](https://askubuntu.com/questions/156484/how-do-i-install-python-imaging-library-pil) 
but without the lipjpeg (as that doesn't exist for pi)


```bash
sudo pip3 install pillow
```


get the ModernGL source and examples (possibly you could find these from whereever pip installed them)
```bash
sudo apt-get install git
git clone <path to ModernGL.git>
```
9th July 2017

```bash
pi@raspberrypi:~/dev/ModernGL/examples/standalone $ python3 01_hello_world.py 
Traceback (most recent call last):
  File "01_hello_world.py", line 9, in <module>
    ctx = ModernGL.create_standalone_context()
  File "/usr/local/lib/python3.4/dist-packages/ModernGL/context.py", line 748, in create_standalone_context
    ctx = Context.new(mgl.create_standalone_context(*size))
mgl.Error: cannot detect the display
```

it's still trying to use glx to get the display.


let's install glxinfo to see what GL we've actually got...
```bash
sudo apt install mesa-utils
...
pi@raspberrypi:~/dev/ModernGL/src $ export DISPLAY=:0
pi@raspberrypi:~/dev/ModernGL/src $ glxinfo | grep "version"
Error: unable to open display :0

```

```bash
sudo pip3 intall PyOpenGL
python
>>> import OpenGL
>>> os.environ['PYOPENGL_PLATFORM'] = 'glx'
>>> import OpenGL.glx
>>> OpenGL.GLX.glXGetClientString(None, OpenGL.GLX.GLX_VENDOR)
b'Mesa Project and SGI'
>>> OpenGL.GLX.glXGetClientString(None, OpenGL.GLX.GLX_VERSION)
b'1.4'
>>> 
```
well that's somewhere, ...
but attempting to get available configs with glXGetFBConfigs gave me seg faults.

I think we need to use EGL to create a windowless context (without talking to X), however the issue here is that only the original Broadcom OpenGLES stack provides EGL, and not the Mesa Desktop GL driver.

```bash
pi@raspberrypi:~/dev $ ldconfig -p | grep EGL
        libbrcmEGL.so (libc6,hard-float) => /opt/vc/lib/libbrcmEGL.so
        libEGL.so (libc6,hard-float) => /opt/vc/lib/libEGL.so
```

15th July 2017

need to install mesa EGL
```bash
 sudo apt install libegl1-mesa  
```

```python
import os
import OpenGL , ctypes
os.environ['PYOPENGL_PLATFORM'] = 'egl' 
from OpenGL.EGL import *
major = ctypes.c_int()
minor = ctypes.c_int()
display = eglGetDisplay(EGL_DEFAULT_DISPLAY)
eglInitialize(display, ctypes.byref(major), ctypes.byref(minor))
print(eglQueryString(0, EGL_VERSION))
print(eglQueryString(0, EGL_VENDOR))
eglTerminate()

```
gives:
```bash
libEGL warning: DRI3: xcb_connect failed
libEGL warning: DRI2: xcb_connect failed
libEGL warning: DRI2: xcb_connect failed
Traceback (most recent call last):
  File "test1.py", line 8, in <module>
    eglInitialize(display, ctypes.byref(major), ctypes.byref(minor))
  File "/usr/local/lib/python3.4/dist-packages/OpenGL/platform/baseplatform.py", line 402, in __call__
    return self( *args, **named )
  File "/usr/local/lib/python3.4/dist-packages/OpenGL/error.py", line 232, in glCheckError
    baseOperation = baseOperation,
OpenGL.error.GLError: GLError(
        err = 12289,
        baseOperation = eglInitialize,
        cArguments = (
                <OpenGL._opaque.EGLDisplay_pointer object at 0x76373620>,
                <cparam 'P' (0x769f7878)>,
                <cparam 'P' (0x763735a8)>,
        ),
        result = 0
)

```
where 12289 is 0x3001  EGL_NOT_INITIALIZED EGL is not or could not be initialized, for the specified display.

14th August 2017

what we need is to use DRM Render Nodes
[see elima's blog](https://blogs.igalia.com/elima/2016/10/06/example-run-an-opengl-es-compute-shader-on-a-drm-render-node/)
I'm going to try and get a version of that sample code (using GLES2 not 3) running both on my 2005 macbook and raspberry pi


16h August

stuff to install on macbook and rpi
```bash
sudo apt install pkg-config libegl1-mesa-dev libgles2-mesa-dev libgbm-dev
```

I now have this compiled and running on my macbook running xubuntu - creating an EGL context suitable for GLES 2 and then throwing it away again. I need to add code to actually do some minimal GLES 2.
See [my copy of elima's code on github](https://github.com/jeremythorne/raspberrypi-playground/tree/master/elima)

this also run without error on raspberrypi
```bash
@raspberrypi:~/dev/raspberrypi-playground/elima $ ./render-nodes-minimal 
MESA-LOADER: failed to retrieve device information
MESA-LOADER: failed to retrieve device information
MESA-LOADER: failed to retrieve device information
pi@raspberrypi:~/dev/raspberrypi-playground/elima $ echo $?
0
```

once I've actually rendered something I'd like to look at patching ModernGL to support render nodes as a means of creating a context when glx returns no display.

23rd August
I now have a simple application that does a glClear to a framebuffer and then a glReadPixels - it's possible this all happens in SW, but I'm hopeful that it is now actually tickling the GPU. Runs both on 2005 macbook and raspberry pi 3.
[code](https://github.com/jeremythorne/raspberrypi-playground/tree/master/headless_gl/gl_clear)
I get slighlty different results on each
pi:
```bash
pi@raspberrypi:~/dev/raspberrypi-playground/headless_gl/gl_clear $ ./headless_gl_clear 
MESA-LOADER: failed to retrieve device information
MESA-LOADER: failed to retrieve device information
MESA-LOADER: failed to retrieve device information
1a 1a 1a ff
```
macbook:
```bash
19 19 19 ff
```
maybe I'm getting different bit depth surfaces? Or just hitting different rounding?

26th August

I now have code running on my macbook-with-xubuntu and raspberry pi that loads data using glTexImage2D, creates a glFramebuffer, renders a textured quad with glDrawArrays and then reads the results using glReadPixels. [My code](https://github.com/jeremythorne/raspberrypi-playground/tree/master/headless_gl/gl_quad). I'm using [stb image](https://github.com/nothings/stb) for saving results.

![my first rendered quad](first_quad.png) 

some first quick benchmarking. 10000 loops of upload, render, read a 256x256 image

2005 macbook with xubuntu
```
OpenGL vendor string: Intel Open Source Technology Center
OpenGL renderer string: Mesa DRI Intel(R) 965GM x86/MMX/SSE2

real    0m10.513s
user    0m7.524s
sys     0m0.404s
```

raspberry pi 3
```
real	0m58.205s
user	0m54.230s
sys	0m0.840s
```
ouch.

More detailed benchmarking (times are for 10000 loops)
using [benchmark code](https://github.com/jeremythorne/raspberrypi-playground/tree/master/headless_gl/benchmark)

| upload size | render size | read size | raspberry pi 3 (time ms) | 2005 intel macbook running xubuntu(time ms) |
|-------------|-------------|-----------|--------------------------|---------------------------------------------|
| 4x4         | 4x4         | 4x4       | 869                      | 986                                         |
| 4x4         | 256x256     | 4x4       | 3498                     | 3727                                        |
| 256x256     | 4x4         | 4x4       | 19822                    | 1772                                        |
| 4x4         | 256x256     | 256x256   | 39582                    | 9340                                        |
| 256x256     | 256x256     | 256x256   | 45664                    | 11836                                       |

that second row gives the raspberry pi GPU a fill rate of about 2GB/s 
(10000 loops * 256 * 256 pixels * 4 bytes per pixel * 2 read in and write out / 3498 - 869 ms)

|               | pi 3 (MB/s) | macbook (MB/s) |
|---------------|-------------|----------------|
| fill rate     | 1900        | 1820           |
| upload rate   | 132         | 3180           |
| download rate | 69          | 445            |

7th September

Obviously the raspberry pi GPU (aka [Videocore V3D](https://docs.broadcom.com/docs/12358545)) wasn't designed to be used like this. It was primarily designed for rendering user interfaces on phones and set top boxes and for playing 3D games. In these cases the textures and geometry are uploaded at the start of day (so little ongoing upload bandwidth) and results were streamed straight to the display (so no download to the CPU, and no waiting on the CPU for the previous frame to complete before issuing instructions for the next). Further glReadPixels was only ever designed for correctness and debugging capability so it's possible we're hitting unoptimised code paths here. Memory bandwith is power hungry, so that partly explains why the macbook has better upload and download speeds, but the pi GPU also expects it's textures to be in "t-format" (a [hilbert curve](https://en.wikipedia.org/wiki/Hilbert_curve) like re-arrangement of pixels that improves cache performance particularly when textures are rotated) so we'll be paying some price for that conversion to and from raster format (are we using ARM NEON to help with this?).

10th Sept
I wanted to run my benchmark under gdb and do some random ctrl+c's to see where we ended up. Unfortunately I'm not getting much of a back trace.

```bash
(gdb) r
Starting program: /home/pi/dev/raspberrypi-playground/headless_gl/benchmark/headless_gl_quad 
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/arm-linux-gnueabihf/libthread_db.so.1".
MESA-LOADER: failed to retrieve device information
MESA-LOADER: failed to retrieve device information
MESA-LOADER: failed to retrieve device information
minimal upload, full render, read
^C
Program received signal SIGINT, Interrupt.
0x76fbca1c in memcmp () from /usr/lib/arm-linux-gnueabihf/libarmmem.so
(gdb) bt
#0  0x76fbca1c in memcmp () from /usr/lib/arm-linux-gnueabihf/libarmmem.so
#1  0x00000000 in ?? ()
(gdb) 
```

and trying to install more sybmols doesn't get anywhere much
```bash
pi@raspberrypi:~/dev/raspberrypi-playground/headless_gl/benchmark $ sudo apt install  libegl1-mesa-dbg libgles2-mesa-dbg 
Reading package lists... Done
Building dependency tree       
Reading state information... Done
Some packages could not be installed. This may mean that you have
requested an impossible situation or if you are using the unstable
distribution that some required packages have not yet been created
or been moved out of Incoming.
The following information may help to resolve the situation:

The following packages have unmet dependencies:
 libegl1-mesa-dbg : Depends: libegl1-mesa (= 11.1.0-1+rpi1) but 13.0.0-1+rpi1 is to be installed
 libgles2-mesa-dbg : Depends: libgles2-mesa (= 11.1.0-1+rpi1) but 13.0.0-1+rpi1 is to be installed
E: Unable to correct problems, you have held broken packages.
```
