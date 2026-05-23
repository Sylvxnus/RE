Author: prestdayzero
Language: C/C++
Difficulty: 2
Platform Windows
Architecture: x86-64


Description
The aim of this challenge is to bypass some of the contraints in order to find a hidden admin panel that allows you to clear users debts

In this writeup, we are going to be going through the steps to reproduce, i am going to include screenshots so that we can see the situations clearly

Step 1
First its always good to try to understand what the characteristics of the binaries that we are downloading, for this we make use of the tool Detect It Easy (DIE)




We can see from the photo here that we are in 64 bit mode, written for the windows operating system. 
The program runs in the console and there is no packing hinted to, this is good, as it means that when we plug the binary into Ghidra, we won't just see gibberish



Step 2 
After running the .exe file through Ghidra, allowing it to scan the program fully, we can see that there is not a "main" function in the Symbol tree, this is ok though, because, Ghidra detected entry.




Much alike the last challenge writeup, when we analyse the decompiled version of the entry function, we can see that there are 2 function calls. 



Now, after the last challenge, we know that the first function is normally just boilerplate code, so I did have a look through it, and this theory was correct, so this means that we can move onto the following function call
For us it was the FUN_140001604();







This is what we saw when we went further down the rabbit hole of this function. Now, we know from the last challenge that we are looking for a few things here, the _get_initial_narrow_environment();, __p__argc, __p__argv

These 3 things indicate the creation of the main function, as they follow the same structure that we see when we write code in C/C++

We can see these three variables being fetched inside this function. Which means its likely that the next important function after these 3 fetches is likely the main functionality for this program.

In this case it was the FUN_140001000(); this is the function call that was assigned to uVar6;



Step 3 

After going in on this function, we can see the main control flow that is sent to the user by this program, this is a nice confirmation that we are in the right place.

My main thought process here was to look for anything that is out of place, anything that shouldn't be sent to the user, something that we might be able to bypass, looking for any conditional statements etc.

If we scroll down, we can see that there is a collection of conditional statements, one of the statements calls a function that displays a hidden admin access panel.
If we follow the code down further, we can then also see that there is control flow for what occurs on this hidden page, including clearing debts... this is what we want. 




If we look closer at the conditional statements, we want to see what kind of things are accepted for that hidden panel... if we highlight the code inside the decompiler, we can then see the assembly code and analyse it more closely.





See here, we compare 0xff to some value AL, (at the time i hadn't analysed what AL was) so i decided to convert 0xff into decimal, now 0xff is 255 in decimal, or -1 in 2's complement representation. So my thought process was to
attempt both of these numbers...

Test -1



So we get an error for -1 




Test 255





Success! Well whilst this is a good find, i wanted to understand more about why it had worked.





Analysis

Initially, the program checks to see if -1 is less than the local_18 variable, this means that the program blocks anything less than -1, but 255 still passes here


Now, this should be fine, as there is an else conditional statement that checks if the user entered value is != \x02, in otherwords if the value entered by the user isnt 2, then the choice is invalid!
So... Why does this fail then?

Well if we look at this one line above, this is the entire vulnerability. We are casting the local_18 variable to a character, instead of an integer

To understand this vulnerablity, we need to understand the sizes of ints and chars in bits.
- Integers = 32 bits
- Chars = 8 bits

Now, when we enter the number 255, its initially an integer, represented in bits as 00000000 00000000 00000000 11111111
If the program then compared them directly, using integer comparisons, 255 would then fail. 

But because we are casting to a char the first 3 blocks of 8 bits become truncated, meaning that we get 11111111 for our char(local_18)
Now, why is this bad?

because 11111111 is -1 in signed 8 bit binary. Meaning that by entering 255 as an integer, because the program casts it to a char, we are essentially entering -1, Allowing us to gain entry to the hidden admin panel

If char was also 32 bits, then this would't have been possible, purely because of the truncation here, we can REV the program to think we have entered -1 even though 255 was entered.

Giving us the final flag of: dzctf(bob_is_free_1337)
