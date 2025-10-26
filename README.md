# Linux-IPC--Pipes
Linux-IPC-Pipes

#### Name: SHYAM S
#### Reg.No: 212223240156

# Ex03-Linux IPC - Pipes

# AIM:
To write a C program that illustrate communication between two process using unnamed and named pipes

# DESIGN STEPS:

### Step 1:

Navigate to any Linux environment installed on the system or installed inside a virtual environment like virtual box/vmware or online linux JSLinux (https://bellard.org/jslinux/vm.html?url=alpine-x86.cfg&mem=192) or docker.

### Step 2:

Write the C Program using Linux Process API - pipe(), fifo()

### Step 3:

Testing the C Program for the desired output. 

# PROGRAM:

## C Program that illustrate communication between two process using unnamed pipes using Linux API system calls
```
// unnamed_pipes.c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/wait.h>
#include <errno.h>

#define MAXBUF 2048

void server(int rfd, int wfd);
void client(int wfd, int rfd);

int main(void) {
    int p1[2], p2[2];
    pid_t pid;

    if (pipe(p1) == -1 || pipe(p2) == -1) {
        perror("pipe");
        exit(EXIT_FAILURE);
    }

    pid = fork();
    if (pid < 0) {
        perror("fork");
        exit(EXIT_FAILURE);
    }

    if (pid == 0) {
        /* Child: server */
        close(p1[1]); /* close write end of pipe1 (client->server) */
        close(p2[0]); /* close read end of pipe2 (server->client) */
        server(p1[0], p2[1]);
        close(p1[0]); close(p2[1]);
        exit(0);
    } else {
        /* Parent: client */
        close(p1[0]); /* close read end of pipe1 */
        close(p2[1]); /* close write end of pipe2 */
        client(p1[1], p2[0]);
        close(p1[1]); close(p2[0]);
        wait(NULL);
    }

    return 0;
}

void server(int rfd, int wfd) {
    ssize_t n;
    char fname[MAXBUF];
    char buff[MAXBUF];

    /* Read filename from pipe */
    n = read(rfd, fname, sizeof(fname));
    if (n <= 0) {
        if (n == 0) fprintf(stderr, "server: no filename received\n");
        else perror("server read");
        return;
    }
    fname[n] = '\0';

    int fd = open(fname, O_RDONLY);
    if (fd < 0) {
        const char *err = "ERROR: can't open file\n";
        write(wfd, err, strlen(err));
        return;
    }

    /* Read from file and write to pipe */
    while ((n = read(fd, buff, sizeof(buff))) > 0) {
        ssize_t written = 0;
        while (written < n) {
            ssize_t w = write(wfd, buff + written, n - written);
            if (w < 0) { perror("server write"); close(fd); return; }
            written += w;
        }
    }
    close(fd);
}

void client(int wfd, int rfd) {
    ssize_t n;
    char fname[MAXBUF];
    char buff[MAXBUF];

    /* Read filename from stdin (allow spaces) */
    printf("Enter filename: ");
    if (fgets(fname, sizeof(fname), stdin) == NULL) {
        perror("fgets");
        return;
    }
    /* remove trailing newline */
    fname[strcspn(fname, "\n")] = '\0';

    /* Send filename length only */
    size_t flen = strlen(fname) + 1; /* include NUL */
    if (write(wfd, fname, flen) != (ssize_t)flen) {
        perror("client write");
        return;
    }

    /* Read file contents from server and print to stdout */
    while ((n = read(rfd, buff, sizeof(buff))) > 0) {
        ssize_t out = write(STDOUT_FILENO, buff, n);
        if (out < 0) { perror("write stdout"); return; }
    }
    if (n < 0) perror("client read");
}

```
## OUTPUT
<img width="528" height="287" alt="image" src="https://github.com/user-attachments/assets/f5de3365-1593-4f25-a79f-4a5032ade4b3" />

## C Program that illustrate communication between two process using named pipes using Linux API system calls
```
// named_fifo.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>
#include <errno.h>
#include <sys/wait.h>

#define FIFO_FILE "/tmp/my_fifo"
#define FILE_NAME "hello.txt"
#define BUFSIZE 1024

void server(void);
void client(void);

int main(void) {
    pid_t pid;

    /* Create FIFO if doesn't exist */
    if (mkfifo(FIFO_FILE, 0666) == -1) {
        if (errno != EEXIST) {
            perror("mkfifo");
            exit(EXIT_FAILURE);
        }
    }

    pid = fork();
    if (pid < 0) {
        perror("fork");
        unlink(FIFO_FILE);
        exit(EXIT_FAILURE);
    }

    if (pid == 0) {
        /* Child: client (reader) - open FIFO for reading first so server won't block */
        client();
        exit(0);
    } else {
        /* Parent: server (writer) */
        server();
        wait(NULL);
        /* Cleanup FIFO */
        unlink(FIFO_FILE);
    }

    return 0;
}

/* Server: read FILE_NAME and write to FIFO */
void server(void) {
    int file_fd = open(FILE_NAME, O_RDONLY);
    if (file_fd == -1) {
        perror("Server: open input file");
        return;
    }

    /* Open FIFO for writing; this will block until a reader opens it (client) */
    int fifo_fd = open(FIFO_FILE, O_WRONLY);
    if (fifo_fd == -1) {
        perror("Server: open fifo for write");
        close(file_fd);
        return;
    }

    char buf[BUFSIZE];
    ssize_t n;
    while ((n = read(file_fd, buf, sizeof(buf))) > 0) {
        ssize_t written = 0;
        while (written < n) {
            ssize_t w = write(fifo_fd, buf + written, n - written);
            if (w < 0) { perror("Server write"); close(file_fd); close(fifo_fd); return; }
            written += w;
        }
    }
    close(file_fd);
    close(fifo_fd);
}

/* Client: open FIFO for reading and print contents to stdout */
void client(void) {
    int fifo_fd = open(FIFO_FILE, O_RDONLY);
    if (fifo_fd == -1) {
        perror("Client: open fifo for read");
        return;
    }

    char buf[BUFSIZE];
    ssize_t n;
    while ((n = read(fifo_fd, buf, sizeof(buf))) > 0) {
        ssize_t out = write(STDOUT_FILENO, buf, n);
        if (out < 0) { perror("Client write stdout"); close(fifo_fd); return; }
    }
    if (n < 0) perror("Client read");
    close(fifo_fd);
}
```
## OUTPUT
<img width="512" height="227" alt="image" src="https://github.com/user-attachments/assets/3f30e85a-aada-441f-9e35-53a95fa132a6" />


# RESULT:
The program is executed successfully.
