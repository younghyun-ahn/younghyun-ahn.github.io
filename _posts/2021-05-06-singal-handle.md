---
layout: post
title: "Singal Handle"
---

Interrupt를 발생시켜 운영체제가 프로그램에 제제를 걸 수 있고 반대로 특정 입력이 들어올때 시그널을 잡아서 처리. 사용자 인터럽트를 발생 -> Ctrl+C / Ctrl+Z / Alt+F4

Singal은 신호라는 의미로 프로세스간 서로 통신할 때 사용. 굉장히 작은 값으로 Interrupt 라고 부르기도 한다. 용도가 제한적이며 여러 시그널이 겹칠 경우 원치 않는 결과가 발생할 수 있다. 운영체제 시그널 목록은 kill -l 명령으로 확인.

Singal handler 는 시그널을 받았을때 특정 동작을 하도록 정의 하는 코드(함수)이다. 보통의 방식으로 종료 or 무시 or 재시작 처리 한다.

시그널 처리 유형
* 종료
* 무시
* 코어 덤프
* 중단
* 재작


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

```console
# kill -15
❯ go build main.go
❯ main      
awaiting signal
terminated
existing

# Ctrl+C
❯ main
awaiting signal
^Cinterrupt
existing

# kill -9
❯ main
awaiting signal
[1]    13673 killed     main
```

Check out the [GitHub repository](https://github.com/younghyun-ahn/go-etc/blob/master/signal_handle/main.go) for more info.