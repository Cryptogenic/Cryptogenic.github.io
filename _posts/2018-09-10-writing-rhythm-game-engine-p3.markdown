---
title:  "Rhythm (GH) Game Engine Part III - Chart Loading"
date:   2018-09-10 20:05
categories: []
tags: []


---

This post is in continuation of the "Rhythm Game Engine" mini-series. It's been a while since I've posted a blog post, and I'd paused development on this game engine for a while as well. There were two reasons for that, one of which was my development was broken up by a trip to Vegas for DefCon 26. The other reason was this part of the game engine was very challenging mentally - this part being loading charts and moving the notes in sync with the song track. How do we map the notes to Z-positions so that they're in-sync with the music? How do we calculate the rate at which to move the notes every frame to keep the track in-sync? All of these will be answered in this post.

##### This is a part of a blog mini-series

[Part I - Architecture](https://specterdev.ca/2018/writing-rhythm-game-engine-p1/)
[Part II - Model View Projection](https://specterdev.ca/2018/writing-rhythm-game-engine-p2/)

*Part III - Chart Loading*

------



## Chart Loading

The patterns of notes for songs played in games like Guitar Hero, Clone Hero, and other similar games are called "charts". Charts are typically written in one of two formats, the common .midi format or a less-common format, .chart. While official charts typically use .midi, most customs use .chart because it's easier to parse, and therefore easier to use for charting software such as Moonscraper.

Since .chart is the easiest to work with - it's what I decided to start with. Let's take a look at a simple chart that just has a standard 120BPM, 4/4 time signature, and 6 green notes, one every 1/8th bar.

```
[Song]
{
  Offset = 0
  Resolution = 192
  ...
}
[SyncTrack]
{
  0 = TS 4
  0 = B 120000
}
[ExpertSingle]
{
  96 = N 0 0
  192 = N 0 0
  288 = N 0 0
  384 = N 0 0
  480 = N 0 0
  576 = N 0 0
}

```

As you can see, the file is in plaintext format, so we don't need to worry about binary parsing. This has a few upsides and a few downsides. One upside is it's easier to work with as we don't have to reverse the format. A downside is it's slightly slower to parse plaintext. Another downside is this format is pretty much exclusively used in clone games like Clone Hero, so there's very little documentation on the format. I hope to change that with this article.

The `[Song]` and `[SyncTrack]` fields are fairly straight forward, so long as you know that "TS" stands for time signature and "B" stands for BPM - but what about the actual notes section? As we can see, each difficulty has it's own notes section, but since I've only charted the expert difficulty, the chart only has an expert section.

I'm going to shed some light on this format in this blog post - hopefully it helps others who want to create projects based on this format.



## The chart format

#### Ticks

First we should talk about what all these numbers are that are used on the right side of the equals sign in the `[Song]` section, and on the left side of the equals sign in the `[SyncTrack]` and note sections. I'm calling these "ticks", and they're essentially numerical positions of where things happen in the song. Usually, as soon as a song is started the song's tick counter will be set to `0` and will be incremented based on the song's BPM and time signature. However, if `offset` is non-zero, the song's tick counter will be initialized with a number less than 0, being `-offset`. 

#### Resolution

The `resolution` value will almost always be set to `192` - I've yet to see a chart with a different value. The name is a bit unintuitive though - what exactly is `resolution`? Simply put, it's how many ticks are represented by 1 bar divided by the time signature. This is a crucial piece of information for setting the rate at which the notes move to sync up with the song's audio, and we'll revisit it soon.

#### SyncTrack

The `SyncTrack` section essentially holds all the information for the song's BPM and time signature. In some songs, the BPM and time signature (TS) will change at multiple points in the song, though most songs have a constant BPM/TS set at the beginning. The number on the left denotes which tick this BPM/TS takes effect at. The string on the right side of the equals sign notes whether or not the BPM or the TS is changing, and the last number on the right is the magnitude of the change. The "/4" is assumed and isn't used for the TS, and the BPM is multiplied by 1000 (not really sure why though, probably some engine internals thing from the engine it was developed for).

#### Notes

Each difficulty has it's own notes section, so this information can be applied for each difficulty. Again, the number on the left side of the equals sign is the tick number, the information on the right is information on the note, let's put it into a generic formula.

```
[tick] = [Note Type] [Note Description] [Sustain Length]
```

The `Note Type` can be one of two characters, "N" or "S". "N" stands for "note" and is the most common type. "S" stands for "star power phrase", and is used to denote star power phrase sections. In this article, we'll ignore "S".

The `Note Description` is an integer which basically holds the note's flags - here's the possible values it can hold:

```
0 - Green Note
1 - Red Note
2 - Yellow Note
3 - Blue Note
4 - Orange Note
5 - Forced Note (Flag)
6 - Tap Note (Flag)
7 - Open Note
```

Oddly enough, if a note has multiple flags (say, it's a red note and also a HOPO (hammer on)), the tick will have two separate lines, one noting that it's a red note (1), the other noting that it's forced (5). This is a bit puzzling because it'd make more sense to combine all the flags into one value using bitwise, but it's what we have to work with.

The `Sustain Length` is the number of ticks that the note will sustain for. Most notes, this will be set to `0`, however sustained notes will have a non-zero value.

#### Parsing

So how do we parse all these notes and create objects for them? We could use C++ string streams, but I think I'd rather shoot myself in the foot than use string streams for this. The trusty libc function `sscanf_s` will be much less painful.

```cpp
unsigned int tick;
int description;
int noteLength;
char noteType;
std::string values;

// Trim whitespace
const regex whitespaces("^[ \t]");
values = regex_replace(line, whitespaces, "");

int cnt = sscanf_s(values.c_str(), "%u = %c %d %d", &tick, &noteType, sizeof(char), &description, &noteLength);
```



#### Mapping ticks to Z-values

Through experimentation, I found the formula for mapping ticks to Z-values on initial chart load is the following related to my camera setup:

```cpp
int zPos = 0.5 - (tick * 0.005);
```



## Note Translation

Once the song starts, we have to start moving these notes at a calculated rate so that the notes match up with the pace of the song (the tempo). Bafflingly, I couldn't really find any information on the Googles about this formula. I could have looked at open-source charting software and reversed pre-existing games, but the code bases were pretty messy, though I did do this as part of my research.

After looking at some open source charting programs and piecing a few other parts of the formula myself through experimentation, I derived the master formula for determining how many milliseconds need to elapse per-tick. Once we have that, moving the notes is easy, as using the above formula for mapping the ticks to Z-values, we can just plug and play the current tick into that formula.

There are a few chart-specific factors that we must consider for the formula. These factors are the `Resolution`, `BPM`, and `Time Signature`, which I talked a bit about above. Using these values we can calculate how many milliseconds it should take to move 1 bar. By extension, since we know how many ticks are in one bar via `Resolution`, we can ultimately determine how many milliseconds there are per-tick.

We can calculate how many minutes are in one bar by dividing the BPM by the Time Signature (TS). For example, if we had a 4/4 time signature with a 120BPM, we'd have `4 / 120`, which gives us `4 / 120 = 0.033 (repeating)`. That means it takes 0.033 minutes to complete 1 bar, aka ~2 seconds. By multiplying this number by 60, we can get how many seconds it takes to complete 1 bar. Multiply this number again by 1000, we get the milliseconds per bar.

```cpp
float mspb; // milliseconds per bar

mspb = (TS / BPM) * 60.0 * 1000.0;
```

The number of ticks that are in one bar can be calculated by multiplying the time signature by the resolution. By then dividing the `mspb` by this value, we can determine how many milliseconds are represented by one tick.

```cpp
float mspt; // milliseconds per tick

mspt = mspb / (resolution * TS);
```

As an example, if we have a 120BPM song with 4/4 Time Signature (and a resolution of 192), there will be 2000 milliseconds in 1 bar. Since there are `192 * 4 = 768` ticks in 1 bar, that means there are `2000 / 768 = ~2.60` milliseconds for every tick. Trying to render notes at this speed is kind of ridiculous, because that means the game would need to run at over 500FPS. On some machines this is possible if the game loop is efficient enough, on lower-end PC's this won't do. Since we don't need performance this amazing, I decided to multiply `mspt` by 4 and increment the tick counter by 4 every loop instead of 1. That leaves the note translation code as the following:

```cpp
float mspb, mspt;

mspb = (TS / BPM) * 60.0 * 1000.0;
mspt = mspb / (resolution * TS);

// Time in GLFW is measured in seconds, so we'll divide mspt by 1000
mspt /= 1000;
mspt *= 4; // Cut down FPS required

if(previousTime == 0)
	previousTime = glfwGetTime();

currentTime = glfwGetTime();

if (currentTime - previousTime >= mspt)
{
    for (auto it = notes.begin(); it != notes.end();)
	{
		Models::Note *note = *it;

		unsigned int noteTick = note->getTick();
		int tickDiff = noteTick - currentTick;

		// If tick is out of this frame - we don't care about it
		if (tickDiff > 500)
			break;

		float previousPositionX = note->getX();
		float previousPositionY = note->getY();
		float previousPositionZ = note->getZ();

		float newPositionZ = 0.5 - (tickDiff * 0.005) + (increment * (0.005));

		note->translate(previousPositionX, previousPositionY, newPositionZ);
        
        it++;
    }
    
    previousTime += mspt;
	currentTick += increment;

	scene->setCurrentTick(currentTick);
}
```

You'll notice I don't bother translating the notes that are too far away. This is purely a performance choice, as moving notes that the player can't see yet is an unnecessary perf hit - inside the game loop, every millisecond counts!



## Final Thoughts

The chart format is fairly straight forward to parse, but you need to understand how to interpret the values it gives you. Trying to derive the note translation formula was very taxing, and really hit my motivation to work on the project. However, after enough research and playing around, I was finally able to figure out how to properly map these notes according to the song. Hopefully others who wish to make similar games or software to work with the chart format can reference this article to help their endeavors go a bit smoother.

As of the writing of this article, I currently have the following (video - click to view):

[![Test Demo](https://img.youtube.com/vi/Fn8FmhVHEc4/0.jpg)](http://www.youtube.com/watch?v=Fn8FmhVHEc4)

There are a few bugs such as notes disappearing before they reach the bottom of the screen shown in the video that have been fixed post-recording. You may also notice that the notes don't look as pretty as they did in the last article - that's simply because I changed the note model and pushed off re-texturing until after I got the mechanics working.
