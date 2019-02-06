# Creating Disc Images in the Jo Engine Environment

This page demonstrates a set of working batch files for cleaning files resulting from compilation, compiling code and writing to disc images, and running the resulting disc images in an emulator. For this, you will need.
- [Jo Engine](../Jo_Engine.md)

Included in Jo Engine under directory ```\JO Engine\Compiler``` is a series of tools and a C compiler (via cygwn if on windows). These tools are utilized via scripts provided with example programs. Below are script files that will function when working in ```\JO Engine\Projects\[YOUR-CODE-FOLDER-NAME]```.

##  Windows

In windows, you will be using the batch script files ```clean.bat```, ```compile.bat```, and a third batch file corresponding to the emulator of your choice for play testing. Each emulator will require different calls to run the resulting disc image. Yabause is the simplest emulator to use for beginners, and so ```run_with_yabause.bat``` is demonstrated, however, it is not necessarily accureate and can sometimes have errors not present on hardware. Once you have grasped how to utilize these scripts, you may wish to use a more accurate emulator for testing (covered separately.)

**```clean.bat```**:
```
@ECHO Off
SET COMPILER_DIR=..\..\Compiler
SET PATH=%COMPILER_DIR%\SH_COFF\Other Utilities;%PATH%

rm -f ./cd/0.bin
rm -f *.o
rm -f %JO_ENGINE_SRC_DIR%/*.o
rm -f ./sl_coff.bin
rm -f ./sl_coff.coff
rm -f ./sl_coff.map
rm -f ./sl_coff.iso
rm -f ./sl_coff.cue

ECHO Done.

```

This file deletes object files and disc image files resulting from compliation. After your first compilation, you will want to run this before ```compile.bat```. The first line simply disables command prompt output so that the only prompted output is displayed. The proceeding two ```SET``` lines allow the script to call the program ```rm.exe``` in the Jo Engine compiler directory to remove files. The ```rm``` statements remove object (.o) files, .bin files, and other files used in or resulting from the disc image creation. Because ```sh-coff``` is the default name for resulting disc files, all disc images are assumed to have that name. If you wish to retain your disc image when cleaning, simply rename the necessary files (.iso, or the .bin/.cue pair.)

**```compile.bat```**:
```
@ECHO Off
SET COMPILER_DIR=..\..\Compiler
SET PATH=%COMPILER_DIR%\SH_COFF\Other Utilities;%COMPILER_DIR%\SH_COFF\sh-coff\bin;%COMPILER_DIR%\TOOLS;%PATH%
make re
JoEngineCueMaker

```

The first two lines function as ```clean.bat```. The final two lines use your make file and call ```JoEngineCueMaker.exe``` to generate disc images, all of which will be named sl_coff.

**```run_with_yabause.bat```**:
```
@ECHO Off
SET EMULATOR_DIR=..\..\Emulators

if exist sl_coff.iso (
"%EMULATOR_DIR%\yabause\yabause.exe" -a -i sl_coff.cue
) else (
echo Please compile first !
)

```

Again, superflouous output is supressed in the first line. The Jo Engine emulator directory is ```SET```. The if statement chacks for the existence of the compiled iso, and displays a request to compile first if it can't be found. Yabause is called with flags ```-a``` and ```-i``` to run the image ```sl_coff.cue```. These flags are needed to run the image properly, so leave them be.

For rapid prototyping, it may be prudent to create a single batch script that calls all three scripts in successsion.

[Back](../Jo_Engine.md)
