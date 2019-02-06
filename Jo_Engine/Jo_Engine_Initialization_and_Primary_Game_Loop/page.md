# Jo Engine Initialization and Primary Game Loop

This example provides the bare essential code for running Jo Engine. For this you will need:
- Jo Engine
- Creating Disc Images in the Jo Engine Environment

```
/*
** Jo Sega Saturn Engine
** Copyright (c) 2012-2017, Johannes Fetz (johannesfetz@gmail.com)
** All rights reserved.
**
** Redistribution and use in source and binary forms, with or without
** modification, are permitted provided that the following conditions are met:
**     * Redistributions of source code must retain the above copyright
**       notice, this list of conditions and the following disclaimer.
**     * Redistributions in binary form must reproduce the above copyright
**       notice, this list of conditions and the following disclaimer in the
**       documentation and/or other materials provided with the distribution.
**     * Neither the name of the Johannes Fetz nor the
**       names of its contributors may be used to endorse or promote products
**       derived from this software without specific prior written permission.
**
** THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND
** ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED
** WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
** DISCLAIMED. IN NO EVENT SHALL Johannes Fetz BE LIABLE FOR ANY
** DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES
** (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES;
** LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND
** ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
** (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS
** SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
*/

#include <jo/jo.h>

//  Primary Logic Loop
void primary_loop(void)
{
	while(1)
	{
		/*
		YOUR CODE HERE
		*/
	}
}

void jo_main(void)
{
	//  Initialize engine with black background
	jo_core_init(JO_COLOR_Black);	

	//	Main loop of game
	primary_loop();	
}

```

The opening block comment giving the Copyright and Licensing agreement for Jo Engine should be retained in any file that makes calls to Jo Engine, if not for moral reasons, then at least for legal ones. This section describes how you are permitted to use Johannes Fetz's code, what liability he has on what you make using it, and restrictions on the distribution of said source code (and by extenstion, anything you make that requires it to run.)

```primary_loop()``` is a while loop where essentially all game logic will take place. It is initialized first so that the call to it in  ```jo_main()``` is possible.

```jo_main()``` is the nessecary main function that is run when the code is compiled and run on the Sega Saturn. ```jo_core_init()``` serves to initialize the game engine. The argument ```JO_COLOR_Black``` gives a default black background color before anything is drawn to the screen. Other colors may be selected from the [Jo Engine Documentation](https://www.jo-engine.orgdoxygen/files.html).

When ```primary_loop()``` is called, the ```while(1)``` condition starts and, provided the program does not crash, never stops. For normal game operation, you will never want to exit or break this loop, as the Saturn will simple stop doing anything. For an "exit game" or "return to title" feature, games states should be used (covered separately.)

This code should result in a black screen after startup.
