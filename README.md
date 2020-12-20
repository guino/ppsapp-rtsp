#### This is a guide to patching ppsapp to enable the RTSP feature (and/or getting snap.cgi/mjpeg.cgi working).

Required tools:
* [Ghidra](https://ghidra-sre.org/)
* Hex editor, i.e. Windows: [HxD](https://mh-nexus.de/en/hxd/) or 'hexedit' in linux

I'm not going to go into details on installing/running ghidra (you can google that), but it should be pretty simple like unpack/run the start script.

Once you open ghidra, click File->New Project... select 'non-shared project', hit next and select a location to save the project (anywhere you like) and the project name like 'ppsapp' then click 'finish'.

Click on Tools->Run Tool->Code Browser -- a new window will open.
Click on File->Import File an select the ppsapp file you want to modify, leave default settings in the window and click 'ok' (if your language setting is blank select ARM v7 little endian).
Click 'YES' when asked to analyze it now.
On the list on the left select 'ARM aggressive instruction finder' and leave the rest as-is, click 'analyze'.

This may take a few minutes (depending on your machine), you should see a progress bar on the bottom right as it analyzes the file, when it's done there will no longer be a progress bar at the bottom.

At that point at the top of the middle window (assembly listing) you should see something like this:
![listing](https://raw.githubusercontent.com/guino/ppsapp-rtsp/main/img/segmentaddr.png)

(IF YOU DON'T WANT RTSP AND JUST WANT SNAP.CGI/MJPEG.CGI TO WORK YOU CAN SKIP TO THAT SECTION BELOW)

Write that value highlighted in green for later -- it usually is 0x10000.

Now let's search for the main function: press CTRL+SHIFT+E and enter **ipc_ring** in the box, then select 'all fields' at the bottom and hit 'next'.
(There's an alternate way to find the main sub but ghidra sometimes doesn't follow it right: on the initial page look for **_start** in the middle window and double click that, then double click on the LAB_0xxxxx in the __uClibc_main() function -- if you find the main sub this way you can skip to the part after the picture of the main sub)

The very first match should have something like this on the right side:

![ipc_ring](https://raw.githubusercontent.com/guino/ppsapp-rtsp/main/img/ringbuffer.png)

That is a function I have been calling initRingBuffer() -- not required: you can select the function name at the top (FUN_000f94dc in the example) and press 'l' (label) to change the name to initBuffer.

Right click oon the function name up top (FUN_000f94dc or initBuffer) and select 'References->Find references to...' and it will open a popup.
Click the first entry in the popup list, it should come to a function that looks like this (please notice I have renamed the functions for clarity) :

![initBuffers](https://raw.githubusercontent.com/guino/ppsapp-rtsp/main/img/initbuffers.png)

Again, right click on the function name at the top (initRingBuffers or FUN_xxxxxx) and select 'References->Find references to...', and the popup will open again (if not already opened).
There should only be one entry in the popup list: click it -- this will be the 'Main' function of the application and should look like this (notice I also renamed another function in it already):

![main](https://raw.githubusercontent.com/guino/ppsapp-rtsp/main/img/main.png)

You want to write-down the address of the main function (that 0009344c in the example) -- if you click the function name it should also highlight the adress on the middle window.

Now that we have the main function address, lets look for the 'init echoshow' function: click on the middle window (assembly listing) then press CTRL+HOME to go to the top of the listing and press CTRL+SHIFT+E to search. This time enter **init echoshow**, make sure 'all fields' is selected at the bottom and press 'next'.

It should come to a function that looks like this:

![initechoshow](https://raw.githubusercontent.com/guino/ppsapp-rtsp/main/img/initechoshow.png)

Write down the address of this function as we may need it later (init echoshow address: 000926f4)

The above may show up greyed out (as pictured) or in white background. But the layout of the code is the same. Greyed out usually means version 2.9.7 and newer where the function isn't being called, while white background (2.9.6 or older) already have a call to that function from the main sub.

The function highlighted in yellow is the 'tuya_ipc_echoshow' function (which actually starts RTSP support), double click on it and it should come to a function like this:

![ipcechoshow](https://raw.githubusercontent.com/guino/ppsapp-rtsp/main/img/hdcheck.png)

Versions 2.9.6 and older probably won't have the part highlighted in green while 2.9.7 and newer probably will -- the highlighted section is the part that prohibts HD RTSP from starting up.

Further down on that function you'll find this bit of code (all versions should have it):

![rtspstart](https://raw.githubusercontent.com/guino/ppsapp-rtsp/main/img/echoshowstart.png)

The green the part is what actually starts echoshow (RTSP) support and we'll want to modify it to always call the function regardless of the param_1 value.

Click on the **if** of that part in green, it should highlight a **beq** instruction on the middle window:

![patchrtsp](https://raw.githubusercontent.com/guino/ppsapp-rtsp/main/img/patchrtsp.png)

Write down that address as the RTSP patch address: 000e569c

Now lets make sure it gets called, right click on the **beq** and select 'Patch instruction' -- it will show a warning which you can just ignore, then it will show a red box around the **beq** -- you want to change the beq to be just **b** and press enter.

Notice that the code on the right side will no longer have the **if** portion around it, just the function call (so it start RTSP no matter what):

![patchedrtsp](https://raw.githubusercontent.com/guino/ppsapp-rtsp/main/img/patchedrtsp.png)

You'll also notice on the middle window that the 4 bytes in front of **b** (previously **beq**) now end with a **EA** instead of **0A** -- that's the first change we need to make that change in the ppsapp with the hex editor:

Copy ppsapp as ppsapp-rtsp and open it in the hex editor (HxD, hexedit, etc), you want to scroll/go to the address we saved for the RTSP patch deducting the load segment address (0x10000) so the address in the file will be 000e569c-10000 = de569c which is the position in the file you'll find the bytes for the beq command. DO CHECK the bytes before/after using the middle window listing to verify you're modifying the right location then change the **0A** to **EA** and save it.

If you're on 2.9.6 or earlier there's a good chance that you're done, but you can follow the below to verify it.
For 2.9.7 and newer I expect you will need to follow the below and make the extra changes.

Let's remove the HD check limitation at the start of the function, click on the **if** word of the highlighted section:

![hdcheck](https://raw.githubusercontent.com/guino/ppsapp-rtsp/main/img/hdcheck.png)

It should highlight a beq command on the middle window:

![hdpatch](https://raw.githubusercontent.com/guino/ppsapp-rtsp/main/img/hdpatch.png)

This time we want to make sure that **return** is NEVER called, so we want to change that **beq** to nop (nothing/no operation) -- so right click the **beq** and select 'patch instruction', then delete the contents on both red boxes and enter just **nop** where it used to be **beq** (and hit enter):

![hdpatched](https://raw.githubusercontent.com/guino/ppsapp-rtsp/main/img/hdpatched.png)

Now again we need to make the change in ppsapp-rtsp, so back to the hex editor this time on address e554c - 10000 = d554c
DO CHECK the bytes around it to make sure we're in the right place then make the change to the 4 bytes so that the **nop** replaces the old beq instruction, which in this case we change **9E 00 00 0A** to **00 F0 20 E3**.

Now the last patch we need to do is to actually call the initEchoShow function, so we'll need to go to the main sub address (we saved it before: 0009344c):

![main](https://raw.githubusercontent.com/guino/ppsapp-rtsp/main/img/main.png)

For each of the function in the main sub you want to check if any of them call the initEchoShow() function (000926f4) this should be obvious by looking at the function names -- if any of them are called FUN_000926f4 then you're done. 

If none of them are called FUN_000926f4, then we need to add a function call to it... the problem is that there's no way to easily add that call without recompiling the code so what we can do is **replace** one of the existing function calls (one that is useless to us) with the one that calls the initEchoshow function. In this case I have been using that function call that is just a 'log output' function:

![addcall1](https://raw.githubusercontent.com/guino/ppsapp-rtsp/main/img/addcall1.png)

Click on the function name to highlight it (like above) which should highlight the **bl** instrunction on the middie window:

![addcall2](https://raw.githubusercontent.com/guino/ppsapp-rtsp/main/img/addcall2.png)

We're going to replace the call for our initEchoShow function, so right click the **bl** and select "patch instruction" -- on the red box to the right change the address to be 0x000926f4 and press enter, it should look like thiis on the left/right:

![addedcall1](https://raw.githubusercontent.com/guino/ppsapp-rtsp/main/img/addedcall1.png)

![addedcall2](https://raw.githubusercontent.com/guino/ppsapp-rtsp/main/img/addedcall2.png)

There's no harm in leaving the parameters intact (they're just going to be ignored). So now we go back to the hex editor and patch the file at the address of that function call: 93528 - 10000 = 83528 (file offset)

Again CHECK the bytes before/after the address to verify they match and change the bytes to make them match the middle window in this case change **20 DC** for **71 FC** and save.

For all the versions I have seen so far the above should be all you need to get RTSP working.

#### IF YOU WANT TO GET SNAP.CGI AND MJPEG.CGI WORKING:

(THERE'S AN ALTERNATE WAY TO SEARCH FOR 00 08 02 00 : see https://github.com/guino/BazzDoorbell/issues/2#issuecomment-739664241)

Click the middle window and then CTRL+HOME to get back up to the top of the listing, then press CTRL+SHIFT+E (search) enter **pps_media_md_get_pic** make sure 'all fields' is selected and click 'next'. When it finds a match look at the right side, you're looking for a function that looks like the below (if it's not the same keep clicking 'next' until you find the one that looks like this):

![jpgaddr](https://raw.githubusercontent.com/guino/ppsapp-rtsp/main/img/jpgaddr.png)

The DAT_00551634 is where jpeg is stored and the address you can use in snap.cgi/mjpeg.cgi (i.e 0x0553164).

Alternate method I used originally (more involved for people knowledgeable in linux):
open a telnet, do a cat /proc/PID/maps (using the process ID of ppsapp listed on 'ps')
for each memory block marked with 'rw' search for JPEG signature JFIF, then search for the file start marker FF D8 -- there will likely be one in the first few memory blocks and another further down the list which should be for the live feed (used in snap/mjpeg.cgi)

#### IF YOU WANT TO GET PLAY.CGI WORKING:

To find the request address for play.cgi:
Click the middle window and then CTRL+HOME to get back up to the top of the listing, then press CTRL+SHIFT+E (search) enter **"talkTask"** (including quotes) in the search box, select 'all fields' at the bottom and hit search.
It should find one function that looks like the below:

![talkTask](https://raw.githubusercontent.com/guino/ppsapp-rtsp/main/img/talkTask.png)

You should double click the function that is highlighted above (below the while line) which will bring you to a function that looks like this:

![playaddr](https://raw.githubusercontent.com/guino/ppsapp-rtsp/main/img/playaddr.png)

The highlighted address (in the first **if** line) is the address to use in play.cgi (in this case 0x00477404).

#### ONLY IF YOU WANT TO PLAY WITH STREAMER-ARM
To find the address for streamer-arm:
Click the middle window and then CTRL+HOME to get back up to the top of the listing, then press CTRL+SHIFT+E (search) enter **ipc_ring_buffer** in the search box, select 'all fields' at the bottom and hit search.
when it finds the function, dismiss the search box, click on the decompiler (c code) window and press CTRL+F, enter **malloc** and press 'next', it should find something like this:
```
varx = Malloc(__n);
*(int *)(&DAT_0041a44c + vary) = varx;
```
That "DAT_0041a44c" is the address of the ring buffer for channel 0 (HD).
For channel 1 (SD) the address is that + 0x13c, so it would be 41a44c+13c=41a588 which is where the address for the ring buffer for channel 1 (SD) is stored.
For the buffer length it should be ```128*1536*6=1179648``` for HD (1080p) or ```128*512*6=393216``` for SD (the calculation is on that same function a few lines before the malloc but the parameters are set on a structure on a different function which you can find if you follow the code back from its references.







