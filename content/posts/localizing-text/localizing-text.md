---
title: "Localizing Text"
date: 2024-08-04T17:41:30-07:00
draft: false
toc: false
images:
tags: 
  - engine
  - tools
  - localization
---

Localization is one of those things many people in game dev talk about but there doesn't seem to be a consensus on whether it is valuable to do in your games. For every presentation or post-mortem talking about the benefits of it and how it was key to their success you can find an equal number stating it didn't help. With that in mind, I'm not entirely sure if I will localize all the text in the game I'm making but that doesn't mean that I did not make a system for localization just in case I do decide to do so. I don't believe the game will have an excessive amount of text so the solution I built for localization is not that complex and I will outline it here.

So how do we make a localization system that is easy to add new languages and is performant?

In short, we make a table loaded with all the localized text and we use unique IDs to lookup the correct translations. Few things are faster than indexing into an array so this is what is used to store all the translated strings instead of something like a hash table. Using a hash table also has the downside of needing to hash the key in order to get the localized text back so there's a performance cost that can be easily avoided by using an array as the underlying data structure to store the text. With a bit of tooling we can automate the process of creating these unique IDs offline and then use them at runtime so we never need any sort of hashing to access to correct localized text.

At the start of the application the localization data of the desired language is loaded into memory where a static object will store the array containing all the localized text. Having it be globally accessible through an API makes it very easy to use anywhere in the application. It is never modified after loading so there is also no worry about accessing that data from a different thread. The unique IDs for each localized text is stored in a auto-generated header file where each ID is a macro that contains an unsigned integer.

That's pretty much all there is to using the localization system so lets take a look at how the authoring side of the system works.

The engine has a localization folder where the user may place json files containing the localization data for their project. The name of each of those files is very important as the name has to be the [language code](https://en.wikipedia.org/wiki/Language_code) of whatever language the data in the file represents. For instance, US English file will be denoted by the language code **en-US**. At the moment it is assumed that the file will contain all the localized text needed in the game. It was mentioned that file itself uses the json format but only because other systems were using json files as well and not because it provides any benefit. The localization file is essentially a key value pair listing the identifier of the localized text and its actual text.

```
Example of an entry in the localization file
{
  "Setting_GraphicsTab": "Graphics",
  "Graphics_Resolution": "Resolution",
  ...
}
```

Tempest has a separate executable, referred to as the Localizer, which will parse the project's localization folder. The output is the header file containing up to date localization IDs for each text in the localization file and binary file containing the localized text for optimal serialization/deserialization tasks later on. So continuing from the example above, a sample output of the header file looks like this:

```cpp
#define LOC_SETTING_GRAPHICSTAB (73u)
#define LOC_GRAPHICS_RESOLUTION (74u)
...
```

Then in order to get the localized text from that ID we can utilize the localization API like the example below.

```cpp
const LocalizedString& graphicsTabText = Localization::Localize(LOC_SETTING_GRAPHICSTAB);
// Use localized text as needed
```

The one downside of this approach is that any time a localized text entry is added or removed, the header file has to be regenerated and this means compiling the engine. For now that isn't too much of an issue but it is an area that could be improved.

Finally, there is a batch file available in the engine to make it easy to run the localizer tool but this is also something that could be improved. For now though this works well with my workflow so there aren't any plans to update this process.

One last thing is that the in-game console has a helper command to trigger a full reload of the localization data. This is very useful during development when only the content of the localized text is modified and no entries are added or removed.

And that is pretty much the totality of the localization system inside of Tempest. As noted at the start, it is a fairly simple solution but still pretty powerful.