---
title: A Hyprland User's Guide to NVIDIA GPUs
---

How do applications actually draw the contents on a screen? 
The OS provides a Display Server that acts as a translation layer between the applications and the GPU to "draw" items on the screen. This is how X11 used to work but it brought in a lot of overhead. This is where wayland comes in, it eliminates the idea of a display server and instead brings in a protocol for communication between applications and the compositor. What is a compositor? The
compositor assumes a dual role: it is both a compositing window manager and the display server itself. It receives off-screen buffers from each application, applies graphical effects or
transformations, and then composites them into a final image to write to the display memory. Hyprland is a compositor!

