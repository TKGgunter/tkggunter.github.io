+++
title = "Uprooting ROOT Part I"
date =  2019-04-27
description = "If you work in HEP, you might just hate ROOT. Let's talk about replacing it."
+++


> <i>
> This post highlights work done during my graduate studies. This particular project attempted to replace the data storage and quick histogram generation/viewing features of ROOT. The data format and associated viewer were able to replace ROOT, at least for my use case.  I decided, however, not to use the format for my thesis work.  At that point too much of my tool chain and analysis pipeline required ROOT, and fully switching would have been more costly than it was worth. However, if I were to do another data analysis I would seriously contemplate constructing a custom format. You can find the final product ere, https://github.com/TKGgunter/PhysicsDataSuite.
> </i>
 
Root is THE underlying technology that backs many modern particle physics experiments.
Self described as a modular scientific software framework, I take issue with this description, it 
deals with everything from data visualization to statistical analysis and data storage.
 
 
PhD students in particle physics one are expected to use this toolset in some capacity.
Most modern High Energy Physics (HEP) analyses rely heavily on ROOT.
I however, don't use ROOT in my data analysis workflow.
I opt for python, using a combination of matplotlib, scipy, numpy and sklearn for my visualisation and analysis needs.
I still rely on ROOT dependency when transforming data from the '.root' format into something a bit more palatable.

I am not alone in this shift. 
Many physicist are moving away from ROOT.  
Personally the fragility and friction that occurs when exploring datasets are ROOT's main drawbacks.
It should be noted that ROOT is under active development and the team has addressed many if its stability issues.

 
So, why not remove ROOT entirely? Why continue to have this dependency when I only use it to read and store data.
Well, I have no good answer other than it's there. This blog post, and perhaps others will detail the design and implementation of a dataformat I created to replaced ROOT.
I hope this blog can be educational and demystify one of the basic technologies we use on a daily bases.
 
## Section I:  What makes a file format?

Constructing a custom file format isn't all that difficult.
It requires some general understanding of how computers work, your data and how you use that data.


To begin lets discuss data, what's being stored.
I work in the high energy domain of particle physics.
In this domain we are usually interested in distributions of floating point and integer numbers.
Scientist in my field often organize data in units of events.
In this case an event is the measurements made after two protons collide with each other.
An easy way to think of this is through the comma separated value model, your typical csv file.
In this model a row is an event and the columns can be thought of as the measurements made during an event (called features from here on).
The number of features aren't necessarily known a priori and the list often grows during the course of a project.
The same is true for the number of events.

<table>
	<tr> 
		<th>feature1 </th>
		<th>feature2 </th>
		<th>... </th>
		<th>featureN </th>
	</tr> 
	<tr> 
		<td> 123.4 </td>
		<td> 1  </td>
		<td>...  </td> 
		<td> "type1" </td>
	</tr> 
	<tr> 
		<td> 13.4 </td>
		<td> 12 </td>
		<td>... </td>
		<td> "type3" </td>
	</tr> 
</table>

 
 
