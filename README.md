1st July 2017

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

```bash
pi@raspberrypi:~/dev/ModernGL/examples/standalone $ python3 01_hello_world.py 
Traceback (most recent call last):
  File "01_hello_world.py", line 9, in <module>
    ctx = ModernGL.create_standalone_context()
  File "/usr/local/lib/python3.4/dist-packages/ModernGL/context.py", line 748, in create_standalone_context
    ctx = Context.new(mgl.create_standalone_context(*size))
mgl.Error: cannot detect the display
```

it's still trying to use glx to get the display. I think we need to use EGL to create a context instead.

