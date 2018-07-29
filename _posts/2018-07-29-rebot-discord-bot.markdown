---
title:  "REBot - A Discord Helper Bot for Reverse Engineers"
date:   2018-07-29 05:21:00
categories: []
tags: []
---
A few months ago, I started realizing how much I was referring to assemblers and disassemblers for simple things, such as shellcode, gadgets, and other short code snippets. It also occured to me through other servers I was in, most notably the "Reverse Engineering" discord server, that sometimes terms and CVE's would get mentioned that weren't familiar to some other chat members. It was then that my idea for REBot was born - a bot that could assemble instructions, disassemble opcodes, and look up definitions and CVE's on the fly.

I also thought it would be neat to throw a few other neat commands in there such as commands to spit out a reversing or exploit development trick (though these are currently limited - contributions in this area would be most welcome). I had REBot spinning up within a week or so, however the source was pretty messy and inefficient. Since then, I've had time to refactor the code to make it less terrible, and the assemble/disassemble commands now directly use the keystone/capstone engine directly rather than using 3rd party wrappers. I decided to use Golang as my language of choice, partially because it was a newer language to me at the time that I wanted to get more familiar with, and partially because it seemed like a good language to suit the project.

As of today, I'm [open sourcing this bot to the public via GitHub](https://github.com/Cryptogenic/REBot). You're free to do what you like with it, but note that you will have to compile keystone and capstone, and set up the go bindings used by bot (or change them if you wish - though I don't really recommend this). You'll also need to configure the bot's Discord token via `config.ini`.

## What can it do?
As stated above, it can assemble and disassemble on the fly, but it has some other fun commands as well. Here's what REBot can do directly from REBot himself:

```
!assemble/asm [architecture] [instructions] - Assembles given instructions into opcodes. Instructions are seperated by a ';'.
!disassemble/disasm [architecture] [opcodes] - Disassembles given opcodes into instructions. Give in 'bb' format separated by a space.
!cve [cve identifier] - Displays information on a given CVE from NVD.
!info [identifier] - Gives information on the given word (like a dictionary).
!retrick - Gives you a random RE trick.
!expltrick = Gives you a random exploit dev trick.
!manual [architecture] - Links a PDF manual for the given architecture.
!commands/cmds - You are here.
```

## License? Do what you want!
If you'd like, you can fork it and use your own version of this bot if you'd like. If you think your edits / additions may be nice to add to REBot's main repo, then feel free to perform a Pull Request. Note however that whether or not these PR's are accepted are at my discretion. The project is licensed under the WTFPL (Do What The F\*ck You Want To Public License), so there's no conditions to modifying or using the project.

## Final Thoughts
I'm not sure how much more time I'll have to devote into this project, however if you find bugs or issues, go ahead and open an issue on the repo and I'll look into them. In terms of new features however - I'm not sure if I'll continue to actively develop it - but you're free to add your own features and send through a request to have them added to the main branch.