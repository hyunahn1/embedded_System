# 4.3 Linux System Programming

## Topics Covered

### POSIX API (Pthreads, Signals, File I/O)

#### Pthreads (POSIX Threads)

##### Thread Basics
```c
#include <pthread.h>

void *thread_function(void *arg) {
    int *value = (int *)arg;
    printf("Thread received: %d\n", *value);
    return NULL;
}

int main(void) {
    pthread_t thread;
    int arg = 42;
    
    // Create thread
    pthread_create(&thread, NULL, thread_function, &arg);
    
    // Wait for thread to complete
    pthread_join(thread, NULL);
    
    return 0;
}

// Compile: gcc -pthread program.c
```

##### Thread Attributes
```c
pthread_attr_t attr;
pthread_attr_init(&attr);

// Set detached state
pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);

// Set stack size
pthread_attr_setstacksize(&attr, 1024 * 1024); // 1MB

// Set scheduling policy and priority
struct sched_param param;
param.sched_priority = 50;
pthread_attr_setschedpolicy(&attr, SCHED_FIFO);
pthread_attr_setschedparam(&attr, &param);
pthread_attr_setinheritsched(&attr, PTHREAD_EXPLICIT_SCHED);

pthread_create(&thread, &attr, thread_function, NULL);
pthread_attr_destroy(&attr);
```

##### Mutexes
```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void critical_section(void) {
    pthread_mutex_lock(&mutex);
    
    // Critical section
    shared_variable++;
    
    pthread_mutex_unlock(&mutex);
}

// Cleanup
pthread_mutex_destroy(&mutex);
```

##### Condition Variables
```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
int ready = 0;

void *producer(void *arg) {
    pthread_mutex_lock(&mutex);
    ready = 1;
    pthread_cond_signal(&cond);  // Wake up consumer
    pthread_mutex_unlock(&mutex);
    return NULL;
}

void *consumer(void *arg) {
    pthread_mutex_lock(&mutex);
    while (!ready) {
        pthread_cond_wait(&cond, &mutex);  // Wait for signal
    }
    // Process data
    pthread_mutex_unlock(&mutex);
    return NULL;
}
```

#### Signals

##### Signal Handling
```c
#include <signal.h>

void signal_handler(int signum) {
    printf("Caught signal %d\n", signum);
    // Cleanup and exit
    exit(0);
}

int main(void) {
    // Register signal handler
    signal(SIGINT, signal_handler);   // Ctrl+C
    signal(SIGTERM, signal_handler);  // Termination request
    
    while (1) {
        // Main loop
        sleep(1);
    }
    
    return 0;
}
```

##### Advanced Signal Handling
```c
#include <signal.h>

void advanced_handler(int sig, siginfo_t *info, void *context) {
    printf("Signal %d from PID %d\n", sig, info->si_pid);
}

int main(void) {
    struct sigaction sa;
    sa.sa_sigaction = advanced_handler;
    sa.sa_flags = SA_SIGINFO;
    sigemptyset(&sa.sa_mask);
    
    sigaction(SIGUSR1, &sa, NULL);
    
    pause();  // Wait for signal
    return 0;
}
```

#### File I/O

##### Low-Level I/O (System Calls)
```c
#include <fcntl.h>
#include <unistd.h>

int main(void) {
    int fd;
    char buffer[256];
    ssize_t bytes_read;
    
    // Open file
    fd = open("/dev/mydevice", O_RDWR);
    if (fd < 0) {
        perror("open");
        return -1;
    }
    
    // Read from file
    bytes_read = read(fd, buffer, sizeof(buffer));
    if (bytes_read < 0) {
        perror("read");
        close(fd);
        return -1;
    }
    
    // Write to file
    write(fd, "Hello", 5);
    
    // Seek to position
    lseek(fd, 0, SEEK_SET);
    
    // Close file
    close(fd);
    
    return 0;
}
```