For my analysis I require:
32 bit floating point (high precision isn't important so maybe this can be 16bit),
32 bit integers (I could probably get away with 8 bits but lets keep things symmetric),
short strings (something less than a tweet, 280 characters).
 
 
In addition we must account for a high volume of data.
Analysis in high energy physics can easily include millions of events. 
For this format to be useful it must have a memory foot print comparable to ROOT's.
 

Now that we have an idea of the problem lets talk more about file formats.
A file format is a kind of design document that defines the meaning of the bits within a file.
The best way to learn about building "document" is by studying the spec. of an some other format like BMP or WAV.
I will not doing an in depth review of these formats, but there is a terrific synopsis at [link](https://gamedevelopment.tutsplus.com/tutorials/create-custom-binary-file-formats-for-your-games-data--gamedev-206).
 
 
 
To begin file formats are not magic.  They are a specification that assigns meaning to a collection of bits and bytes.
One can in fact look directly at a binary file, with a standard editor like vim, and often deduce the format through visual inspection.
If we open a ROOT file you'll notice that the first four characters(bytes) decode to "root".  Most files are like this.
 If you open an executable the first three characters are probably "ELF" and a bitmap is "BM".
These notations give the operating system and programs a way to determine how the file should be decoded, what format they should assume the file is in.
Continuing through the file we notice a ton of random characters, though occasionally there will be some human readable text.
If you haven't done this already I would suggest going through a few binary files on your own time to get an intuition of how these files are constructed.
 
 
Now that we are on the same page, lets begin laying out a general header specification. 
As stated previously the beginning bytes are reserved for indicating the file format.
Like BMP and WAV files we'll begin the file with a format tag.
Next I want communicate how our data is stored, the number of features stored, feature names, total number of events stored, etc.
 
I'm thinking something like this:

<table> 
	<tr>
		<td> Signature </td>
		<td> 24 bit </td>
		<td> "T", "D", "F" </td>
	</tr>
	<tr>
		<td> Compression flag </td>
		<td> 8 bit </td>
		<td> integer </td>
	</tr>
	<tr>
		<td> Offset </td>
		<td> 32 bit </td>
		<td> integer </td>
	</tr>
	<tr>
		<td> number of features </td>
		<td> 32 bit </td>
		<td> integer </td>
	</tr>
	<tr>
		<td> number of events </td>
		<td> 32 bit </td>
		<td> integer </td>
	</tr>
	<tr>
		<td> feature names </td>
		<td> ?? bit </td>
		<td> string </td>
	</tr>
	<tr>
		<td> feature type and size</td>
		<td> 8 bit and 32 bit</td>
		<td> integer and integer </td>
	</tr>
</table> 


Signature is our file format tag. I've chosen the characters "T", "D", "F", for Thoth's data format. 
Yup super original. When we eventually read these files we'll first check the first three bytes of the file to determine if we are reading the correct file format.
At this point I should note that there are often formats that share the same signature, [TDF](https://software.broadinstitute.org/software/igv/TDF) for example. 
The last thing to note is the 24 bit allocated to this specification. Of course this is the because ASCII characters require 8 bits.

Compression flag is as the name suggest. The value held here describes how that data is compressed.
Offset describes when the header ends and when the contiguous data block begin.
Number of features and number of events are as the names suggest. It should be noted that by having these be 32 bit integers the maximum number of features and events is 2 billion, which we probably will not hit in features, but may very well hit in number of events.  For now I'm going to say that hitting this limit is very unlikely. 

You may have noticed feature names have no bit size and is not a standard Clang data type. 
In fact Strings can vary greatly depending on the language you develop in.
In C strings are an array of bytes that are terminated by a trailing zero.
Without this zero most C programs will assume that the string goes on forever.
The trailing zero, method for structuring a string, is one of the most compact ways to store a string but its also the most error prone.
Alternatively on could store string as the length of the string, number of characters in the string, followed an array of the characters.
This is far less error prone but can be a fair bit larger, especially for short strings.
I'm going to choose the less error prone route for this this task.
This means feature names will be a 32bit int then a character array. 
I will be storing characters as 8 bit integers like out file signature.

We also need to store type information.
This is pretty easy we have only 3 types we are interested in storing, ints, floats and strings.
There is a bit more information I'd like the store here.
The total number of bytes used to store the data for that feature over all events.
This additional information has a bit todo with the overall structure of our file which I'll explain now. 
 
 
## Section II. Data 

ROOT stores data in a event model. 
You can think about it as rows of a csv.
For example lets say you have the following dataset.


| Feature1 | Feature2 | Feature3|
|----------|----------|---------|
|1.0       |  1231    |  "alpha"|
|34.3	   |   11     |  "alpha"|
|12.3      |   231    |  "beta" |


In root data would be stored 1.0, 1231, "alpha", 34.3, 11, "alpha"...
This type of structure is extremely helpful, and natural when dealing with detectors, or time ordered data.
However most HEP analyses are not like. 

HEP analyses are more interested in distributions or the shape of the data, for a given feature over many events.
Contiguous chunks of data, as in storing columns contiguously, is much more useful.
This is because computers can load data from these contiguous chunks far quicker than it can mixed data.
If you've ever worked in pyroot and gone between ROOT files and HDF files you'll experience this directly. 
Loading ROOT files are at least 5-10x slower than HDF files. HDF files much like our format is column based.
Write this data is simply writing the arrays to disk in the order we've listed the features in the header.
 

That's all for now.
Next time I'll cover implementation of a simple data viewer for linux. Thank you for your time.

>If you have any corrections please contact me through github and let me know. 
>I am not a ROOT engineer so I may have gotten some of the details wrong.


