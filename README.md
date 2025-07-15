# Kilo code study

Since i usually just code and experiment i decided to try something different and study someone else code.
I'll analyze and write up what i've learned from this [project](https://github.com/antirez/kilo)

Being my first time, i decided to start from the main function and from there go step by step.
Thats because starting from line 0 would be confusing and i wouldn't understand the real workflow of the program.

Step by step means that i will most of the times get a piece of code and test it separately, trying to understand
what it does in details.

So let's start.

```c

int main(int argc, char **argv) {
    if (argc != 2) {
        fprintf(stderr,"Usage: kilo <filename>\n");
        exit(1);
    }

    initEditor();
    editorSelectSyntaxHighlight(argv[1]);
    editorOpen(argv[1]);
    enableRawMode(STDIN_FILENO);
    editorSetStatusMessage(
        "HELP: Ctrl-S = save | Ctrl-Q = quit | Ctrl-F = find");
    while(1) {
        editorRefreshScreen();
        editorProcessKeypress(STDIN_FILENO);
    }
    return 0;
}
```
Thats what the main function looks like.

Starting from the top we can see that the program accept only one argument, namely the file name. Everything's good as of now.

Then ***initEditor()*** gets called.

```c
static struct editorConfig E;

void initEditor(void) {
    E.cx = 0;
    E.cy = 0;
    E.rowoff = 0;
    E.coloff = 0;
    E.numrows = 0;
    E.row = NULL;
    E.dirty = 0;
    E.filename = NULL;
    E.syntax = NULL;
    updateWindowSize();
    signal(SIGWINCH, handleSigWinCh);
}

// The struct we're dealing with. I put it here so you've a reference.

struct editorConfig {
    int cx,cy;  /* Cursor x and y position in characters */
    int rowoff;     /* Offset of row displayed. */
    int coloff;     /* Offset of column displayed. */
    int screenrows; /* Number of rows that we can show */
    int screencols; /* Number of cols that we can show */
    int numrows;    /* Number of rows */
    int rawmode;    /* Is terminal raw mode enabled? */
    erow *row;      /* Rows */
    int dirty;      /* File modified but not saved. */
    char *filename; /* Currently open filename */
    char statusmsg[80];
    time_t statusmsg_time;
    struct editorSyntax *syntax;    /* Current syntax highlight, or NULL. */
};


```
What does this function do? 
1) Well it mostly initializes eveything to NULL or 0, depending if the variable is int or pointer.
2) calls ***updateWindowSize()*** function.

```c
void updateWindowSize(void) {
    if (getWindowSize(STDIN_FILENO,STDOUT_FILENO,
                      &E.screenrows,&E.screencols) == -1) {
        perror("Unable to query the screen for size (columns / rows)");
        exit(1);
    }
    E.screenrows -= 2; /* Get room for status bar. */
}
``` 
3) Here we find ***getWindowSize(STDIN_FILENO, STDOUT_FILENO, &E.screenrows, &E.screencols)***
```c
/* Try to get the number of columns in the current terminal. If the ioctl()
 * call fails the function will try to query the terminal itself.
 * Returns 0 on success, -1 on error. */
int getWindowSize(int ifd, int ofd, int *rows, int *cols) {
    struct winsize ws;

    if (ioctl(1, TIOCGWINSZ, &ws) == -1 || ws.ws_col == 0) {
        /* ioctl() failed. Try to query the terminal itself. */
        int orig_row, orig_col, retval;

        /* Get the initial position so we can restore it later. */
        retval = getCursorPosition(ifd,ofd,&orig_row,&orig_col);
        if (retval == -1) goto failed;

        /* Go to right/bottom margin and get position. */
        if (write(ofd,"\x1b[999C\x1b[999B",12) != 12) goto failed;
        retval = getCursorPosition(ifd,ofd,rows,cols);
        if (retval == -1) goto failed;

        /* Restore position. */
        char seq[32];
        snprintf(seq,32,"\x1b[%d;%dH",orig_row,orig_col);
        if (write(ofd,seq,strlen(seq)) == -1) {
            /* Can't recover... */
        }
        return 0;
    } else {
        *cols = ws.ws_col;
        *rows = ws.ws_row;
        return 0;
    }

failed:
    return -1;
}
```
Now the fun part begin.
Starting from the parameters.
***int ifd --> input fd.*** 
***int ofd --> output fd.***
***int *rows --> pointer where number of rows will be stored.***
***int *cols --> same but for columns***

