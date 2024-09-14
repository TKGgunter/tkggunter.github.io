+++
title = "Uprooting ROOT Part II"
date =  2020-03-07
description = "Learning how to interface with X11 to build a simple data viewer for our simple data format."
+++

> <i>
> This post highlights work done during my graduate studies. This particular project attempted to replace the data storage and quick histogram generation/viewing features of ROOT. The data format and associated viewer were able to replace ROOT, at least for my use case.  I decided, however, not to use the format for my thesis work.  At that point too much of my tool chain and analysis pipeline required ROOT, and fully switching would have been more costly than it was worth. However, if I were to do another data analysis I would seriously contemplate constructing a custom format. You can find the final product ere, https://github.com/TKGgunter/PhysicsDataSuite.
> </i>


Last time, [link](@/uproot/UpRootingI.md), we discussed and built a simple data format to hold high energy physics(HEP) data. 
Today we will be discussing X11 and how we can use it to create a simple data viewer. 
This post is intended to be an introduction X11, a way to help folks get started using a tool that has a rather negative reputation.
It is not intended to be a step by step guide to building the data viewer, nor is it a comprehensive guide to X11.
**Disclaimer: I am not a X11 expert. There will be more efficient ways of doing things, this is only meant to help readers get started**

X11 was chosen because it is the most used window management system on Linux. And Linux is THE operating system for CERN and much of the HEP community.


## Goals:

I've used the term data viewer, quite a bit over course of this blog series, but have failed to define what I mean.
For my needs a data viewer gives a user the ability to view histograms of saved features, and occasionally plot multiple histograms on top of each other.
This functionality requires a few features from X11. They are as follows:
- window creation
- draw rectangle
- draw characters
- mouse interaction.


We'll be cover the basics of window creation, drawing and event gathering.
For anyone who wants to skip a head you can learn a lot from Brian Hammond's example [code](https://fruttenboel.verhoeven272.nl/mocka/arch/hi.c). 
I will not be copying Hammond's example, but there will be enough over lap such that if you feel comfortable with the code above you need not bother with this blog.


## X11 basics:

To begin we need to initialize a window.
To do this we first request access to the window management system. The X server is x11's window management system.
Note, every operating system has some window management system; learning any management system is good preparation for learning others.
X server is an unique service as it allows for local and remote window management. 
When working remotely x server will connect via TCP or DECnet communication channels.
These connections can be rather low in bandwidth, unlike desktop broad casting, and is often an acceptable means conduct analyses.



