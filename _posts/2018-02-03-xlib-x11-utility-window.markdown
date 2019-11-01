---
layout: post
title: "Creating a utility window using properties in Xlib/X11"
date: 2018-02-03
categories:
---

# Creating a utility window using properties in Xlib/X11
[Yarn](https://github.com/Toqozz/yarn) is a notification daemon designed to be a more visually featureful alternative to the beloved [dunst](https://github.com/dunst-project/dunst).  It was important to me that Yarn be at least as lightweight as dunst, which meant avoiding GTK/Qt and drawing with Xlib/Cairo instead.

The main thing this meant was that I had to interface with X11 directly.  I wanted a pure X11 utility window -- no window manager positioning, no window borders, just compositor shadows.

![Pure utility window](/assets/2018_pure_utility_window.png)
*Pure X11 utility window (bspwm).*

The advantage of a window like this is that it can be placed anywhere by us, and resized independently of the window manager (assuming the WM is standards compliant).  While this approach might not be the most elegant depending on your sentiment, it's the only real way to have a good level of customization and control over your window.  This type of solution is also likely to be necessary for achieving a fullscreen mode in some 3D application, or similar.

---

To understand what goes into this, we should first have some knowledge on how X11/Xlib actually interacts with the window manager.  
In the basic sense, when we first create a window using `XCreateWindow()`, it creates an object with a number of default parameters.  Later, the window manager reads these parameters and decides exactly how to show the window.  As long as your window manager supports the parameters / hints that you want to use, we simply need to provide the WM with these hints.

When creating a new window with Xlib, it typically looks a little like the following:
```c
// Return a window.
Drawable *
create_x11_window(int x, int y, int w, int h) {
    Display *display;
    Drawable drawable;
    int screen;   // Screen #.

    // Error if no open..
    if ((display = XOpenDisplay(NULL)) == NULL)   // Set display though.
        exit(1);

    screen = DefaultScreen(display);   // Use primary display.
    XVisualInfo vinfo;
    // Match the display settings.
    XMatchVisualInfo(display, screen, 32, TrueColor, &vinfo);

    XSetWindowAttributes attr;
    // We need all 3 of these attributes, or BadMatch: http://stackoverflow.com/questions/3645632/how-to-create-a-window-with-a-bit-depth-of-32
    attr.colormap = XCreateColormap(display, DefaultRootWindow(display), vinfo.visual, AllocNone);
    attr.border_pixel = 0;
    attr.background_pixel = 0;

    // Returnts a window (a drawable place).
    drawable = XCreateWindow(display, DefaultRootWindow(display),
            x,y,     // Position on screen.
            w,h,     // Width, Height.
            0,       // Border width.
            vinfo.depth, InputOutput, vinfo.visual,   // Depth, Class, Visual type.
            CWColormap | CWBorderPixel | CWBackPixel, // Overwritten attributes.
            &attr);

    // Apply the Atoms to the new window.
    // Request that the X server report these events.
    x_set_wm(drawable, display);
    XSelectInput(display, drawable, ExposureMask | ButtonPressMask | KeyPressMask);

    return drawable;
}
```

And the property magic is performed in our `x_set_wm()` function:
```c
// Apply atoms to window.
static void
x_set_wm(Window win, Display *dsp) {
    Atom property[3];  // Change 2 things at once, (parent + 2 children).

    // Set window's WM_NAME property.
    XStoreName(dsp, win, "yarn");
    // No children.
    property[2] = XInternAtom(dsp, "_NET_WM_NAME", false); // Get WM_NAME atom and store it in _net_wm_title.
    XChangeProperty(dsp, win, property[2], XInternAtom(dsp, "UTF8_STRING", false), 8, PropModeReplace, (unsigned char *) "yarn", 4);

    // Set window's class.
    XClassHint classhint = { "yarn", "yarn" };
    XSetClassHint(dsp, win, &classhint);

    // Parent.
    property[2] = XInternAtom(dsp, "_NET_WM_WINDOW_TYPE", false);   // Let WM know type.
    // Children.
    property[0] = XInternAtom(dsp, "_NET_WM_WINDOW_TYPE_NOTIFICATION", false);
    property[1] = XInternAtom(dsp, "_NET_WM_WINDOW_TYPE_UTILITY", false);
    // Reach for 2 longs, (2L).
    XChangeProperty(dsp, win, property[2], XA_ATOM, 32, PropModeReplace, (unsigned char *) property, 2L);

    // Parent.
    property[2] = XInternAtom(dsp, "_NET_WM_STATE", false);   // Let WM know state.
    // Child.
    property[0] = XInternAtom(dsp, "_NET_WM_STATE_ABOVE", false);
    // Reach for 1 long, (1L).
    XChangeProperty(dsp, win, property[2], XA_ATOM, 32, PropModeReplace, (unsigned char *) property, 1L);
}
```

This function demonstrates the general method of assigning a few properties to your window;
1. Get the atom of the property/properties that you want to set or add using `XInternAtom()`.
2. Change / add to the window property using `XChangeProperty()`.

I've made an effort to comment the code heavily so that it's easy to understand, particularly when viewed beside the Xlib function documentation.  Hopefully this is enough to make reasonable sense of the situation.

---

One thing that was initially confusing to me was finding which atoms were available to use, and then figuring out which atoms I should actually use to accomplish what I wanted.

To my understanding, properties starting with "`_NET`" are part of the newer XDG/Freedesktop window manager specification.  There are a lot more of these, and they are generally preferred over legacy variants.

Freedesktop has a list of the available (newer) properties [here](https://specifications.freedesktop.org/wm-spec/1.3/ar01s05.html), whereas the older properties can be found [here](https://tronche.com/gui/x/xlib/ICC/).

You can also use the `xprop` utility to see the properties of existing windows you're using.  This can be helpful in making complicated windows "just-so" and really understanding properties.  Under most distributions, `xprop` is available as the `xorg-xprop` package.

![Getting window properties with xprop](/assets/2018_xprop.png)

---

Hopefully from here you'll have all you need to explore this field on your own, and correctly propertize your windows.


