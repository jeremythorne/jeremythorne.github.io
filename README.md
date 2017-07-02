1st July 2017

Some notes on headless rendering on a raspberry pi

Setup a micro SD card with a minimal raspbian image

Follow [caffinc's guide](https://caffinc.github.io/2016/12/raspberry-pi-3-headless/) to set up your pi without a screen or keyboard

install python3 pilow and ModernGL

```bash
sudo apt-get update
```
and then  [ModernGL](https://github.com/cprogrammer1994/ModernGL#installing-on-ubuntu-from-source)
```bash
sudo apt-get install python3-pip
sudo pip3 install ModernGL
sudo pip3 install pillow
```
but that fails because it can't find jpeg headers...


get the ModernGL source and examples (possibly you could find these from whereever pip installed them)
```bash
sudo apt-get install git
git clone <path to ModernGL.git>
```

