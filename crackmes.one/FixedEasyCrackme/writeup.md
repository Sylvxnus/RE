Crackme Author: Kanax01
Language C/C++
Platform: Windows
Difficulty: 1
Architecture: x86-64


Tools used: Detect It Easy (DIE), Ghidra (Developed by NSA)

Description of Challenge: 
This challenge is a beginner-level reverse engineering for Windows x86-64, the goal being to find the correct password that the program accepts.

In this writeup we are going to be going through the steps to reproduce and thought processes, including questions that i had along the way and answers found post research.



Step 1
Before we put the .exe file into my decompiler(at present, i use Ghidra) i wanted to gain some more information about the binary itself. To do this, we made use of a tool known as Detect it Easy (DIE)
This tool is a popular file type identification tool. It helped us to learn about the following things...

<img width="678" height="261" alt="Screenshot 2026-05-23 145152" src="https://github.com/user-attachments/assets/6e07dfca-6f4b-4788-a3d2-987e1f760d09" />
- PE64: This is a Portable Executable (PE) file format for executable code on 64 bit Windows operating systems.
- MSVC: This is the compiler that is being used, the Microsoft Visual C++ compiler
- Not packed: There is no indication from DIE that there is any packing programs being used, which is way more simple for us.
  We don't need to unpack anything.


Step 2
Now that we have scanned the file using DIE and have learnt about its file types and characteristics, we can safely plug the .exe into the decompiler, Using Ghidra, developed by the NSA.

When we give Ghidra the file, we first let the program analyse it. Ghidra analyses the binaries for you, it then also attempts to convert them into C/C++. This is very helpful as understanding C or C++ is much easier than understanding x86 Assembly.

<img width="267" height="268" alt="Screenshot 2026-05-23 172852" src="https://github.com/user-attachments/assets/9801b0bb-1f43-40ba-a900-402396069c29" />

After Ghidra has analysed the code, it comes back that it has found a function called "entry". 
This is interesting as this function could be somewhere that leads into the main part of the program, which is more than likely what we are going to be looking for if we want to find the password that the program is expecting or it will lead us to something that we can use to find this password.

We can click on the entry function inside the symbol tree inside Ghidra, this then brings up the decompiled version of this function. 

<img width="260" height="150" alt="Screenshot 2026-05-23 173007" src="https://github.com/user-attachments/assets/12873de8-0f58-4108-a21e-a0a6c972daef" />

Inside this function we can see that there are 2 more functions that are run from this, meaning that this is likely the function that runs the main functionality of this program.
- FUN_140002510();
- FUN_140001fac();

Now, we don't know what these functions do at this stage, so the next part is to double click on these functions to see what they do...


Step 3
The first function (FUN_140002510()) is a random number seed initialiser. its MSVCs stack canary. This is a buffer overflow protection mechanism. 

<img width="510" height="581" alt="Screenshot 2026-05-23 173054" src="https://github.com/user-attachments/assets/d77e4d41-0a0c-4c7e-8c67-016c7891c4f8" />

Essentially, this is a security cookie that needs to be randomised to make it unpredictable to an attacker. 

First it checks if the security code has its default value, which is always 0x2b992ddfa232
It the default is still present, it then collects several sources of randomness and XORs them together
- Current time
- Current thread ID
- Current process ID
- Query Performance counter
- (ulonglong)local_18 = the stack address

Xoring all of these values together will mean that there will be a different security code every single time the program is ran.
Whilst, this is very interesting and taught me quite a bit about memory buffer overflow protections are implemented, it is not what we are looking for in our challenge, the security code config indicates that this is probably boilerplate and we can look for other things that may be more of an interest to us. so lets move onto our next function

The second function (FUN_140001fac()) was the second called function within the entry function.
<img width="488" height="701" alt="Screenshot 2026-05-23 173141" src="https://github.com/user-attachments/assets/ce0767b1-cb34-49d6-9ac1-a641f154c34f" />

Having a look through this function, we can see that there is quite a lot of boilerplate configuration such as _scrt_acquire_startup_lock(); which is for threading set-up

But something interesting is after we have fetched our __p__argc, __p__argv and _get_initial_narrow_environment();, 
We see the following...


Now... Three arguments, called right after getting our argc/argv/evnp, this could be the signature of the main function here.
A good tip to remember is that in any MSVC binary, find where argc and argv get fetched, the very next significant function is probably the main. In this case, for me it was the FUN_1400012a0();
So we now want to know what this function does as it looks interesting, the next step is to click on it on the decompiler and see if we can go in on it and find out what it does.


Step 4

<img width="485" height="709" alt="Screenshot 2026-05-23 173321" src="https://github.com/user-attachments/assets/16fe0e70-6f4c-42a1-81a3-0c0149ddd990" />

After clicking on this function inside Ghidra, we can clearly see that if we are to scroll down we can see the following.
pcVAR6 = s_EasyPassword_1400050b8

Well, this is very interesting, we can also see in the photo above that this is what we are looking for. There is a message that is printed out after that password varaible is entered saying that we cracked the code. 

We can also see that there is memcmp method. This is really interesting, as this is a giveaway that it is comparing 2 blocks of memory. The program also does a length check to ensure that the password is a certain length, we can click on the condition inside Ghidra to see the exact length that is required.

<img width="804" height="114" alt="Screenshot 2026-05-23 172903" src="https://github.com/user-attachments/assets/8a84d350-be18-4b5a-9ee8-4283d8158866" />

We can see in the photo that the program is looking for a password that has exactly 12 characters, this is because C is 12 in hexadecimal notation. 
If we ever see memcmp, strcmp, strncmp, we know that we have more than likely found some sort of check. one argument is always the user input and the other is always the value that the program is looking for.

But this is just the variable name, we need to know the value that it is set to in the code itself, during the static analysis. So once again, inside Ghidra, we click on the variable name and see if we can go in on it to learn more.


<img width="831" height="145" alt="Screenshot 2026-05-23 150834" src="https://github.com/user-attachments/assets/d1212196-11d4-4f64-a9c6-8010d3ac23bf" />

And there we go, we can see that the value for this is shown clear as day in the Assembly code, this now means that we have potentially found the code that could unlock the program and help us REV this program.

Another interesting find, is that Ghidra already identified it as a string in the decompiler, but in the assembly, we can see that there is a small tag ds.
ds means defined string meaning that Ghidra recognised these bytes for this variable as printable ASCII

Step 5
So now that we know the password, we can run the program, inside the command line, we know that it runs in the command line, because DIE actually told us in the first line of its analysis that it runs on the console. Which is very helpful!

<img width="322" height="118" alt="Screenshot 2026-05-23 173530" src="https://github.com/user-attachments/assets/37fcfb7a-3f26-485f-ae8d-66d24065868b" />



So we run the program and entered the password....and we cracked it!



Outro:
This was my first attempt at any sort of RE, I have learnt MIPS this year at university, but hadn't gone any further than this, I am going to be doing more writeups for these sorts of challenges, expanding my knowledge base into different assembly/low level languages.