##### Standard I/O (Buffered)
```c
#include <stdio.h>

int main(void) {
    FILE *fp;
    char buffer[256];
    
    // Open file
    fp = fopen("data.txt", "r");
    if (!fp) {
        perror("fopen");
        return -1;
    }
    
    // Read line
    while (fgets(buffer, sizeof(buffer), fp)) {
        printf("%s", buffer);
    }
    
    // Close file
    fclose(fp);
    
    return 0;
}
```

##### Memory-Mapped I/O
```c
#include <sys/mman.h>
#include <fcntl.h>

int main(void) {
    int fd;
    void *mapped;
    size_t size = 4096;
    
    fd = open("datafile", O_RDWR);
    
    // Map file to memory
    mapped = mmap(NULL, size, PROT_READ | PROT_WRITE, 
                  MAP_SHARED, fd, 0);
    if (mapped == MAP_FAILED) {
        perror("mmap");
        close(fd);
        return -1;
    }
    
    // Access mapped memory
    char *data = (char *)mapped;
    data[0] = 'A';
    
    // Sync changes to disk
    msync(mapped, size, MS_SYNC);
    
    // Unmap
    munmap(mapped, size);
    close(fd);
    
    return 0;
}
```

### Inter-Process Communication (IPC)

#### Pipes
```c
#include <unistd.h>

int main(void) {
    int pipefd[2];
    pid_t pid;
    char buffer[256];
    
    // Create pipe
    pipe(pipefd);  // pipefd[0] = read, pipefd[1] = write
    
    pid = fork();
    
    if (pid == 0) {
        // Child process
        close(pipefd[1]);  // Close write end
        read(pipefd[0], buffer, sizeof(buffer));
        printf("Child received: %s\n", buffer);
        close(pipefd[0]);
    } else {
        // Parent process
        close(pipefd[0]);  // Close read end
        write(pipefd[1], "Hello from parent", 18);
        close(pipefd[1]);
        wait(NULL);
    }
    
    return 0;
}
```

#### Named Pipes (FIFOs)
```c
#include <sys/stat.h>
#include <fcntl.h>

// Create FIFO
mkfifo("/tmp/myfifo", 0666);

// Writer process
int fd = open("/tmp/myfifo", O_WRONLY);
write(fd, "Hello", 5);
close(fd);

// Reader process
int fd = open("/tmp/myfifo", O_RDONLY);
char buffer[256];
read(fd, buffer, sizeof(buffer));
close(fd);
```

#### Shared Memory
```c
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>

// Create shared memory
int shm_fd = shm_open("/myshm", O_CREAT | O_RDWR, 0666);
ftruncate(shm_fd, 4096);

// Map shared memory
void *ptr = mmap(0, 4096, PROT_READ | PROT_WRITE, 
                 MAP_SHARED, shm_fd, 0);

// Write to shared memory
sprintf((char *)ptr, "Shared data");

// Another process can read
// char *data = (char *)ptr;

munmap(ptr, 4096);
close(shm_fd);
shm_unlink("/myshm");
```

#### Message Queues
```c
#include <mqueue.h>

// Create/open message queue
mqd_t mq;
struct mq_attr attr;
attr.mq_flags = 0;
attr.mq_maxmsg = 10;
attr.mq_msgsize = 256;
attr.mq_curmsgs = 0;

mq = mq_open("/mymq", O_CREAT | O_RDWR, 0666, &attr);

// Send message
char *msg = "Hello MQ";
mq_send(mq, msg, strlen(msg), 0);

// Receive message
char buffer[256];
mq_receive(mq, buffer, sizeof(buffer), NULL);

mq_close(mq);
mq_unlink("/mymq");
```

#### Unix Domain Sockets
```c
#include <sys/socket.h>
#include <sys/un.h>

// Server
int server_fd = socket(AF_UNIX, SOCK_STREAM, 0);
struct sockaddr_un addr;
addr.sun_family = AF_UNIX;
strcpy(addr.sun_path, "/tmp/socket");

bind(server_fd, (struct sockaddr *)&addr, sizeof(addr));
listen(server_fd, 5);

int client_fd = accept(server_fd, NULL, NULL);
char buffer[256];
read(client_fd, buffer, sizeof(buffer));

// Client
int sock = socket(AF_UNIX, SOCK_STREAM, 0);
connect(sock, (struct sockaddr *)&addr, sizeof(addr));
write(sock, "Hello", 5);
```

