# Playing A Sound Effect

(Work in Progress) The code in this example was provided by **Ponut64** with structural code written by **XL2**.

The make file will need to tell Jo Engine to include SGL, Audio, and Dual CPU support.

**```makefile```:**
```
JO_COMPILE_USING_SGL = 1
JO_COMPILE_WITH_AUDIO_MODULE = 1
JO_COMPILE_WITH_DUAL_CPU = 1
...
```

The sound effects are .PCM files that are either 15.360 kHz or 30.720 kHz in bitrate. These bitrates are selected to come out to 1024 and 2048 bytes of sound information per frame, respectively. CD sectors are generally expected to be 2048 in size, so a fixed bitrate of either of these two values is recommended for a clean division of sound information in memory. The sound files are available below, and must be stored in the ```/CD/``` subdirectory of your project folder:
- [BSTEP.PCM](BSTEP.PCM) (15.36 kHz)
- [BTN1.PCM](BTN1.PCM) (30.72 kHz)
- [J32.PCM](J32.PCM) (30.72 kHz)
- [JAM.PCM](JAM.PCM) (15.36 kHz)
- [LSTEP.PCM](LSTEP.PCM) (30.72 kHz)
- [RIFL.PCM](RIFL.PCM) (15.36 kHz)
- [RUMB.PCM](RUMB.PCM) (15.36 kHz)

The code consists of a single file with thre functions handling sound loading and processing, and a remaining five functions that handle file system loading and the game loop.

