

```c
/* gcc -o prlimit prlimit.c    */

#define _GNU_SOURCE
#define _FILE_OFFSET_BITS 64
#include <stdio.h>
#include <time.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/resource.h>
#define errExit(msg) do { perror(msg); exit(EXIT_FAILURE); } while (0)

int
main(int argc, char *argv[])
{
  struct rlimit old, new;
  struct rlimit *newp;
  pid_t pid;

  if (!(argc == 2 || argc == 4)) {
    fprintf(stderr, "Usage: %s <pid> [<new-soft-limit> "
        "<new-hard-limit>]\n", argv[0]);
    exit(EXIT_FAILURE);
  }

  pid = atoi(argv[1]); /* PID of target process */

  newp = NULL;
  if (argc == 4) {
    new.rlim_cur = atoi(argv[2]);
    new.rlim_max = atoi(argv[3]);
    newp = &new;
  } else {
    new.rlim_cur = RLIM_INFINITY;
    new.rlim_max = RLIM_INFINITY;
    newp = &new;
  }

  if (prlimit(pid, RLIMIT_CORE, newp, &old) == -1)
    errExit("prlimit-1");
  printf("Previous core file size limits: soft=%lld; hard=%lld\n",
      (long long) old.rlim_cur, (long long) old.rlim_max);

  /* Retrieve and display new CPU time limit */

  if (prlimit(pid, RLIMIT_CORE, NULL, &old) == -1)
    errExit("prlimit-2");
  printf("New core file size limits: soft=%lld; hard=%lld\n",
      (long long) old.rlim_cur, (long long) old.rlim_max);

  if ( old.rlim_cur != new.rlim_cur ||
      old.rlim_max != new.rlim_max) {
    fprintf(stderr , "failed to change core file size of process %d .", pid);
    exit(EXIT_FAILURE);
  }

  exit(0);
}

```