The `X11/Xlib.h` contains most X11 functions and macros we are interested in. 
`XOpenDisplay` is used to gain access to the X server.
We'll be using the return value of this function quite a bit, so it's best to place it in some general scope.
You can find the documentation to `XOpenDisplay` and any function/constant we might use [here](https://tronche.com/).

The next want a window.
`XCreateSimpleWindow` can be used to create one.
`XCreateSimpleWindow` inherits properties from the specified root window. If you'd like to set these properties yourself you can use [XCreateWindow](https://tronche.com/gui/x/xlib/window/XCreateWindow.html).
To get the default root window call `DefaultRootWindow`.


Now that we have asked the X server to generate a window, we must now ask that window to be mapped to the display and give it an Atom.
We can map the window using 'XMapWindow'.
Attaching the Atom is done using 'XSetWMProtocols'.

Our current main function is as follows:

```
#include <X11/Xlib.h>

int main(int n_args, char** args){
    Display *dis;
    int screen;
    Window win;
    GC gc;
    static Atom wm_delete_window;
    int window_width  = 1000;
    int window_height = 600 ;

    {//Initialization
        dis=XOpenDisplay((char *)0);

        win=XCreateSimpleWindow(dis, DefaultRootWindow(dis), 0, 0, window_width, window_height, 0, 0, 0);

        XMapWindow(dis, win);

        wm_delete_window = XInternAtom(dis, "WM_DELETE_WINDOW", False);
        XSetWMProtocols(dis, win, &wm_delete_window, 1);
    }

    while (true) {
    }
}

```

To compile and run you'll need to run something like the following in terminal, `g++ PROGRAM_NAME.c -o PROGRAM_NAME -lX11 -std=c++11 && ./PROGRAM_NAME`.
Running this code should result in an empty black window.


Now that we have a window we should attempt to draw something.
x11 contains a plethora of basic drawing tools. 
Lets begin with drawing a rectangle.
`XDrawRectangle` is the x11 function used to create a rectangle.
if you check the arguments you'll notice one argument we have yet to satisfy, GC. 
GC stands for graphical context.
Graphical context are required for all draw calls.
A graphics context can be obtained through the use of `XCreateGC`.
With a GC we can now draw a rectangle.



Drawing strings can be done in a similar manner. `XDrawString`, is the x11 function used to draw text.
If you are coming from a non C background remember this is a C api. Strings must be passed via a character pointer.
The length of the string is passed separately. 


If you've been following along you will have noticed that none of your commands are drawn.
This is because your commands have not been exposed.
To allow for periodic expose events, I believe these occur when ever the x server feels, we need to include a selection mask.
`XSelectInput` is the function we are interested in. The `ExposureMask` is required to updated the window.


Lets take another  look at the code now.
```
#include <X11/Xlib.h>



int main(int n_args, char** args){
    Display *dis;
    int screen;
    Window win;
    GC gc;
    static Atom wm_delete_window;
    int window_width  = 1000;
    int window_height = 600 ;

    {//Initialization
        dis=XOpenDisplay((char *)0);

        win=XCreateSimpleWindow(dis, DefaultRootWindow(dis), 0, 0, window_width, window_height, 0, 0, 0);
        XSelectInput(dis, win, ExposureMask);

        XMapWindow(dis, win);

        wm_delete_window = XInternAtom(dis, "WM_DELETE_WINDOW", False);
        XSetWMProtocols(dis, win, &wm_delete_window, 1);

        gc = XCreateGC(dis, win, 0, 0);
    }


    XEvent event;
    while (true) {
        XNextEvent(dis, &event);
        {
            int x = 50;
            int y = 50;
            int w = 50;
            int h = 50;

            XSetForeground(dis, gc, 0xffffffff);
            XDrawRectangle(dis, win, gc, x, window_height - y - h, w, h);
        }
    }
}

```

Executing the code above an outline of a rectangle should be displayed at the bottom of the screen.

Now that you've drawn a few things you might notice that the axis used by x11 is left handed.
Meaning that origin, (0, 0), is found in the top left corner of the window instead of the bottom left.
If you find this hard to use you can of course create a wrapper function.


The final step requires get events.  Events are interactions with mouse, keyboard, or other peripherals they can also be generated from code by things like Exposure events.
To extend the number of events we interact with we need include additional event types in our input mask. Current we are only accepting exposure events. 
To increase we need to bitwise or `ExposureMask` with a few other masks we are interested in. 
For this project I want to set things up for mouse events. `ButtonPressMask` is needed for this. (If you want to do things with keyboard events `KeyPressMask` is needed.)


Your `XSelectInput` call should look something like the following, `XSelectInput(dis, win, ExposureMask|ButtonPressMask);`.
With that we can gather mouse events. You can use the tronche documentation elucidate yourself to the members of the XEvent.
The final example program is as follows. I won't spell out the rest. Hopefully this has been enough of a guide to get started. Good luck.
If you would like to check out the full data viewer you can find it [here](https://github.com/TKGgunter/PhysicsDataSuite).

```
#include <X11/Xlib.h>

int main(int n_args, char** args){
    Display *dis;
    int screen;
    Window win;
    GC gc;
    static Atom wm_delete_window;
    int window_width  = 1000;
    int window_height = 600 ;

    {//Initialization
        dis=XOpenDisplay((char *)0);

        win=XCreateSimpleWindow(dis, DefaultRootWindow(dis), 0, 0, window_width, window_height, 0, 0, 0);
        XSelectInput(dis, win, ExposureMask|ButtonPressMask|ButtonReleaseMask);


        XMapWindow(dis, win);

        wm_delete_window = XInternAtom(dis, "WM_DELETE_WINDOW", False);
        XSetWMProtocols(dis, win, &wm_delete_window, 1);

        gc = XCreateGC(dis, win, 0, 0);
    }


    XEvent event;
    while (true) {


        while (XPending(dis) != 0) {

            XNextEvent(dis, &event);
            if((Atom)event.xclient.data.l[0] == wm_delete_window){
                return 0;
            }
            if (event.type == Expose || event.type==ButtonRelease) {
                XClearWindow(dis, win);
            }
            if (event.type==ButtonPress) {//Hello world 
                int x = 400;
                int y = 300;
                int w = 90;
                int h = 50;

                XSetForeground(dis, gc, 0xffffffff);
                XDrawRectangle(dis, win, gc, x, window_height - y - h, w, h);
                XDrawString(dis, win, gc, x+12, window_height - y - h/2, "Hello World", 11);
            }
        }

        {
            int x = 50;
            int y = 50;
            int w = 50;
            int h = 50;

            XSetForeground(dis, gc, 0xffffffff);
            XDrawRectangle(dis, win, gc, x, window_height - y - h, w, h);
        }
    }
    return 0; 
}
```


The next and final article in this series will show off the final product. Thank you for reading. Stay tuned.




