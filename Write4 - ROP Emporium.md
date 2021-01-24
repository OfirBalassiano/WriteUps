# Write4 - ROP Emporium


We’re going to walk through a challenge from ROP Emporium called “Write4”.  
ROP Emporium contains a series of CTF challenges related to ROP ([Return Oriented Programming](https://en.wikipedia.org/wiki/Return-oriented_programming)).  
  

Ok, Let’s start!.

  

We received three files for this challenge: `"write432"`, `"libwrite432.so"`, and `"flag.txt"`.

![](https://lh6.googleusercontent.com/mlmB6N65RaAo0T5wteZRe-mWeik6JLAKodVoKcpAxSOsBZJIbMYLKHC0xMWgxzBP3a-mC_YRKwj9sMD3-bUsRHbnz63xDx5BlfWzLiZQj0RzmaMCQKfCOoltMhAv5YYFZAJ6PaaS)  
  
According to the instructions in the challenge, we need to use the `print_file()` function to display the contents of `flag.txt`. This function takes as an argument the name of the file and displays its contents

We will run write432:

  

![](https://lh5.googleusercontent.com/WyZAy_d2knpwxY4y_B_RXB3d3xZ3-22cqjYEIkwmE71Z4xKE1No87l6mYVtg7BSYN1-O8GHH1vVo5xZ0reT0U0cX3NdgOUdDmy4pYd3K9rourxTZitCbjJoG8ux0vIbLeFHrYRZG)  
The program requests some input from the user, and exits immediately afterwards.

  
  

We will go into more detail about the functionality of the files, and find out how we can run `print_file()` and read the desired files-write432:

  

we will use Radare see which functions exists

  

![](https://lh6.googleusercontent.com/H-MpQZdJq9_qp2Rmqe95uk26rsT6Oz1OCgsb5OjjqpezhXW-0LWrlaCCTZgfzLRorSBnxCR8KFQKd8Xln6wIinjCTtV0V1MsVt1GyrFdKl0she5ISjcFo9IzEyuRAenctMsE6TH2)

First thing that stands out: There are two imported functions: `pwnme()` and `usefulFunction()`.

Let's disassemble the usefulFunction function:![](https://lh5.googleusercontent.com/S6TBRO6FEravhEzsYOILyNOjox7sNJsJyKC1UKCJKmAfSKlthetuULwx5SdWal2g7RUrg3v4Hp24NQ_JlGXb4XV8kwABXqbdThfKej90to0Ccoh1gBm_RhydC78jlSU_56tbIYR7)

It can be seen that it calls the `print_file()` function with some string as an argument.

In order to read the pwnme function, we need to access the libwrite432.so file:

If you are unfamiliar with the Procedure Linkage Table (PLT), I highly recommend that you [read about it now](https://www.technovelty.org/linux/plt-and-got-the-key-to-code-sharing-and-dynamic-libraries.html).

  

Next, we will disassemble pwnme:

![](https://lh6.googleusercontent.com/8ukTdgIK1OmlAEk6NQo9mmdbmzlSn13oeqSKwUcvSJXP63YIL9ZrLI0lJr50HT4V9vnKxLlBIu_W1cqccUXJ31XCE8hcApNh5-ubeU41v2o0MpEJyx174zfB1dviC4YqLTu-MT9a)

Here are a number of things going on. Firstly, we can see that the program is printing out information. Secondly, there is a buffer of size `0x20` (32) bytes, that is getting zeroed out with memset.

Eventually, we have a `_read()` function that will read up to `512` bytes from the input into the buffer.

  

In this situation, a typical buffer overflow scenario occurs so we will use gdb to debug the binary while it is running.

  

We will use pwntools to better understand how we can control the `EIP`.

In order to trigger the buffer overflow, I created a 100 byte string, which we will insert as input and use to trigger the `EIP`

	python2 -c 'from pwn import *; print(cyclic(100))'
	aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaa


![](https://lh5.googleusercontent.com/0RrjCR4ijMDGRsWTTJxe2wL-53O1dFU80KejYrk26o-HwKBJBaeJhcT0YX-XYw2rmweYOehZRPU5nrFXPXs16eUQ21AKy1BQYV72B641R-hc6_VhVzI9u84_TKrYwuDXSHhN5ehW)  
As we expected the program crashed, and it can be seen that the value that appears in the `EIP` is `0x6161616c` ('laaa') which means that for our payload, we will need a padding of about 44 bytes.

![](https://docs.google.com/drawings/u/0/d/sLbftQy43nY9v8WX1RPSyMQ/image?w=561&h=69&rev=11&ac=1&parent=1w1tKXv6q-WxivHxSLVGf6zbY3pOX8Rl4OrYVNiP8MRw)

  

We will check if we're right and attempt to override the `EIP` to get the program to run `print_file()`:

	python2 -c 'from pwn import *; print("A"*44 +p32(0x8048538) )' > check



![](https://lh4.googleusercontent.com/5lkf64PF38ZaTcZ-XUyOc90ap30ThKxGgJiurAWIg_A7Es5FHTDROPmxNUjfJNx5onit81S-h9hT61-p7FwM9JfpujJ1VjynjteC8GfURocN4h67LpMW-kYrXQPriAClj5EzRD7x)

Ah! We were able to run the function we wanted! Now let's see what the stack looks like:  
I will put a breakpoint in `print_file()` function (`0x8048538`) -

![](https://lh6.googleusercontent.com/2-Nce5ub0KWkEXg-g9RvuudWApY5iG78H2RZoc7Ev7ySLqPixCMSWmP1aQOHIEOVNsM4h6qvoImwiA5hs04LKktTQ-8fQxRcUkBnIwPCymM6jLEYTZdfBbze6Lw_PYSWmgKjRkOt)

Amazing! As we thought exactly.

We now fully understand our purpose:

We need to write "flag.txt" to some memory address and pass that address as a parameter to the print_file function.  
  

These are the steps we want to perform:

1) Find a memory location with *write* permissions where we can write our data.

2) Find a way to place values from the stack to the memory address we want.  (We can only write directly to the stack.)

3) Write a payload that places “flag.txt” in a memory address,	 and call print_file() with this address as an argument.

  
  
  
  
  

## 1. Find a Writable Place

 

I will check the sections and their permissions in order to look for places where you may write.

![](https://lh4.googleusercontent.com/A0iLlFMOVRjB_1MPftWiALhh_Su-Waj3GIc-kX-nd4knytcKPUvKAuKvhB7uOTBl4dRFkvw9svkFDynROck22LTxUsDhk9gJSbR9zVfkWcuVxmmSq_yIOTpzYd88wq13yB7ANgoe)

There are a few sections with `write` permissions (w), I chose to store our string in the.data section.

The base address of this section is `0x0804a018`.

  

## 2. Find Gadgets

  

Next, we will search for a gadget that serves our purpose.

During the work I noticed an interesting symbol:

![](https://lh5.googleusercontent.com/9nguniyiX5i_YwvsuV8b3KPXdWNt-NGmGRDqUSGTLS3l8rywfAIn2ZgKesdOakT2nqsqswOwfTvcoh5hUMVcnY-3fNDzDsBBT8UXdCwO_OXYqGSTIjBlMvrcSAG-HV8jH09xATF1)

If we disassemble `usefulGadgets`:

![](https://lh4.googleusercontent.com/tUtB4a7AuIwImAY-DT_Vzz_CqkEQ5c4wJSlwtKq37jTzXlqG-QFmwviCvSVZ65yoNgBg3pCQH797y9x52bZLGetG2wxrnraON0A4wZrt5qxG1EjFpqyM6WQzJ246kpy3TIi2kjcW)

Interesting, we got a gadget that places the value of ebp inside the address of edi! Great! We can now write things to addresses in memory!

Now, the next thing we need to do (in order to use this gadget) is find another gadget that will allow us to transfer our data (`"flag.txt"`) and the address to which we want to register (`0x0804a018`) from the stack to the `ebp` and `edi` registers respectively.  
  
How do we transfer information from the stack to the registers? Using `pop` instructions.

Looking for a suitable gadget:

![](https://lh6.googleusercontent.com/5nk9-85tGGFbQs0Mj3wcIm8N0vopxrvdpMwswoeKNUXQWI20NG1J7Pp9-DGwCQSCLRyrA2hmMRVWmLxLar-IYkDClp6NarkCY9euXn3n-7MOxZ0ODkXDOJMgnnetFJOFs7sbpaEw)

We found the perfect match!

Yes, in talking about `pop edi; pop ebp; ret;` in `(0x080485aa)`  
  

  

We found a gadget that does exactly what we wanted - Moves data from the stack to edi and ebp.

Now we can move on to writing our payload phase.

  

For clarification, this is what our stack should look like in general:

  
![](https://docs.google.com/drawings/u/0/d/sjRmKbYF3Bkcv0dQBO9_RwA/image?w=483&h=265&rev=161&ac=1&parent=1w1tKXv6q-WxivHxSLVGf6zbY3pOX8Rl4OrYVNiP8MRw)

  

## 3. Putting it all together

![](https://lh6.googleusercontent.com/ShY51EC_0_yl8dLgNfmBOLQsTG9sC-Mm_VBrRPb7zPWzQRzz1nQlVC0fMz3sgWVo1hhVGR2RSRdgGCWr-CegKUo73-6OPhd7KnxD460P_M7JlaaxsmI9-XI8_vTQg9lRk_PHLD51)

  

As you can see, I separated the writing process into two parts. Each time, I wrote four bytes - `"flag"` and then `".txt"`

  
For the first part I used the gadgets we found to write `“flag”` to `0x0804a018`

Afterward, I did the same thing with `“.txt”`, but this time I wrote to `0x0804a018 + 4`

  
Let’s see what our payload looks like in Debugger:

1.  We will write our payload to the file (`$ python2 ./rop.py > payload`)
    
2.  Put breakpoints in the following places to keep track of what's happening in the stack:
    

-   In the first gadget, `pop edi; pop ebp; ret;` → `(0x080485aa)`
    

-   In the second gadget - `mov dword [edi], ebp` → `(0x08048543)`
    
-   In `print_file()` function → `(0x8048538)`
    

  
![](https://lh5.googleusercontent.com/spebVmliktg9IMcVaogxyvMCHVxKjRyOtnqp6-r81Kb228Xg9WUEO-FFa-JhurF3pGBMyr8uvnEhkf2Vhpi67_Hf8zuBVcBZQD9pAmmv-xvJT0buFGeCUB65OjN7gJ9rABgH8CeW)

  

![](https://lh5.googleusercontent.com/bo72mCRzOcUBYAqdqptSKdcmMa_L41bPrxdDJFqzizAyn_nwIRX6N_AMrdUih_fizo7NaW266FbmVFpJkKLH-OnfGRKHSF8csqkn2yGBd-6hK02I3-2SHdzEB2G_WbGo_lsI0KN0)

  
1st breakpoint at the pop gadget:
![](https://lh6.googleusercontent.com/ekgrzIVRGAQll-wwoVAO-syFiscQgYkGuYNFOoOToRN6ifKcqZe_3AOn1CEr-1kVFEe5DoIb1MZuDtTTow-Hu-0MeZNhDXpDhGur-rmfDxKyyGpYZ_K4KgIwbiUZjtLev5qJm87_)

  
  
![](https://lh4.googleusercontent.com/PWrFHzEuatdQkgl240FHIve-K9tYj-LLvekfGJA-r9By692Ud1RVd2ef3YI9Q3OtWgvL9KIdOqmY8xyD5vO_x3BdBNZUHmiW-6QK5oP8srVfWnTntQeJWJY1YFWNCJSl8VcE9-7D)  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  

As you can see, at the end of the first gadget, `ebp = "flag"` and `edi = 0x804a018`, exactly as we wanted!

  

Next up is the second gadget: writing to a memory address.
![](https://lh4.googleusercontent.com/GdwExIvqUl_gWxfihqIv7mOzirRS54dJZxThWUikclKlplWav6vQ-IqdDjV6S0h5o7zCr0wDB_Ep6otj-Ux8J_PDhK9nZP8ZD3Tt-O2loEWYnxdPjcynssIIVthcSYz8G_eAAQ5I)After running the second gadget, now the address in `edi` contains the value we wanted (`"flag"`).

  

Also, note that after the `ret` that runs now, we will go back to the first gadget and start writing the rest of the data - `".txt"`

  

We will continue the program until we reach the `print_file()` function:

  

  
![](https://lh4.googleusercontent.com/z8VlOEi8N8W48lV13hPDV5nfoO7ViklsUyJ-eYzKNMLXF5nLbqTpGQX83pYmDo_YnW6uHO1araqhBPY-RUHOiQ8ZpgmsF1ToyebROGHW-lukOu13kuJpQv_Ks4L4o-1CAePvZFXw)Excellent, `print_file()` will run now, receiving as a parameter the full name of our file -”flag.txt”

We will keep running and get our flag:

![](https://lh6.googleusercontent.com/PFyKrhNda0cyXXtm6oXaM8UUlsAL3r2_SAsRRl9cAvpO5zQkBmmfbd3sHQCAxVERp00j1HVKBIx77Fw5XIdV8PkVJoZA4LBBzyt-vVHm1Idxqd4EAsrzJBWXjmhewzUIGkIL3w6t)
