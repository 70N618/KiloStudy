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

# initEditor() function.

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

***A good takeway from that code is that we can always use functions and at the same times checks for errors.***

At this point we're done with the ***getCursorPosition();*** function.

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
        retval = getCursorPosition(ifd,ofd,&orig_row,&orig_col); // 
        if (retval == -1) goto failed;

        /* Go to right/bottom margin and get position. */  <----------- WE'RE HERE
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
What happens next?
As the comment states we're going to bring the cursor on the botto/right margin.
This can be achieved by using Ansi Escape Sequences as we did before.

```c
 if (write(ofd,"\x1b[999C\x1b[999B",12) != 12) goto failed;
```
In this case we have:
+ \x1b: our classic escape sequence
+ [999C -> 	moves cursor right 999 columns
+ \x1b: same as before
+ [999B -> moves cursor bottom 999 columns.
+ 12 because \x1b count as one and others are one each.

That will move our cursor to the bottom right corner of our terminal.

```c
/* Go to right/bottom margin and get position. */  <----------- WE'RE HERE
        if (write(ofd,"\x1b[999C\x1b[999B",12) != 12) goto failed;
        retval = getCursorPosition(ifd,ofd,rows,cols);
        if (retval == -1) goto failed;
``` 
Then guess what? We call the get cursor position again and store data inside rows and cols. (our parameters).
Keep in mind that rows and cols are different from orig_rows/cols which is our starting point.

```c
 /* Restore position. */
        char seq[32];
        snprintf(seq,32,"\x1b[%d;%dH",orig_row,orig_col);
        if (write(ofd,seq,strlen(seq)) == -1) {
            /* Can't recover... */
        }
```

Then we write restore our cursor position back to the original one.
We use a temporary buffer, and this escape sequence. ***"\x1b[%d;%dH"*** with the previously saved orig_row and orig_col.
We're combining both printf with ESC sequences. ***ESC[{line};{column}H*** <-- *Go to line;column #*
If write fails we're cooked in this case. Our cursor will stay at the bottom right corner probably.
Else we're done with this function aswell.
I really liked the way goto were used to handle errors.

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
We're back here. but there's not to much to see.

```c
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
```
So we go back there and here we can see the signal function being used to handle SIGWINCH signal.
That signal is sent when the Window size changes and it's handled with this function.

```c
void handleSigWinCh(int unused __attribute__((unused))) {
    updateWindowSize();
    if (E.cy > E.screenrows) E.cy = E.screenrows - 1;
    if (E.cx > E.screencols) E.cx = E.screencols - 1;
    editorRefreshScreen();
}
```
This calls the chain of function we've just seen but there's something interesting

Parameters:  
- `int __sig`
- `__sighandler_t __handler (aka void (*)(int))`

These are the function parameters. As we can see the signal function, accept a signal, and a pointer to a function that returns void and accept a "dummy int".
But since we're not gonna use that dummy int, we can avoid get a warning from the compiler using ***int unused __attribute__((unused))***

```c
if (E.cy > E.screenrows) E.cy = E.screenrows - 1; E.cy --> E.cy/E.cx are the coordinates of the cursor.
    if (E.cx > E.screencols) E.cx = E.screencols - 1;
```
This is probably to update the cursor position based on the screen rows/cols and preventing it to go out of bounds.

We now have the ***editorRefreshScreen()*** function.

```c
/* This function writes the whole screen using VT100 escape characters
 * starting from the logical state of the editor in the global state 'E'. */
void editorRefreshScreen(void) {
    int y;
    erow *r;
    char buf[32];
    struct abuf ab = ABUF_INIT;

    abAppend(&ab,"\x1b[?25l",6); /* Hide cursor. */
    abAppend(&ab,"\x1b[H",3); /* Go home. */
    for (y = 0; y < E.screenrows; y++) {
        int filerow = E.rowoff+y;

        if (filerow >= E.numrows) {
            if (E.numrows == 0 && y == E.screenrows/3) {
                char welcome[80];
                int welcomelen = snprintf(welcome,sizeof(welcome),
                    "Kilo editor -- verison %s\x1b[0K\r\n", KILO_VERSION);
                int padding = (E.screencols-welcomelen)/2;
                if (padding) {
                    abAppend(&ab,"~",1);
                    padding--;
                }
                while(padding--) abAppend(&ab," ",1);
                abAppend(&ab,welcome,welcomelen);
            } else {
                abAppend(&ab,"~\x1b[0K\r\n",7);
            }
            continue;
        }

        r = &E.row[filerow];

        int len = r->rsize - E.coloff;
        int current_color = -1;
        if (len > 0) {
            if (len > E.screencols) len = E.screencols;
            char *c = r->render+E.coloff;
            unsigned char *hl = r->hl+E.coloff;
            int j;
            for (j = 0; j < len; j++) {
                if (hl[j] == HL_NONPRINT) {
                    char sym;
                    abAppend(&ab,"\x1b[7m",4);
                    if (c[j] <= 26)
                        sym = '@'+c[j];
                    else
                        sym = '?';
                    abAppend(&ab,&sym,1);
                    abAppend(&ab,"\x1b[0m",4);
                } else if (hl[j] == HL_NORMAL) {
                    if (current_color != -1) {
                        abAppend(&ab,"\x1b[39m",5);
                        current_color = -1;
                    }
                    abAppend(&ab,c+j,1);
                } else {
                    int color = editorSyntaxToColor(hl[j]);
                    if (color != current_color) {
                        char buf[16];
                        int clen = snprintf(buf,sizeof(buf),"\x1b[%dm",color);
                        current_color = color;
                        abAppend(&ab,buf,clen);
                    }
                    abAppend(&ab,c+j,1);
                }
            }
        }
        abAppend(&ab,"\x1b[39m",5);
        abAppend(&ab,"\x1b[0K",4);
        abAppend(&ab,"\r\n",2);
    }

    /* Create a two rows status. First row: */
    abAppend(&ab,"\x1b[0K",4);
    abAppend(&ab,"\x1b[7m",4);
    char status[80], rstatus[80];
    int len = snprintf(status, sizeof(status), "%.20s - %d lines %s",
        E.filename, E.numrows, E.dirty ? "(modified)" : "");
    int rlen = snprintf(rstatus, sizeof(rstatus),
        "%d/%d",E.rowoff+E.cy+1,E.numrows);
    if (len > E.screencols) len = E.screencols;
    abAppend(&ab,status,len);
    while(len < E.screencols) {
        if (E.screencols - len == rlen) {
            abAppend(&ab,rstatus,rlen);
            break;
        } else {
            abAppend(&ab," ",1);
            len++;
        }
    }
    abAppend(&ab,"\x1b[0m\r\n",6);

    /* Second row depends on E.statusmsg and the status message update time. */
    abAppend(&ab,"\x1b[0K",4);
    int msglen = strlen(E.statusmsg);
    if (msglen && time(NULL)-E.statusmsg_time < 5)
        abAppend(&ab,E.statusmsg,msglen <= E.screencols ? msglen : E.screencols);

    /* Put cursor at its current position. Note that the horizontal position
     * at which the cursor is displayed may be different compared to 'E.cx'
     * because of TABs. */
    int j;
    int cx = 1;
    int filerow = E.rowoff+E.cy;
    erow *row = (filerow >= E.numrows) ? NULL : &E.row[filerow];
    if (row) {
        for (j = E.coloff; j < (E.cx+E.coloff); j++) {
            if (j < row->size && row->chars[j] == TAB) cx += 7-((cx)%8);
            cx++;
        }
    }
    snprintf(buf,sizeof(buf),"\x1b[%d;%dH",E.cy+1,cx);
    abAppend(&ab,buf,strlen(buf));
    abAppend(&ab,"\x1b[?25h",6); /* Show cursor. */
    write(STDOUT_FILENO,ab.b,ab.len);
    abFree(&ab);
}

/* Set an editor status message for the second line of the status, at the
 * end of the screen. */
void editorSetStatusMessage(const char *fmt, ...) {
    va_list ap;
    va_start(ap,fmt);
    vsnprintf(E.statusmsg,sizeof(E.statusmsg),fmt,ap);
    va_end(ap);
    E.statusmsg_time = time(NULL);
}
```
That's a big function.

Let's start from the comment, 
- '/* This function writes the whole screen using VT100 escape characters'
- '* starting from the logical state of the editor in the global state 'E'. */' 

Whats a VT100? 
"The VT100 is a video terminal, introduced in August 1978 by Digital Equipment Corporation (DEC). It was one of the first terminals to support ANSI escape codes." as stated from Wikipedia.
Even tho physical VT100 are no longer common, their functionality is emulated in many software applications, such as terminal emulators in operating systems like Linux. 

[Here](https://espterm.github.io/docs/VT100%20escape%20codes.html) you can find a good list of VT100 escape codes.

As for the second line of the comment i've yet not discovered what it means.

Let's start with the code.

```c
int y;
    erow *r;
    char buf[32];
    struct abuf ab = ABUF_INIT;

----------------------------- EROW STRUCTURE ------------------------------
/* This structure represents a single line of the file we are editing. */
typedef struct erow {
    int idx;            /* Row index in the file, zero-based. */
    int size;           /* Size of the row, excluding the null term. */
    int rsize;          /* Size of the rendered row. */
    char *chars;        /* Row content. */
    char *render;       /* Row content "rendered" for screen (for TABs). */
    unsigned char *hl;  /* Syntax highlight type for each character in render.*/
    int hl_oc;          /* Row had open comment at end in last syntax highlight
                           check. */
} erow;

------------------------------ ABUF STRUCTURE ------------------------------
/* We define a very simple "append buffer" structure, that is an heap
 * allocated string where we can append to. This is useful in order to
 * write all the escape sequences in a buffer and flush them to the standard
 * output in a single call, to avoid flickering effects. */
struct abuf {
    char *b;
    int len;
};

```
So we've a pointer ***'r'*** to a struct which represents a single line of the file.
And then we've a pointer ab to ***ABUF*** struct, which as stated from the comments,
it is an heap allocated string (allocated with malloc) where we can happend to.
To give you some context.

```c
  abAppend(&ab,"\x1b[?25l",6); /* Hide cursor. */
    abAppend(&ab,"\x1b[H",3); /* Go home. */

void abAppend(struct abuf *ab, const char *s, int len) { 
    char *new = realloc(ab->b,ab->len+len); // -->  We're basically allocating spaces for previous string + len of new string.

    if (new == NULL) return;
    memcpy(new+ab->len,s,len); // Using memcpy to append to the heap allocated string the new string.
    ab->b = new;
    ab->len += len; // Update len and string.
}
```
Thats how the function get used.

