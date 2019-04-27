---
layout: post
title:  "Uprooting ROOT Part I"
date:   2018-10-21 08:17:28 -0500
categories: Uprooting ROOT
---
 
Root is THE underlying technology that backs many modern particle physics experiments.
Self described as a modular scientific software framework, I take issue with this description, it 
deals with everything from data visualization to statistical analysis and data storage.
 
 
PhD students in particle physics one are expected to be able to use this toolset in some capacity.
Most modern analyses rely heavily on ROOT.
I however, don't use ROOT in my data analysis workflow.
I opt for python, using a combination of matplotlib, scipy, numpy and sklearn for my visualisation and analysis needs.
I still rely on ROOT dependency when transforming data from the root format into something a bit more palatable.

I am not alone in this shift. 
Many physicist are moving away from ROOT.  
Personally the fragility and friction that occurs when exploring datasets are ROOT's main draw backs.
It should be noted that ROOT is under active development and the team as addressed any stability issues.

 
So why not remove ROOT entirely. Why continue to have the root dependency then I only use to read data stored in the root format.
Well there is no good answer. This collection of blog posts detail the design and implementation of a dataformat the I have replaced ROOT with.
I hope this blog can be educational and demystify one of the basic technologies we use on a daily bases.
 
Section I: What makes a file format

Constructing a custom file format isn't all that difficult.
I find that the most important concept is to know that your data is and how you are going to interact with it.

To begin lets discuss data, what are we going to store.
I work in the high energy domain of particle physics.
In this domain we are usually interested in distributions of floating point and integer numbers.
Scientist in my field often organize data in units of events.
In this case an event is the measurements made after two protons collide with each other.
An easy way to think of this is through the comma separated value model.
Where a row is an event and the columns can be thought of as the measurements made during an event (called features from here on).
These features aren't necessarily known a priori and the list often grow during the course of a project.

<table>
	<tr> 
		<th>feature1
		<th>feature2
		<th>...
		<th>featureN
	</tr> 
	<tr> 
		<td> 123.4
		<td> 1
		<td>...
		<td> "type1"
	</tr> 
	<tr> 
		<td> 13.4
		<td> 12
		<td>...
		<td> "type3"
	</tr> 
</table>

 
 
For my analysis I require:
32 bit floating point (high precision isn't important so maybe this can be 16bit),
32 bit integers (I could probably get away with 8 bits but lets keep things symmetric),
short strings (something less than a tweet, 280 characters).
 
 
In addition we must account for a high volume of data.
Analysis in high energy physics can easily include millions of events and so the memory foot print must be small atleast as small as ROOT.
 

Now that we have an idea of the problem at hand we lets talk more about file formats.

The best way to learn the building blocks of constructing a file format is looking at the spec of an some other format like BMP or WAV.
I will not doing an in depth review of these formats, but there is a terrific synopsis at https://gamedevelopment.tutsplus.com/tutorials/create-custom-binary-file-formats-for-your-games-data--gamedev-206.
 
 
 
TO begin file formats are not magical.  They are specification that assign meaning to a particular collection of bits and bytes.
One can in fact look directly at a binary file with your standard editor like vim.
If we open a ROOT file you'll notice that the first four characters(bytes) decode to "root".  Most files are like this.
 If you open an executable the first three characters are probably "ELF" and a bitmap is "BM".
These notations give the os and programs an understanding of how the file is formatted.
Continuing through the file we notice a ton of random characters, this is mostly binary, though occasionally there will be some human readable text.
If you haven't done this already I would suggest going through a few binary files on your own time to get an intuition of how these files are constructed.
 
 
So I'll layout a general spec and we'll fill in the details was we go.
 
The beginning bytes of a file generally tell let the reading program know what that data that follows is and our that data is formatted, we just went over this.
Like BMP and WAV files we'll begin the file with a format tag.
As mentioned earlier we want the have small files, so we'll need to note the compression of the of file.
Next I want communicate how our data is stored, meaning we need to dictate the number of features stored, feature names, total number of events stored, etc.
 
I'm thinking something like this:

<table> 
	<tr>
		<tb> Signature
		<tb> 24 bit
		<tb> "T", "D", "F"
	</tr>
	<tr>
		<tb> Compression flag
		<tb> 8 bit
		<tb> int
	</tr>
	<tr>
		<tb> Offset
		<tb> 32 bit
		<tb> int
	</tr>
	<tr>
		<tb> number of features
		<tb> 32 bit
		<tb> int
	</tr>
	<tr>
		<tb> number of events
		<tb> 32 bit
		<tb> int
	</tr>
</table> 


Signature is our file format tag. I've chosen the characters "T", "D", "F", for Thoth's data format. 
Yup super original. When we eventually read these files we'll first check the first three bytes of the file to determine if we are reading the correct file format.
At this point I should note that there are often formats that share the same signature, https://software.broadinstitute.org/software/igv/TDF for example. 
The last thing to note is the 24 bit allocated to this specification. Of course this is the because ASCII characters require 8 bits.

Compression flag is as the name suggest. The value held here describes how that data is compressed.

Offset describes when the header ends and when the contiguous data blocks begin.

Number of features and number of events are as the names suggest. It should be noted that by having these be 32 bit integers the maximum number of features and events is 2 billion, which we probably will not hit in features, but may very well hit in number of events.  For now I'm going to say that hitting this limit is very unlikely. 
 
 
Section II. Data :
Now it's time for some data.
Specifically the list of features we're saving, their names, associated type info among other things.
Lets start with strings.

Strings can be weird depending on the language you come from.
In C strings are an array of bytes that are terminated by a zero at the end of the array.
Without this zero C will just read forever.
This method is one of the most compact ways to store a string but its also the most error prone.
Another way to store as string is a length and the array of characters.
This is far less error prone but can be a fair bit larger, especially for short strings.
I'm going to choose the less error prone route for this this task.
This means feature names will be a 32bit int then a character array.

We also need to store type information.
This is pretty easy we have only 3 types we are interested in storing, ints, floats and strings.
There is a bit more information I'd like the store here.
The total number of bytes used to store the data for that feature over all events.
This additional information has a bit todo with the overall structure of our file which I'll explain now. 

ROOT is stores data in a event model. 
You can think about it as rows of a csv.
For example lets say you have the following dataset.

| Feature1 | Feature2 | Feature3
---------------------------------
1.0        |  1231    |  "alpha"
34.3	   |   11     |  "alpha"
12.3       |   231    |  "beta"


Data would be stored 1.0, 1231, "alpha", 34.3, 11, "alpha"...
This structure means its easy to maintain event order when reading out data.
This type of structure is extremely helpful when dealing with detectors, or time ordered data.
However most of my analysis are not like. 
They are more interested in the distribution or shape of a feature over many events.
For this it is better to store features in contiguous chunks so 1.0, 34.3, 12.3... for feature1 then 1231, 11, 231, for feature2... and so on and so forth.

 
So lets review how our feature list might look.
(feature name {string}, type{8 byte int}, buffer size{32 byte int}), ...
 
This format spec rather exactly meets the description of what I need.
The format allows for easy access to the number of events the file holds.
It also allows for great are given great flexibility.
 
 
The only things missing is the data.
We're going to store the data in contiguous arrays....
There's not much else to say.  One can think of it as a list of contiguous arrays where each array holds the data for a given feature.

That's all for now.
Next time I'll cover implementation and detail the String spec for our data.


