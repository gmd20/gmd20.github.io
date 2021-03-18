makefile 的默认target是第一个 target ，所以把all放到最前面才行。
这个makefile要学习一下才行

```text
SRC     = .
CFLAGS  = -O2 -Wall -g -I../include 
LDFLAGS = -Wl,-rpath="/usr/mylibdir" -L/usr/mylibdir
LIBS    = -lmylib

CC := gcc
AR := ar

objs = main.o util.o \
       test.o

hdrs = test.h

all: test

$(objs): $(hdrs)

.c.o:
	$(CC) $(CFLAGS) -c $<

test: $(objs)
	$(CC) $(LDFLAGS) -o $@ $(objs) $(LIBS)

clean:
	@rm -f test $(SRC)/*.o $(SRC)/*~


.PHONY : all clean

```
