+++
title = "Uprooting ROOT Part III"
date =  2021-04-29
description = "If you work in HEP, you might just hate ROOT. Let's talk about replacing it. FINAL POST"
+++

> <i>
> This post highlights work done during my graduate studies. This particular project attempted to replace the data storage and quick histogram generation/viewing features of ROOT. The data format and associated viewer were able to replace ROOT, at least for my use case.  I decided, however, not to use the format for my thesis work.  At that point too much of my tool chain and analysis pipeline required ROOT, and fully switching would have been more costly than it was worth. However, if I were to do another data analysis I would seriously contemplate constructing a custom format. You can find the final product ere, https://github.com/TKGgunter/PhysicsDataSuite.
> </i>

Through the course of this blog series I've discussed data formats and x11.
This blog post will show off the program/library I made using that information.
The code can be found [here](https://github.com/TKGgunter/PhysicsDataSuite).


# Interfacing with the file format

As stated in the initial blog the file format was originally made to replace ROOT.
As such one would need to interface with it like one might ROOT. 
HEP scientist often work on a event by event model.
I wanted to keep this interface as simple as possible. 
Below is an example for initializing our data file.

```
    SerialDataFormat sdf;
    std::map<std::string, float>        float_map;
    std::map<std::string, int>          int_map;
    std::map<std::string, std::string>  string_map;

    sdf.vars_float  = &float_map;
    sdf.vars_int    = &int_map;
    sdf.vars_str    = &string_map;

    float_map["grade"]  = 0.0;
    float_map["height(cm)"]  = 0.0;
    int_map["age"]      = 0;
    string_map["name"]  = "";

    stage_variables( &sdf );

```

`stage_variables` primes the system to know which inputs to look out for.
When updating the events one might program the following;

```
    while (true){
        float_map["grade"] = 10.0;
        float_map["height(cm)"] = 0.5;
        int_map["age"]     = 20;

        update_buffers( &sdf );
        ///....
    }

```

`update_buffers` pushes the new values to buffers held by sdf.
It's a pretty easy set up can can be extended with some additional functions or macros.

There are some issues with this setup. One being the lack of compile time or runtime checking for spelling mistakes that can occur during when updating the maps. 
Runtime errors can be added with the use of macros or functions, but compile time errors would be preferable.  
If reflection were available in C/C++ compile time errors could more easily be a thing.


# Data viewer 
The data view is an extremely stripped down version of what ROOT does. 
This is because ROOT does too much.
Often this additional functionality ROOT provides gets in the what of common task.
The viewer list all features in file, and plots histograms of the distributions, allows for some simple overly functionality, logy plotting.
A video of the viewer is below.


<iframe src="../../sdfviewer.mp4" type="video/mp4" width="500px" height="300px">
</iframe>


If I were to use this regularly I'd add a few additional features. 
3 and 2 sigma cut offs, toggle able colors would among them. 

# Conclusion

I had a blast making this a few years back, and it's been fun reviewing the code and writing the blog post. 
I hope you enjoyed reading it. Thanks for reading.





