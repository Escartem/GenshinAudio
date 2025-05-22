# this repo is not updated anymore, check [here for new content](https://github.com/umaichanuwu/GenshinLinks). You can still create issues or discussions post though to ask for something
---
# GenshinAudio (also see : [GenshinTextures](https://github.com/Escartem/GenshinTextures) | [GenshinCutscenes](https://github.com/Escartem/GenshinCutscenes))
All audio extracted from Genshin Impact, music, voicelines and everything else

Have fun listening them :)

If you have any questions or suggestions :
[![Discord Banner 2](https://discordapp.com/api/guilds/503554429648371712/widget.png?style=shield)](https://discord.gg/fzRdtVh) or [create a discussion](https://github.com/Escartem/GenshinAudio/discussions/new)

---
## How to download
You can :
* [Browse the files](https://github.com/Escartem/GenshinAudio/tree/master/GeneratedSoundBanks) and download the ones you need individually
* [Download the entire repo](https://github.com/Escartem/GenshinAudio/archive/refs/heads/master.zip) as a zip file (~18Gb)
* Clone the repo with `git clone https://github.com/Escartem/GenshinAudio` including .git folder (~34Gb)

---
## Some details

* all files are in English, I do not plan to add other languages unless being asked so if you want to see another one just [create a discussion](https://github.com/Escartem/GenshinAudio/discussions/new) and ask it here
* files are not named (see the technical stuff section to see why) you'll have to find manually if you're searching for a specific one. Here is a quick reference though:
  * `Minimum` -> login screen
  * `Music` -> music who would have guessed that
  * `Streamed and Banks` -> looks like generic sounds not related to english pack
  * In language folder there is `External` and `PCK`, `External` seems to have more sfx but I'm not sure
* due to github limitations, some folders don't display all the files when browsing in the website

---
## Technical stuff about extraction and why files are not named

If you want to help give files a name or understand about the extraction here are a few things:

In `.pck` files, containing the audio, there are mulitples markers to identify the files in it, one `wave` to indicate start of file with `RIFF` or `RIFX` 4 bytes before for endianness and then there should be a `list` marker with filename after it and then `data` with data of the audio file.

![i1](https://user-images.githubusercontent.com/43405304/201537848-ef9bc20d-1cec-4b44-a9c2-a0c645f6bb06.png)

However for most of the files the `list` marker is not here along with filename

![i2](https://user-images.githubusercontent.com/43405304/201538207-9bb88974-cf08-47e1-9a8c-8a6d0a7b314b.png)

I'm no unity expert but I guess the filenames might be mapped somewhere else and referenced through their offset.

If you are interested, here is the script I used with quickBMS :

```bash
for i = 1                        # run through loop with count variable i
   FindLoc OFFSET string "WAVE" 0 ""   # search for "WAVE", save position as variable OFFSET
   if OFFSET == ""                  # when nothing is found
      cleanexit                  # the script exits (e.g. at end of file)
   endif
   math OFFSET -= 8               # jump to possible
   goto OFFSET                     # RIFF/RIFX file start
   getDstring IDENT 4               # read string of 4 bytes, save variable as IDENT
   if IDENT == "RIFX"               # differentiate between header possibilities
      endian big                  # set endianness to big, if file has RIFX identifier
      callfunction write 1         # see function section below
   elif IDENT == "RIFF"            # endianness stays little
      callfunction write 1         # also run function
   else                        # string "WAVE" found, but doesn't belong to wave file
      set SIZE 0xc               # do as if something with 0xc bytes was found to continue search from the right position
   endif
   set SEARCH OFFSET               # set marker to position from where to search next
   math SEARCH += SIZE               # (that would be after the file that was found)
   if SEARCH == FSIZE               # in case the last found file ends with the main file, we exit
      cleanexit
   endif
   goto SEARCH                     # if we haven't exited the script above, we set out cursor to after the last found file
next i

startfunction write                  # function "write" starts here, is called when a wave file is found above
   get NAME basename               # save name without extension under variable NAME
   string NAME += "_"               # add underscore to the name
   string NAME += i               # add the loop variable to the name
   goto OFFSET                     # set cursor to the beginning of the found file
   get DUMMY long                  # RIFF/RIFX identifier, not needed
   get DUMMY long                  # riff size, not needed
   get DUMMY long                  # "WAVE", not needed, we arrive at the "fmt " section
   for                           # loop search for the "data" section at the start of the stream (get the stream size from there)
      getDstring AREA_NAME 4         # name of area in header
      get AREA_SIZE long            # size of area in header
      savepos MYOFF               # save position we are at
      if AREA_NAME == "data"         # when we arrive at the needed "data" area:
         break                  # we exit the loop
      else                     # otherwise:
         math MYOFF += AREA_SIZE      # not reached "data" area -> adjust cursor position...
         goto MYOFF               # ... and go there
      endif
   next                        # remember: the cursor is now directly at the stream start
   set STREAMSIZE AREA_SIZE         # the last AREA_SIZE is the size we need (size of the audio stream)
   set HEADERSIZE MYOFF            # 
   math HEADERSIZE -= OFFSET         # calculate the size of the stream header (offset - offset = size)
   set SIZE HEADERSIZE               # 
   math SIZE += STREAMSIZE            # calculate complete file size (header + stream = file)
   log MEMORY_FILE OFFSET SIZE         # write file to memory
   math SIZE -= 8                  # subtract 8 bytes to get the riff size
   putVarChr MEMORY_FILE 4 SIZE long    # write the correct riff size to the header inside the memory
   string NAME += ".wem"            # add extension to the name (the name could contain the name of the first marker if the file has markers)
   math SIZE += 8                  # add the subtracted 8 bytes again
   log NAME 0 SIZE MEMORY_FILE         # write file in memory to disk
endfunction                        # remember that we continue with our next i now!
```

there was also a `retrieveName` function that I removed because most of the filenames that were found (if there was some) were in chineese and vgmstream replaced those characters with `????` which the script didn't liked. But here was the function when `AREA_NAME` was equal to "LIST" :

```bash
startfunction retrievename            # get possible file name from first marker name, rmember: our cursor is after the size of the LIST area
   get DUMMY long                   # always "adtl", not needed
   get DUMMY long                  # always "labl", not needed
   get MSIZE long                  # size of the label area for this marker
   math MSIZE -= 4                  # subtracting 4 bytes leaves us with the length of the marker label
   get DUMMY long                  # marker type, not needed
   getDstring MNAME MSIZE            # cursor is at the beginning of the label name, now get the marker name with the desired length MSIZE
   string NAME += "~"
   string NAME += MNAME            # add the marker name to the file name
endfunction
```

If you have any ideas about that, any help is appreciated <3

---
## Credits

I do not own the music as they are the property of Genshin, but if you are using them it would be appreciated to credit this repo to help it gain popularity, thx <3
