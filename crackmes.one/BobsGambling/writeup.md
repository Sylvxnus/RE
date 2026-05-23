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


<img width="617" height="351" alt="image" src="https://github.com/user-attachments/assets/7974b8fd-e287-4f18-9214-ee8053b9a532" />


We can see from the photo here that we are in 64 bit mode, written for the windows operating system. 
The program runs in the console and there is no packing hinted to, this is good, as it means that when we plug the binary into Ghidra, we won't just see gibberish



Step 2 
After running the .exe file through Ghidra, allowing it to scan the program fully, we can see that there is not a "main" function in the Symbol tree, this is ok though, because, Ghidra detected entry.

<img width="406" height="362" alt="image" src="https://github.com/user-attachments/assets/df3e10f9-4bd0-4359-ad16-0543e8e29eb2" />


Much alike the last challenge writeup, when we analyse the decompiled version of the entry function, we can see that there are 2 function calls. 


<img width="178" height="158" alt="image" src="https://github.com/user-attachments/assets/0c855d52-4992-44fe-af14-0ac1a37b8231" />

Now, after the last challenge, we know that the first function is normally just boilerplate code, so I did have a look through it, and this theory was correct, so this means that we can move onto the following function call
For us it was the FUN_140001604();

This is what we saw when we went further down the rabbit hole of this function. Now, we know from the last challenge that we are looking for a few things here, the _get_initial_narrow_environment();, __p__argc, __p__argv

These 3 things indicate the creation of the main function, as they follow the same structure that we see when we write code in C/C++

<img width="417" height="450" alt="image" src="https://github.com/user-attachments/assets/4b942f56-5650-4f54-8fe9-cd7a6179fc18" />

We can see these three variables being fetched inside this function. Which means its likely that the next important function after these 3 fetches is likely the main functionality for this program.

In this case it was the FUN_140001000(); this is the function call that was assigned to uVar6;



Step 3 

<img width="590" height="698" alt="image" src="https://github.com/user-attachments/assets/b7a4b7b9-4906-4fe0-ba30-42033c94ff86" />

After going in on this function, we can see the main control flow that is sent to the user by this program, this is a nice confirmation that we are in the right place.

My main thought process here was to look for anything that is out of place, anything that shouldn't be sent to the user, something that we might be able to bypass, looking for any conditional statements etc.

<img width="632" height="725" alt="image" src="https://github.com/user-attachments/assets/74cab14a-eb5d-4850-9fd1-de0e3446f4e5" />


If we scroll down, we can see that there is a collection of conditional statements, one of the statements calls a function that displays a hidden admin access panel.
If we follow the code down further, we can then also see that there is control flow for what occurs on this hidden page, including clearing debts... this is what we want. 

If we look closer at the conditional statements, we want to see what kind of things are accepted for that hidden panel... if we highlight the code inside the decompiler, we can then see the assembly code and analyse it more closely.


<img width="681" height="118" alt="image" src="https://github.com/user-attachments/assets/80682fa0-13ac-45e7-94be-434c3219b4db" />


See here, we compare 0xff to some value AL, (at the time i hadn't analysed what AL was, but that is the user entered value) so i decided to convert 0xff into decimal, now 0xff is 255 in decimal, or -1 in 2's complement representation. So my thought process was to
attempt both of these numbers...

Test -1

<img width="583" height="339" alt="image" src="https://github.com/user-attachments/assets/7f7f84bb-5dfb-4a6e-8f85-5d49f1e00f19" />


So we get an error for -1 
Test 255

<img width="587" height="346" alt="image" src="https://github.com/user-attachments/assets/72a98200-70ba-4d9d-a7ab-372c304603ac" />

<img width="248" height="204" alt="image" src="https://github.com/user-attachments/assets/62c3b160-f594-44c9-afa1-ee0f04c79bf8" />


Success! Well whilst this is a good find, i wanted to understand more about why it had worked.



Analysis

<img width="349" height="40" alt="image" src="https://github.com/user-attachments/assets/b51a0405-9012-415b-a292-be08ca9b5636" />

Initially, the program checks to see if -1 is less than the local_18 variable, this means that the program blocks anything less than -1, but 255 still passes here


<img width="469" height="158" alt="image" src="https://github.com/user-attachments/assets/ebcf30f2-6d8c-4385-9085-1757c3ee8429" />

Now, this should be fine, as there is an else conditional statement that checks if the user entered value is != \x02, in otherwords if the value entered by the user isnt 2, then the choice is invalid!
So... Why does this fail then?


<img width="226" height="32" alt="image" src="https://github.com/user-attachments/assets/c3252d9c-f0c3-47c0-8bb4-0ed937bb39b1" />

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
