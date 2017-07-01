1st July 2017

Some notes on headless rendering on a raspberry pi

Follow [caffinc's guide](https://caffinc.github.io/2016/12/raspberry-pi-3-headless/) to set up your pi without a screen or keyboard

install python3 

```bash
sudo apt-get update
sudo apt-get install python3
sudo apt-get install python3-dev
sudo apt-get install python-pip
```
and then  [ModernGL](https://github.com/cprogrammer1994/ModernGL)

```bash
sudo pip install ModernGL
```
but that can't find Python.h ...