### Process Management

#### Process Creation
```c
#include <unistd.h>
#include <sys/wait.h>

int main(void) {
    pid_t pid = fork();
    
    if (pid < 0) {
        perror("fork");
        return -1;
    } else if (pid == 0) {
        // Child process
        printf("Child PID: %d\n", getpid());
        execl("/bin/ls", "ls", "-l", NULL);
        // execl only returns on error
        perror("execl");
        exit(1);
    } else {
        // Parent process
        printf("Parent PID: %d, Child PID: %d\n", getpid(), pid);
        
        int status;
        waitpid(pid, &status, 0);
        
        if (WIFEXITED(status)) {
            printf("Child exited with status %d\n", WEXITSTATUS(status));
        }
    }
    
    return 0;
}
```

#### Daemon Process
```c
#include <syslog.h>

void daemonize(void) {
    pid_t pid;
    
    // Fork and exit parent
    pid = fork();
    if (pid > 0) exit(0);  // Parent exits
    
    // Create new session
    setsid();
    
    // Fork again
    pid = fork();
    if (pid > 0) exit(0);
    
    // Change working directory
    chdir("/");
    
    // Close standard file descriptors
    close(STDIN_FILENO);
    close(STDOUT_FILENO);
    close(STDERR_FILENO);
    
    // Open syslog
    openlog("mydaemon", LOG_PID, LOG_DAEMON);
}

int main(void) {
    daemonize();
    
    syslog(LOG_INFO, "Daemon started");
    
    while (1) {
        // Daemon work
        sleep(10);
    }
    
    closelog();
    return 0;
}
```

### Systemd & Init Systems

#### Systemd Service Unit
```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/usr/local/bin
ExecStart=/usr/local/bin/myapp
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

#### Systemd Commands
```bash
# Start service
systemctl start myapp

# Stop service
systemctl stop myapp

# Enable at boot
systemctl enable myapp

# Check status
systemctl status myapp

# View logs
journalctl -u myapp -f
```

#### SysV Init Script
```bash
#!/bin/sh
### BEGIN INIT INFO
# Provides:          myapp
# Required-Start:    $remote_fs $syslog
# Required-Stop:     $remote_fs $syslog
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: My Application
### END INIT INFO

case "$1" in
    start)
        echo "Starting myapp"
        /usr/local/bin/myapp &
        ;;
    stop)
        echo "Stopping myapp"
        killall myapp
        ;;
    restart)
        $0 stop
        $0 start
        ;;
    *)
        echo "Usage: $0 {start|stop|restart}"
        exit 1
        ;;
esac
```

## Key Concepts

- **System Calls**: Kernel entry points (open, read, write, fork, etc.)
- **File Descriptors**: Integer handles to open files (0=stdin, 1=stdout, 2=stderr)
- **Process ID (PID)**: Unique identifier for each process
- **User/Group ID**: Security and permissions

## Practical Exercises

1. Create multi-threaded application with synchronization
2. Implement signal handler for graceful shutdown
3. Use pipes for parent-child communication
4. Create shared memory between processes
5. Write a daemon process
6. Create systemd service unit
7. Implement producer-consumer with message queues

## Best Practices

1. **Always check return values** of system calls
2. **Use pthread_join or pthread_detach** to avoid resource leaks
3. **Close file descriptors** when done
4. **Handle signals** for clean shutdown
5. **Use appropriate IPC** for the use case
6. **Follow RAII** where possible (C++ or cleanup functions)

## Performance Considerations

- **System call overhead**: Minimize frequency
- **Buffered I/O**: Use for better performance
- **Memory mapping**: Efficient for large files
- **Lock contention**: Minimize critical sections

## References

- "The Linux Programming Interface" by Michael Kerrisk
- "Advanced Programming in the UNIX Environment" by W. Richard Stevens
- Linux man pages (man 2 syscall, man 3 function)
- POSIX standards documentation