First thing i've noticed is that a winsize struct gets decleared.
By using 'go to definition' command i found that this struct is defined inside the ***ioctl-types.h*** file as it follows.
In order to use that we need this ***#include <sys/ioctl.h>*** in our file.

```c
struct winsize
  {
    unsigned short int ws_row;
    unsigned short int ws_col;
    unsigned short int ws_xpixel;
    unsigned short int ws_ypixel;
  };

```
So what we're trying to do, as the function name also suggests, is trying to get the current terminal size (columns and rows).
First we try to call the **ioctl()** function, 
```c
int ioctl(int fd, TIOCGWINSZ, struct winsize *argp);
```
This function takes as parameter an
+ fd: open file descriptor. (1)
+ TIOCGWINSZ being a operation code to retrieve Window sizes, stored inside the Kernel.)
+ *argp: an untyped pointer to memory. (Our pointer to a winsize structure) 

IF this function doesn't fail, we're basically done with this function, and everything gets stored inside winsize.
But antirez decided not to rely only on a function.
So if the function returns -1 or the number of columns is 0 (How's that possible??) we retrieve the window size by ourselves.

```c
/* ioctl() failed. Try to query the terminal itself. */
        int orig_row, orig_col, retval;

        /* Get the initial position so we can restore it later. */
        retval = getCursorPosition(ifd,ofd,&orig_row,&orig_col);
        if (retval == -1) goto failed;

```
First it gets the original cursor position and stores it inside ***orig_row*** and ***orig_col***.
How's that done?
Let's take a look at the ***getCursorPosition(ifd,ofd,&orig_row,&orig_col);*** function.

```c
/* Use the ESC [6n escape sequence to query the horizontal cursor position
 * and return it. On error -1 is returned, on success the position of the
 * cursor is stored at *rows and *cols and 0 is returned. */
int getCursorPosition(int ifd, int ofd, int *rows, int *cols) {
    char buf[32];
    unsigned int i = 0;

    /* Report cursor location */
    if (write(ofd, "\x1b[6n", 4) != 4) return -1;

    /* Read the response: ESC [ rows ; cols R */
    while (i < sizeof(buf)-1)
    {
        if (read(ifd,buf+i,1) != 1) break;
        if (buf[i] == 'R') break;
        i++;
    }
    buf[i] = '\0';

    /* Parse it. */
    if (buf[0] != ESC || buf[1] != '[') return -1;
    if (sscanf(buf+2,"%d;%d",rows,cols) != 2) return -1;
    return 0;
}
```
The comment itself is pretty much self-explanatory, but i still could not understand it so i had to do some researches.
That led me to understanding ***ANSI ESCAPE SEQUENCES*** which can be used in many ways.
In our case, we use it to retrieve the cursor position by writing ***"\x1b[6n"*** to the ***STDOUT_FILENO***
where ***"\x1b"*** is basically ESC character in Hexa, and ***[6n*** request cursor position and reports as ESC[#;#R.

Since this might not be clear at first let's test it out with a very simple C program.

```c
#include <unistd.h>

int main()
{
    write(STDOUT_FILENO,"\x1b[6n",4);
}
```
outputs:

<img width="1288" height="725" alt="Screenshot from 2025-07-15 18-57-56" src="https://github.com/user-attachments/assets/5b448f01-483f-4187-b82b-083e8389320d" />

As you can we're inside a terminal and if we launch the program it will return the current position of our cursor inside the terminal.
If we keep spamming it will increment until it reaches the end. Once the end is reached the number will be same, unless we use ***clear*** or something else.
Now this should be pretty clear.

Next, we just read the output of our write from the terminal.
```c
/* Read the response: ESC [ rows ; cols R */
    while (i < sizeof(buf)-1)
    {
        if (read(ifd,buf+i,1) != 1) break;
        if (buf[i] == 'R') break;
        i++;
    }
```
Since we're dealing with an array of char where each char is 1 byte, we can just use sizeof to iterate it.
-1 is because we wanna make room for the '\0' null terminator.

We read 1 char at time from ***STDIN_FILENO*** and store it inside ***buf***. If an error occurs or 'R' is found we break.
***Why 'R'?***
because

```c 
"write(STDOUT_FILENO,"\x1b[6n",4);" // "reports as ESC[#;#R."
```
as you can also see from the previous screenshot, that basically means we've reached the end.

Once the end is reached, we put the good ol' '\0' and proceed to parse given string using sscanf, which will eventually store
the results of given rows and columns inside ***"rows"*** and ***"cols"***.

***A goodtakeway from that code is that we can always use functions and at the same times checks for errors.***



 
