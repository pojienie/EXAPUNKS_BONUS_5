# EXAPUNKS - BONUS STAGE 5 - Netronics Net40 Modem

I came up with this solution after I realized I can carry two pieces of information in one number if both pieces of information are numbers and are less than 100. For this task, the two pieces of information are

1. distance between two X: A phone number is only 11 digits long so distance between two X is definitely less than 100.
2. current digit: Each digit is only 0 ~ 9 so each digit is definitely also less than 100.

Thus, instead of X, I want it to be the above two pieces of information. I used 3rd and 4th digit to store the first piece of the information and used 1st and 2nd to store the second piece of the information. This way, I can conveniently access them via `MODI` and `DIVI`. For example, the following phone number is one of the test cases:

```
X 5 1 5 9 X 3 4 7 X 2
```

For the above number, I want to format them to this:

```
9900 5 1 5 9 600 3 4 7 500 2 -3
```

the 4th and 3rd digits are distance between to X. If it's 99, it means there is no X before it, otherwise, it means (distance to previous X * 100). For example, 600 means the distance between 600(second X) and 9900(first X) is 5. 500 means the distance between 500(third X) and 600(second X) is is 6. Why is the distance always off by one? Because you can save one instruction this way. You can just do:

```
DIVI F 100 X ; this advances F position by 1
MULI X -1 X
SEEK X
```

If the distance is not off by one, you need one more instruction:

```
DIVI F 100 X ; this advances F position by 1
MULI X -1 X
SEEK -1
SEEK X
```

The 2nd and 1st digits are the current digit to try.

I also write down an additional value that is also the distance. But it's the distance between the end of the file and the last X. Thus, -3 means the distance between EOF and last X is 2.

With the above stuff in mind, let's get down to the code.

My code is basically 3 major steps:

1. initialization
2. try all phone numbers and copy them if it's correct
3. clean up

## Initialization

For initialization, there are 4 mini-steps:

1. necessary stuff (GRAB... LINK)
2. find the first X and replace it with 9900
3. find the next X and replace it with `distance * 100`
4. when it's EOF, it means every number is scanned and every X is replaced.

### necessary stuff

It's just some boilerplate that must be done:

```
GRAB 300
LINK 800
```

### find the first X and replace it with 9900

```
MARK FIND_FIRST_X
TEST F > -1
TJMP FIND_FIRST_X
SEEK -1
COPY 9900 F
```

### find the next X and replace it with `distance * 100`

```
MARK FIND_REMAINING_X
COPY 100 X

MARK F_R_X_LOOP ;Find Remaining X Loop
TEST EOF        ;if it's end of file, it means all numbers are scanned and all X are replaced
TJMP FOUND_ALL
ADDI X 100 X
TEST F > -1     ; if this is true, it means it's not X.
TJMP F_R_X_LOOP

;found an X! replace X with (distance * 100)
SEEK -1
COPY X F
JUMP FIND_REMAINING_X
```

### when it's EOF, it means every number is scanned and every X is replaced.

But we also need to put down the distance between EOF and last X:

```
MARK FOUND_ALL
DIVI X -100 X
SUBI X 1 F
```

Once all these instructions are executed, the phone number should have all X replaced with useful information. There is also one more number at the end of the file which indicates the distance between the end of the file and the last X.

## try all phone numbers

Now, we should try dialing the numbers. This step can be broken down to several mini-steps

1. try the phone number
2. if it's correct, save them.
3. find the last X
4. add 1 to the current X
5. carry the number
6. go to 1.

### try the phone number

This one is easy. Since we know that each phone number is always 11 digits and I can use `MODI` to retrieve the digit, I can just do the following:

```
MARK TEST_PHONE
SEEK -9999
@REP 11
MODI F 100 #DIAL
@END
```

### if it's correct, save them.

to see if the number is correct, we need another EXA to `LINK 800` and then report it:

```
REPL TEST_LINK
NOOP
NOOP
TEST MRD ; test if my child EXA is still alive
TJMP CORRECT_NUMBER  ;it's still alive! it means the number is valid
JUMP FIND_LAST_DIGIT ;oops, it died. It means it's not a valid phone number

...somewhere else...

MARK TEST_LINK
LINK 800
LINK -1
COPY 0 M ; tell the mother EXA that I am still alive!
```

Now, if it's correct, we need to save them. My way to achieve this to create an EXA at the start of the test case and its sole job is to copy global `M` to a file:


```
; this is for XB
GRAB 301
MARK R
COPY M F
JUMP R
```

As for the main EXA, It'll do this to send the digit:

```
MARK CORRECT_NUMBER
VOID M ; eats the local M value so that the child EXA can die peacefully
COPY -1 #DIAL
SEEK -9999
MODE ;GLOBAL
@REP 11
MODI F 100 M
@END
MODE ;LOCAL
```

### find the last X

Whether it's correct or not, I'll have to add the number by 1 and try again. Thus, I need to find the last X.

```
MARK FIND_LAST_DIGIT
SEEK F
```

Wait what? Just one instruction? Yes. Actually, the original version of this part is this:

```
MARK FIND_LAST_DIGIT
SEEK 9999
SEEK -1
SEEK F
```

The first two lines guarantee that I am accessing the distance between EOF and the last X. However, there are only two ways to reach this part of the program: Right after dialing the number (if the number isn't correct) or right after sending the phone number to another EXA (if the number is correct). In both of these cases, file position will conveniently be 12th position which happens to be the distance between EOF and the last X.

### add 1 to the current X

Just, you know, add 1.

```
MARK ADD_ONE
ADDI F 1 X
SEEK -1
COPY X F
SEEK -1
```

### carry the number

This part is a bit complex because there were several things to check. First, we need to check if it really needs carrying. If it doesn't need carrying, just test the number:

```
MARK CARRY
MODI F 100 X
TEST X < 10
TJMP TEST_PHONE
```

Then, we need to test if it's the first X. If it is, it means we are done!

```
SEEK -1
DIVI F 100 X
TEST X = 99
TJMP DONE
```

Finally, it's not first X and it really needs to carry the number to previous X. After carrying, we need to add 1 to the previous X AND see if it needs carrying as well. Thus, after carrying, we just `SEEK` to the correct file position and `JUMP` to `ADD_ONE` again:

```
SEEK -1
COPY F X
DIVI X 100 T ;SEEK
SUBI X 10 X
SEEK -1
COPY X F

MULI T -1 T
SEEK T
JUMP ADD_ONE
```

Finally, after all the above step, every phone number is tried and we are done! Time to clean up!

## clean up

Since I didn't write any termination code to the previous EXA, I need to kill it:

```
LINK -1
KILL
```

And that's it! As of this writing, this code performs really well judging by the histogram in BOTH cycles and instruction count!

You can find XA's and XB's code in this repository. I also saves the histogram data and test case data in this repository.