**```main.c```:**
```
/*
Demo by Ponut64
THIS SOFTWARE, ASIDE FROM THE ASSOCIATED LIBRARIES AND CONTRIBUTIONS USED TO CREATE IT, IS PUBLIC DOMAIN.
Uses Jo Engine, copyright Johannes Fetz
**Warning: This demo is WAY more complex than you explicitly "need".
**It's also just a little bit broken! Lovely! 
**However! I show you an interruptible or asynchronous loading process. If you need things to work more simply or in a less specific way, see Jo's audio demo.
**I also show how to use slSoundRequest for PCM sound.
*/

/**
DEMO DESCRIPTION:
Demonstrates loading a PCM file from CD asynchronously and having you push a button to play it back.
Directory changing is not demonstrated. Directory changes insert a mandatory system halt so can't be done asynchronously. Dir change commands can be found in XL2's demos.
Sound file conversion help:
16-bit AIFF files will work, but you have to cut the header off or skip it somehow (or else it'll play a bit of garbage). You can use SoX to convert to this format.
"Raw" PCM format files made with FFMPEG are ideal. Here's a batch command line sample that was used for a file in this demo:

ffmpeg -i rumbling.wav -f s16be -ac 1 -ar 15360 RUMB.PCM
PAUSE

You can find ffmpeg at https://www.ffmpeg.org/ .
**/


#include <jo/jo.h>
//SNDRAM is the location of sound memory in the SH2's memory map. Your program is sent to the SH2's so it uses their memory map. 
#define SNDRAM  (631242752)
//There is a region of SNDRAM that the M68K [Sound CPU] and/or SCSP [Sound processor] have reserved for certain hardware functions. 
//At this far in sound memory, its OK to place whatever you want.
#define PCMBUF1 (SNDRAM + 40960)
//This is a macro that re-maps an SH2 memory address in sound RAM to a memory address for the M68K [sound CPU] as its memory map is confined to sound RAM [512K].
#define MAP_TO_SCSP(sh2map_snd_adr) ((sh2map_snd_adr - SNDRAM)>>4)
//These are pitch words for playing back PCM audio. While what I put here are integers, SGL is reading them as WORD data (16-bit). It would be expressed as 0x7992. Doesn't matter!
//The pitch words represent the bitrate.
///In this folder, you can find a file called "pwordprog.c". Compile this on the internet using GCC or otherwise to find your pitch word. Just change the bitrate line.
#define M3072KHZ	(31122)
#define S1536KHZ	(29074)
//The following is GFS data.
//Sector size is the CD sector size. It can be 2048 or 2352. Jo Engine by default compiles at 2048. It's a good idea to manage memory in sectors (separate files by at least 2KB in memory).
#define     SECT_SIZE   (2048)
//RD_UNIT is the number of sectors read per loop. A safe assumption of Saturn CD bandwidth is 300 KB/s. At 30 FPS, that's 300 / 30 = 10 KB, divide by 2 KB, get 5 sectors. 
//However, here is an oddity that you would not mathematically expect: The CD system appears to be able to read 10 sectors per frame at 30 FPS. It's probably due to it expecting 60 Hz / 50 Hz ops.
//You can expect as much from real hardware! However, it is over the specification and the quality of the burn and your Saturn's laser may make reading at that rate inconsistent.
//So, we settle for the RD_UNIT of 8.
///RD_UNIT is also used for PCM timing in this case. You don't have to do the same.
///^< Explanation: For 15360 bitrates, 15360 bits * 16 bit PCM = 245760 raw bitrate / 8 bits = 30720 bytes/s / 30 = 1024 bytes per frame.
///^< Explanation: For 30720 bitrates, 30720 bits * 16 bit PCM = 491520 raw bitrate / 8 bits = 61440 bytes/s / 30 = 2048 bytes per frame.
#define     RD_UNIT     (8)
//RD_STEP is the same number above, just expressed as bytes rather than sectors.
#define		RD_STEP		(RD_UNIT * SECT_SIZE)

/**
PCM DATA STRUCTURE
Top lines: GFS Information
file_done : if there is actually data in this pcm data.
active : if this is actively being filled with data.
dstAddress : where the data is going to go.
fid : file ID. Use GFS_NameToId((Sint8*)name) on your file. Changing folders and such is possible but not covered here.

pitchword : The bitrate, converted into a pitch word for the sound CPU.
playsize : the size of data to be played back. NOTE: There are issues with play-sizes that run over 255 frames. Limitation of sound CPU? Dunno.
loctbl : [Archaic] Represents the order in your PCM buffer segments. A suggestion, that even I may not follow.
segments : the number of PCM buffer segments the file consumes. [Big files don't play that well..]
playtimer : an active timer of how long this PCM sound effect has been played.
frames : the number of frames this sound needs to play. The math is strange. The base factor is 1 frame per 16 KB, as derived from the reading process [8 sectors].
^< Explanation: For 15360 bitrates, 15360 bits * 16 bit PCM = 245760 raw bitrate / 8 bits = 30720 bytes/s / 30 = 1024 bytes per frame.
^< Explanation: For 30720 bitrates, 30720 bits * 16 bit PCM = 491520 raw bitrate / 8 bits = 61440 bytes/s / 30 = 2048 bytes per frame.
For 15.360KHz playback, 1KB is played per frame. For 30.720KHz playback, 2KB is played per frame.
**/
typedef struct{
	bool	file_done;
	bool	active;
	int	dstAddress;
	Sint8*	fid;
	
	int	pitchword;
	int	playsize;
	int	loctbl;
	int	segments;
	int	playtimer;
	int	frames;
} pcmdat;

//Definition of some pcmdata.
static pcmdat pcm_slot[1];
//Definition of a GFS Handle. GFS Handles are required to use SBL file system functions. We use SBL file system functions as they are the fastest. They can also be re-used.
static GfsHn fileHandle;
//rd_frames : the desired number of frames to read the file.
int	rd_frames = 0;
//curRdFrame : the current number of frames read into the file.
int curRdFrame = 0;
//what it says. [sector offset]
int point_in_file_to_seek = 0;
//Control information for the 8 PCM channels.
bool			ch_on[8];
Uint8			CH_SND_NUM[8];
bool			channel_ready[8];
//SGL data for user-defined synch.
Sint32 framerate;
Sint8 SynchConst;

///From XL2
void	update_gamespeed()
{
    static int curtime;
    curtime = jo_get_ticks();
    static int lasttime=0;
    int frmrt = (curtime-lasttime);
    framerate = (frmrt)>>4;
    lasttime = curtime;
	
    if (framerate <= 0) framerate=1;
    else if (framerate > 5) framerate=5;
}

///This goes at interrupt. Control is performed by ch_on boolean.
//Notice: slSoundRequest is broken into multiple parameters by newlines (comma).
/**
So here's a massive, messy wall of code. I will do my best to explain it.
Input sound_number: the a pointer to the sound designated to the specified channel. This is not controlled here! 
Notify an array of sound number channels as to which channel gets which sound effect. I hope you understand. It's a bit complicated.
Input channel: The PCM channel the sound effect will play on. There should be 8 available for mono playback. We assume CH0 is eaten for music.
if the channel is ready (not busy)...
if we turn the channel on ...
if the sound has just started playing... Request a PCM sound effect!
--For more information, in SBL6 source code, go to HOST.ASM for SDD.
Data format: "byte, byte, word, word, word, byte, byte"
Command macro SND_PCM_START (an SBL macro that for a 0x00-sized value sent to the command registers of the M68K [sound CPU])
channel : the channel. Open up Windows 10 calculator and switch its mode to "Programmer". Go to the bit graph and read from below.
Interestingly, if you have 16-bit mono sound, it's just the channel number as an integer.
Explanation from sound driver source code:
; P1 :   D7      = L ( mono ) / H ( stereo )
;        D6      = no care
;        D5      = no care
;        D4      = L ( 16bit PCM ) / H ( 8bit PCM )
;        D3      = no care
;        D2～D0  = PCM Stream#
ex. channel 2, mono, 8 bit pcm would be integer value 146.
224 : Pan & volume. Explanation from sound driver source code:
; P2   : D7～D5  = Direct level [DISDL]					*
;        D4～D0  = Direct Pan   [DIPAN]					*
ex. max volume, no pan: 224
ex. half volume, no pan: 128
ex. max volume, right pan: 239
ex. max volume, left pan: 255
ex. max volume, mid-right pan: 231
ex. max volume, mid-left pan: 247
MAP_TO_SCSP: A macro that maps an SH2 address to the sound CPU's internal memory address range. This is the actual location of your sound data in sound RAM.
playsize: the size of the data to play back. (time before loop)
Run the playsize lesser or exactly the size of the file! If it runs over, you get garbage out of the sound system and that's bad! If it's lesser, it will loop a little bit.
pitchword: bitrate.
0: Effect. [Not studied]
0: Effect level.
NOTICE: Parameters 3 (Address), 4 (size), 5 (pitchword), and 6 (effect) are duplicated in the case of stereo playback (Right ch first, then left ch)

playtimer: Because this runs at VBLANK, it is hit and added to 2 times per frame (for 30 fps of course).
Dividing the playsize by half of its data throughput per frame gets double a number double the exact frames the file size will run.
Because playtimer gets two added to it every frame (because we're at vblank), it works out as two numbers that can be compared as..
when equal, we are done with the sound file.

This is assuming the sound file is small enough for the sound CPU to play continuously. It is not possible for the sound CPU to playback PCM sound the length of its memory.
I'm assuming the assembly driver has an 8-bit timer to the length of a sound effect.
If you want to play a longer sound continuously, you have to offset your start address and re-send the command to playback the sound on that channel at the right time.

WARNING: Emulators perceive the pitch word differently between one another. Bizhawk is the most accurate to real hardware, as far as I know. Bizhawk has a common code-base with Mednafen.
If a sound effect doesn't quite play for the right time in an emulator, well.. the hard test of hardware is left to settle if it's actually a bug of yours, or of theirs.
You could make better timing code.
**/
void sound_on_channel(Uint8 sound_number, Uint8 channel){
	static bool ready_play = false;
	static int offset = 0;
	static int SndCnst = 200;
		if(channel_ready[channel] == true){
	if(ch_on[channel] == true){
	if(pcm_slot[sound_number].playtimer < 1) ready_play = true;
	if(ready_play == true){
		slSoundRequest("bbwwwbb",
		SND_PCM_START,
		channel,
		224,
		MAP_TO_SCSP(pcm_slot[sound_number].dstAddress + offset),
		0,
		(pcm_slot[sound_number].pitchword),
		0, 0);
		ready_play = false;
	}
	pcm_slot[sound_number].playtimer ++;
	if(pcm_slot[sound_number].playtimer == SndCnst || pcm_slot[sound_number].playtimer == (2 * SndCnst) || pcm_slot[sound_number].playtimer == (3 * SndCnst) || pcm_slot[sound_number].playtimer == (4 * SndCnst) || pcm_slot[sound_number].playtimer == (5 * SndCnst)){
		ready_play = true;
	}
	jo_printf(18, 21, "(%i)", ready_play);
	if(pcm_slot[sound_number].pitchword == S1536KHZ){
	offset = pcm_slot[sound_number].playtimer * 512;
	if(pcm_slot[sound_number].playtimer >= ( (pcm_slot[sound_number].playsize / 512) ) ){ch_on[channel] = false;}
	} else if(pcm_slot[sound_number].pitchword == M3072KHZ){
	offset = pcm_slot[sound_number].playtimer * 1024;
	if(pcm_slot[sound_number].playtimer >= ( (pcm_slot[sound_number].playsize / 1024) ) ){ch_on[channel] = false;}
	}
}
//sound complete, close channel
	if(ch_on[channel] != true){
		slSoundRequest("b", SND_PCM_STOP, channel);
		offset = 0;
		ready_play = false;
		pcm_slot[sound_number].playtimer = 0;
	}
	jo_printf(18, 17, "(PCM SFX)");
	///Notice: This data can get loose between channels, so it's a good idea to print it off-screen if you don't want to see it, rather than comment it out.
	jo_printf(18, 18, "(%i) checked ch", ch_on[channel]);
	jo_printf(18, 19, "(%i) pcm time", pcm_slot[sound_number].playtimer); 
	jo_printf(18, 20, "(%i) target time", pcm_slot[sound_number].frames);
	// jo_printf(18, 21, "(%i)", offset);
	}
}

//Triggers a sound...
//At the desired channel (member of an array)...
//With the desired sound effect (member of an array).
void	trigger_sound(Uint8 channel, Uint8 sound_number){
			channel_ready[channel] = true;
			CH_SND_NUM[channel] = sound_number;
			ch_on[channel] = true;
}

void	pop_load_pcm(void(*game_code)(void)){
	Sint32	file_size = 0;
	Sint32	sectors = 0;
	Sint32	cur_loop_read_amount = 0;
	Sint32	gfs_stat_data = 0;
	fileHandle = GFS_Open((Sint8*)pcm_slot[0].fid);
	//What's that NULL data? Stuff we don't need!
	GFS_GetFileSize(fileHandle, NULL, &sectors, NULL);
	GFS_GetFileInfo(fileHandle, NULL, NULL, &file_size, NULL);
///How many frames are we reading?
	if(RD_STEP < file_size){
		rd_frames = (file_size + (RD_STEP - 1))/(RD_STEP);
	} else {
		rd_frames = 1;
	}
///Seek to the desired spot on the file
	GFS_Seek(fileHandle, point_in_file_to_seek, GFS_SEEK_SET);
///Set read & transfer parameters
	GFS_SetReadPara(fileHandle, RD_STEP);
	GFS_SetTransPara(fileHandle, RD_UNIT);
///Transfer mode should be SCU because this is going to sound RAM.
///If it were going to LWRAM, it should be CPU.
///If it were anywhere else, you could make it CPU or SCU.
///Just do not set GFS_TMODE_SDMA0 or SDMA1. It is likely to crash on real hardware.
	GFS_SetTmode(fileHandle, GFS_TMODE_SCU);
///"New CD Read" --> This commands the SH1 to command the CD to start pre-reading data to the CD block buffer.
	GFS_NwCdRead(fileHandle, file_size);
///Only read when we've read less than we want to.
	for( ; curRdFrame < rd_frames ; ){
	///Because we've already started the CD read, "NwFread" becomes a FETCH command rather than a READ command.
	///NwFread can be either a FETCH or a READ command depending on whether or not the file is being pre-read to the CD block buffer.
	/**
	GFS_NwFread
	Params:
	filehandle: GFS handle of file.
	RD_UNIT: The number of sectors to read per execution.
	dstAddress + curRdFrame * RD_STEP: Every execution (loop), we want to change where we put this new data so we don't overwrite the data we previously read.
	For continuous reads (by GFS_Load), that's not necessary. Here, it is necessary, as each time GFS_NwFread is hit, it starts a new fetch to the address.
	We offset the address by the current frames read into the file multiplied by the size of each read loop, RD_STEP.
	RD_STEP: The amount each loop/execution of NwFread will fetch/read.
	**/
	/**
	game_code -> Your game.
	slSynch -> Enforces synch constant [frame time limit].  Otherwise it will run as fast as it can. You actually don't want that!
	GFS_NwExecOne: Execute the file system commands (both NwCdread and NwFread start here).
	GFS_NwGetStat: Get the amount we've read in this loop so far, and get the status of the GFS. If the status is ever 2, you have a problem. [Emulators frequently ignore said problems]
	**/
		GFS_NwFread(fileHandle, RD_UNIT, (Sint32*)(pcm_slot[0].dstAddress + (curRdFrame * RD_STEP)), RD_STEP);
		//Move the seek pointer forward each loop by the amount read each loop.
		point_in_file_to_seek += RD_UNIT;
		do{
			game_code();
			slSynch();
			GFS_NwExecOne(fileHandle);
			GFS_NwGetStat(fileHandle, &gfs_stat_data, &cur_loop_read_amount);
			
	jo_printf(0, 15, "(%i) cur frame read", curRdFrame);
	jo_printf(0, 16, "(%i) total frames to read", rd_frames);
	jo_printf(0, 17, "(%i) sect", sectors);
	jo_printf(0, 18, "(%i) rdsize", cur_loop_read_amount);
	jo_printf(0, 19, "(%i)fzise", file_size);
	jo_printf(0, 20, "(17) loop label");
		jo_printf(0, 7, "(%i) seek pt", point_in_file_to_seek);
		jo_printf(0, 10, "(%i) fs stat", gfs_stat_data);
			
		}while(gfs_stat_data != GFS_SVR_COMPLETED && cur_loop_read_amount < RD_STEP);
		curRdFrame++;
	///FOR-DO-WHILE READ LOOP END STUB
	}
	GFS_Close(fileHandle);
	//If we've read as much or more than we need to...
		if(curRdFrame >= rd_frames){
			if(pcm_slot[0].file_done != true){
///How many buffers is the sound going to consume?
///What is the remainder in the last or only buffer consumed?
			if(file_size > 16384){
				pcm_slot[0].segments	= (file_size + (16384 - 1))/(16384);
			} else {
				pcm_slot[0].segments 	= 1;
			}
			pcm_slot[0].playsize = file_size;
			pcm_slot[0].file_done = true;
			//You don't specifically need to do this. I do, because it matches up with the bitrates I expect to use (15360 and 30720) and my read speed (8 sectors / 16 kb).
			pcm_slot[0].frames = rd_frames;
			}
		rd_frames = 0;
		curRdFrame = 0;
		///SFX HANDLER END STUB
		}
///SOUND LOAD REQUEST END STUB

//This shows how you keep running your game after the file is done reading. To re-start reading after setting up what you want to read, insert a BREAK.
	do{
	slSynch();
	game_code();
	}while(pcm_slot[0].file_done == true);
}

void			my_vblank(void){
	sound_on_channel(CH_SND_NUM[1], 1);
}

//This number represents something going on in your game. It won't stop whether the file is or isn't reading.
FIXED cyclicNumber = 0;
void			my_draw(void)
{
	cyclicNumber+=50;
	slPrintFX(cyclicNumber, slLocate(0, 8));
	if(jo_is_input_key_pressed(0, JO_KEY_Y)){
		trigger_sound(1, 0);
	};
}

void			master_file_system(void){
//Index of included files:
/**
BSTEP.PCM -> 15.36 sample.
BTN1.PCM -> 30.72 sample.
J32.PCM -> 30.72 sample.
JAM.PCM -> 15.36 samle.
LSTEP.PCM -> 30.72 sample.
RIFL.PCM -> 15.36 sample.
RUMB.PCM -> 15.36 sample, large size.
**/
	pcm_slot[0].fid = GFS_NameToId((Sint8*)"J32.PCM");
	pcm_slot[0].dstAddress = PCMBUF1;
	///pcm_slot[0].pitchword = S1536KHZ;
	pcm_slot[0].pitchword = M3072KHZ;
	pop_load_pcm(my_draw);
}

void			jo_main(void){
	
	jo_core_init(JO_COLOR_Black);
	//Synch data [XL2]
	slDynamicFrame(OFF); 
    SynchConst=(Sint8)2;
    framerate=2;
	
	//Register our function at SGL's one allowed interrupt.
	slIntFunction(my_vblank);
	jo_core_add_callback(master_file_system);
	jo_core_run();
}

/*
** END OF FILE
*/

```

[Back](../Jo_Engine.md)
