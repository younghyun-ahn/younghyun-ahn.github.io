---
layout: post
title: "Singal Handle"
---

`Interrupt`를 발생시켜 운영체제가 프로그램에 제재를 걸수 있다. 반대로 특정 입력이 들어올때 `Signal`을 잡아서 처리할수 있다. Singal은 신호라는 의미로 프로세스간 서로 통신할 때 사용한다. 굉장히 작은 값으로 Interrupt 라고 부르기도 한다. 용도가 제한적이며 여러 시그널이 겹칠 경우 원치 않는 결과가 발생할 수 있다.

`Singal handler`는 시그널을 받았을때 특정 동작을 하도록 정의 하는 코드(함수)이다. 보통의 방식으로 종료하거나 무시 또는 재시작 처리 한다. 운영체제 인터럽트를 수신하고 Singal handler를 정의하여 응용프로그램이 (가능한)깔끔하게 종료될 수 있도록 처리하는 것 이 목적이다.

![screenshot](../../../assets/images/dont_sigkill.png)

출처: [www.quora.com](https://www.quora.com/What-is-the-difference-between-the-SIGINT-and-SIGTERM-signals-in-Linux-What%E2%80%99s-the-difference-between-the-SIGKILL-and-SIGSTOP-signals)

## 시그널 처리 유형

* 종료
* 무시
* 코어 덤프
* 중단
* 재시작

## 시그널 정보

```terminal
❯ kill -l
HUP INT QUIT ILL TRAP ABRT EMT FPE KILL BUS SEGV SYS PIPE ALRM TERM URG STOP TSTP CONT CHLD TTIN TTOU IO XCPU XFSZ VTALRM PROF WINCH INFO USR1 USR2

❯ man signal
No    Name         Default Action       Description
1     SIGHUP       terminate process    terminal line hangup
2     SIGINT       terminate process    interrupt program
3     SIGQUIT      create core image    quit program
4     SIGILL       create core image    illegal instruction
5     SIGTRAP      create core image    trace trap
6     SIGABRT      create core image    abort program (formerly SIGIOT)
7     SIGEMT       create core image    emulate instruction executed
8     SIGFPE       create core image    floating-point exception
9     SIGKILL      terminate process    kill program
10    SIGBUS       create core image    bus error
11    SIGSEGV      create core image    segmentation violation
12    SIGSYS       create core image    non-existent system call invoked
13    SIGPIPE      terminate process    write on a pipe with no reader
14    SIGALRM      terminate process    real-time timer expired
15    SIGTERM      terminate process    software termination signal
16    SIGURG       discard signal       urgent condition present on socket
17    SIGSTOP      stop process         stop (cannot be caught or ignored)
18    SIGTSTP      stop process         stop signal generated from keyboard
19    SIGCONT      discard signal       continue after stop
20    SIGCHLD      discard signal       child status has changed
21    SIGTTIN      stop process         background read attempted from control terminal
22    SIGTTOU      stop process         background write attempted to control terminal
23    SIGIO        discard signal       I/O is possible on a descriptor (see fcntl(2))
24    SIGXCPU      terminate process    cpu time limit exceeded (see setrlimit(2))
25    SIGXFSZ      terminate process    file size limit exceeded (see setrlimit(2))
26    SIGVTALRM    terminate process    virtual time alarm (see setitimer(2))
27    SIGPROF      terminate process    profiling timer alarm (see setitimer(2))
28    SIGWINCH     discard signal       Window size change
29    SIGINFO      discard signal       status request from keyboard
30    SIGUSR1      terminate process    User defined signal 1
31    SIGUSR2      terminate process    User defined signal 2
```

## Example

```go
// main.go
sigs := make(chan os.Signal, 1)
done := make(chan bool, 1)

signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)
//signal.Notify(sigs, os.Interrupt)
signal.Notify(sigs, os.Kill)

go func() {
	sig := <-sigs
	fmt.Println(sig)
	done <- true
}()

fmt.Println("awaiting signal")
<-done
fmt.Println("existing")
```

### kill -9 (kill program)

```console
❯ go build main.go
❯ main
awaiting signal
[1]    13673 killed     main
```

### kill -15 (terminate signal)

```console
❯ go build main.go
❯ main      
awaiting signal
terminated
existing
```

### Ctrl+C (사용자 인터럽트)

```console
❯ go build main.go
❯ main
awaiting signal
^Cinterrupt
existing
```

Check out the [GitHub repository](https://github.com/younghyun-ahn/go-etc/blob/master/signal_handle/main.go) for more info.