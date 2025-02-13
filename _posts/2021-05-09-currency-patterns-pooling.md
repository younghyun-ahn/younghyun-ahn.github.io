---
layout: post
title: "Go Currency Patterns - Pooling"
---

버퍼가 있는 채널을 이용하여 공유가 가능한 리소스의 풀을 생성하고, 이 리소스들을 원하는 개수만큼의 고루틴에서 개별적으로 활용할 수 있는 기능을 작성한 코드가 모여있다. 이 패턴은 데이터베이스 연결이나 메모리 버퍼 등 공유되는 리소스의 정적인 집합을 관리할 때 특히 유용하다. 고루틴이 이런 리소스 중 하나를 사용해야 할 경우에는 리소스를 할당받고 사용한 후 다시 풀에 반환하는 구조로 동작한다.(풀 사이즈 초과시 생성&사용후 폐기)

일종의 컨텍스와 같은 성격의 공유 풀을 생성하고 풀내 팩토리 메서드를 두어 공유 객체를 생성/재사용 한다. 불필요한 전역객체(컨텍스트) 생성을 하지 않고 공유객체를 보다 우아하게 전달하고 관리 할 수 있다. 

## pool.go

```go
// 사용자가 정의한 리소스의 집합을 관리하는 패키지
package pool

import (
	"errors"
	"io"
	"log"
	"sync"
)

// Pool 구조체는 여러 개의 고루틴에서 안전하게 공유학 위한 리소스의 집합을 관리한다.
// 이 풀에서 관리하가 위한 리소스는 io.Closer 인터페이스를 반드시 구현해야 한다.
type Pool struct {
	m         sync.Mutex
	resources chan io.Closer
	factory   func() (io.Closer, error)
	closed    bool
}

// ErrorPoolClosed 에러는 리소스를 획득하려 할 때 풀이 닫혀있는 경우에 발생한다.
var ErrPoolClosed = errors.New("풀이 닫혔습니다.")

// New 함수는 리소스 관리 풀을 생성한다.
// 풀은 새로운 리소스를 할당하기 위한 함수와 풀의 크기를 매개변수로 정의한다.
func New(fn func() (io.Closer, error), size uint) (*Pool, error) {
	if size <= 0 {
		return nil, errors.New("풀의 크기가 너무 작습니다.")
	}

	return &Pool{
		factory:   fn,
		resources: make(chan io.Closer, size),
	}, nil
}

// 풀에서 리소스를 획득하는 메서드
func (p *Pool) Acquire() (io.Closer, error) {
	select {
	// 사용 가능한 리소스가 있는지 검사한다.
	case r, ok := <-p.resources:
		log.Println("리소스 획득:", "공유된 리소스")
		if !ok {
			return nil, ErrPoolClosed
		}
		return r, nil

	// 사용 가능한 리소스가 없는 경우 새로운 리소스를 생성한다.
	default:
		log.Println("리소스 획득:", "새로운 리소스")
		return p.factory()
	}
}

// 풀에 리소스를 반환하는 메서드
func (p *Pool) Release(r io.Closer) {
	// 안전한 작업을 위해 잠금을 설정한다.
	p.m.Lock()
	defer p.m.Unlock()

	// 풀이 닫혔으면 리소스를 해제한다.
	if p.closed {
		r.Close()
		return
	}

	select {
	// 새로운 리소스를 큐에 추가한다
	case p.resources <- r:
		log.Println("리소스 반환:", "리소스 큐에 반환")

	// 리소스 큐가 가득 찬 경우 리소스를 해제한다.
	default:
		log.Println("리소스 반환:", "리소스 해제")
		r.Close()
	}
}

// 풀을 종료하고 생성된 모든 리소스를 해제하는 메서드
func (p *Pool) Close() {
	// 안전한 작업을 위해 잠금을 설정한다
	p.m.Lock()
	defer p.m.Unlock()

	// 풀이 이미 닫혔으면 아무런 작업도 수행하지 않는다.
	if p.closed {
		return
	}

	// 풀을 닫힌 상태로 전환한다.
	p.closed = true

	// 리소스를 해제하기에 앞서 채널을 먼저 닫는다.
	// 그렇지 않으면 데드락에 걸릴 수 있다.
	close(p.resources)

	// 리소스를 해제한다.
	for r := range p.resources {
		r.Close()
	}
}
```

## main.go

```go
// pool 패키지를 이용하여 데이터베이스 연결 풀을 생성하고 활용하는 예제
package main

import (
	"github.com/younghyun-ahn/go-currency-patterns/pooling/pool"
	"io"
	"log"
	"math/rand"
	"sync"
	"sync/atomic"
	"time"
)

const (
	maxGoroutines   = 50 // 실행할 수 있는 고루틴의 최대 개수
	pooledResources = 10  // 풀이 관리할 리소스의 개수
)

// 공유 자원을 표한한 구조체
type dbConnection struct {
	ID int32
}

// dbConnection 타입의 풀에 의해 관리될 수 있도록 io.Closer 인터페이스를 구현한다.
// Close 메서드는 자원의 해제를 담당한다.
func (dbConn *dbConnection) Close() error {
	log.Println("닫힘: 데이터베이스 연결", dbConn.ID)
	return nil
}

// 각 데이터베이스에 유일한 id를 할당하기 위한 변수
var idCounter int32

// 풀이 새로운 리소스가 필요할 때 호출할 팩토리 메서드
func createConnection() (io.Closer, error) {
	id := atomic.AddInt32(&idCounter, 1)
	log.Println("생성: 새 데이터베이스 연결", id)

	return &dbConnection{id}, nil
}

// 애플리케이션 진입점
func main() {
	var wg sync.WaitGroup
	wg.Add(maxGoroutines)

	// 데이터베이스 연결을 관리할 풀을 생성한다.
	p, err := pool.New(createConnection, pooledResources)
	if err != nil {
		log.Println(err)
	}

	// 풀에서 데이터베이스 연결을 가져와 질의를 실행한다.
	for query := 0; query < maxGoroutines; query++ {
		// 각 고루틴에는 질의 값의 복사본을 전달해야 한다.
		// 그렇지 않으면 고루틴들이 동일한 질의 값을 공유하게 된다.
		go func(q int) {
			performQueries(q, p)
			wg.Done()
		}(query)
	}

	// 고루틴의 실행이 종료될 때까지 대기한다.
	wg.Wait()

	// 풀을 닫는다.
	log.Println("프로그램을 종료합니다.")
	p.Close()
}

// 데이터베이스 연결 리소스 풀을 테스트한다.
func performQueries(query int, p *pool.Pool) {
	// 풀에서 데이터베이스 연결 리소스를 획득한다.
	conn, err := p.Acquire()
	if err != nil {
		log.Println(err)
		return
	}

	// 데이터베이스 연결 리소스를 다시 풀로 되돌린다.
	defer p.Release(conn)

	// 질의문이 실행되는 것처럼 얼마 동안 대기한다.
	time.Sleep(time.Duration(rand.Intn(5)) * time.Second)
	log.Printf("질의: QID[%d] CID[%d]\n", query, conn.(*dbConnection).ID)
}
```

## 실행 결과

```console
❯ go build -race pooling/main.go
❯ main
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 1
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 2
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 3
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 5
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 9
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 11
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 10
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 질의: QID[24] CID[10]
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 13

2021/05/09 15:52:51 리소스 획득: 공유된 리소스

2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 14
2021/05/09 15:52:51 질의: QID[38] CID[14]
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 16
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 18
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 4
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 19
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 질의: QID[43] CID[13]
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 7
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 20
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 21
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 22
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 질의: QID[10] CID[22]
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 23
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 17
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 24
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 26
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 25
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 27
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 28
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 29
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 30
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 31
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 32
2021/05/09 15:52:51 질의: QID[17] CID[32]
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 33
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 질의: QID[18] CID[33]
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 34
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 35
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 36
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 37
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 38
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 39
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 6
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 8
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 40
2021/05/09 15:52:51 질의: QID[28] CID[40]
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 12
2021/05/09 15:52:51 리소스 반환: 리소스 큐에 반환
2021/05/09 15:52:51 리소스 반환: 리소스 큐에 반환
2021/05/09 15:52:51 리소스 반환: 리소스 큐에 반환
2021/05/09 15:52:51 리소스 반환: 리소스 큐에 반환
2021/05/09 15:52:51 리소스 반환: 리소스 큐에 반환
2021/05/09 15:52:51 리소스 반환: 리소스 큐에 반환
2021/05/09 15:52:51 리소스 반환: 리소스 큐에 반환
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 15
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 41
2021/05/09 15:52:51 질의: QID[29] CID[41]
2021/05/09 15:52:51 리소스 반환: 리소스 큐에 반환
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 42
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 43
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 44
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 45
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 46
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 47
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 48
2021/05/09 15:52:51 질의: QID[12] CID[24]
2021/05/09 15:52:51 리소스 반환: 리소스 큐에 반환
2021/05/09 15:52:51 리소스 획득: 새로운 리소스
2021/05/09 15:52:51 생성: 새 데이터베이스 연결 49
2021/05/09 15:52:52 질의: QID[26] CID[12]
2021/05/09 15:52:52 질의: QID[14] CID[17]
2021/05/09 15:52:52 질의: QID[35] CID[8]
2021/05/09 15:52:52 리소스 반환: 리소스 큐에 반환
2021/05/09 15:52:52 리소스 반환: 리소스 큐에 반환
2021/05/09 15:52:52 질의: QID[4] CID[34]
2021/05/09 15:52:52 리소스 반환: 리소스 해제
2021/05/09 15:52:52 질의: QID[23] CID[39]
2021/05/09 15:52:52 닫힘: 데이터베이스 연결 8
2021/05/09 15:52:52 질의: QID[34] CID[37]
2021/05/09 15:52:52 리소스 반환: 리소스 해제
2021/05/09 15:52:52 질의: QID[49] CID[1]
2021/05/09 15:52:52 닫힘: 데이터베이스 연결 39
2021/05/09 15:52:52 질의: QID[44] CID[18]
2021/05/09 15:52:52 리소스 반환: 리소스 해제
2021/05/09 15:52:52 닫힘: 데이터베이스 연결 1
2021/05/09 15:52:52 질의: QID[21] CID[9]
2021/05/09 15:52:52 질의: QID[47] CID[10]
2021/05/09 15:52:52 리소스 반환: 리소스 해제
2021/05/09 15:52:52 닫힘: 데이터베이스 연결 18
2021/05/09 15:52:52 리소스 반환: 리소스 해제
2021/05/09 15:52:52 닫힘: 데이터베이스 연결 34
2021/05/09 15:52:52 질의: QID[46] CID[21]
2021/05/09 15:52:52 리소스 반환: 리소스 해제
2021/05/09 15:52:52 닫힘: 데이터베이스 연결 37
2021/05/09 15:52:52 질의: QID[9] CID[26]
2021/05/09 15:52:52 리소스 반환: 리소스 해제
2021/05/09 15:52:52 닫힘: 데이터베이스 연결 9
2021/05/09 15:52:52 리소스 반환: 리소스 해제
2021/05/09 15:52:52 닫힘: 데이터베이스 연결 26
2021/05/09 15:52:52 리소스 반환: 리소스 해제
2021/05/09 15:52:52 닫힘: 데이터베이스 연결 10
2021/05/09 15:52:52 리소스 반환: 리소스 해제
2021/05/09 15:52:52 닫힘: 데이터베이스 연결 21
2021/05/09 15:52:53 질의: QID[27] CID[6]
2021/05/09 15:52:53 질의: QID[5] CID[30]
2021/05/09 15:52:53 질의: QID[6] CID[29]
2021/05/09 15:52:53 질의: QID[25] CID[4]
2021/05/09 15:52:53 질의: QID[20] CID[2]
2021/05/09 15:52:53 질의: QID[22] CID[3]
2021/05/09 15:52:53 질의: QID[8] CID[28]
2021/05/09 15:52:53 질의: QID[11] CID[23]
2021/05/09 15:52:53 질의: QID[1] CID[36]
2021/05/09 15:52:53 리소스 반환: 리소스 해제
2021/05/09 15:52:53 닫힘: 데이터베이스 연결 6
2021/05/09 15:52:53 리소스 반환: 리소스 해제
2021/05/09 15:52:53 닫힘: 데이터베이스 연결 30
2021/05/09 15:52:53 리소스 반환: 리소스 해제
2021/05/09 15:52:53 닫힘: 데이터베이스 연결 29
2021/05/09 15:52:53 리소스 반환: 리소스 해제
2021/05/09 15:52:53 닫힘: 데이터베이스 연결 2
2021/05/09 15:52:53 리소스 반환: 리소스 해제
2021/05/09 15:52:53 닫힘: 데이터베이스 연결 3
2021/05/09 15:52:53 리소스 반환: 리소스 해제
2021/05/09 15:52:53 닫힘: 데이터베이스 연결 28
2021/05/09 15:52:53 리소스 반환: 리소스 해제
2021/05/09 15:52:53 닫힘: 데이터베이스 연결 4
2021/05/09 15:52:53 리소스 반환: 리소스 해제
2021/05/09 15:52:53 닫힘: 데이터베이스 연결 23
2021/05/09 15:52:53 리소스 반환: 리소스 해제
2021/05/09 15:52:53 닫힘: 데이터베이스 연결 36
2021/05/09 15:52:53 질의: QID[48] CID[45]
2021/05/09 15:52:53 리소스 반환: 리소스 해제
2021/05/09 15:52:53 닫힘: 데이터베이스 연결 45
2021/05/09 15:52:54 질의: QID[0] CID[7]
2021/05/09 15:52:54 질의: QID[31] CID[43]
2021/05/09 15:52:54 질의: QID[36] CID[11]
2021/05/09 15:52:54 질의: QID[13] CID[49]
2021/05/09 15:52:54 질의: QID[32] CID[31]
2021/05/09 15:52:54 질의: QID[15] CID[25]
2021/05/09 15:52:54 질의: QID[19] CID[35]
2021/05/09 15:52:54 질의: QID[7] CID[27]
2021/05/09 15:52:54 질의: QID[39] CID[15]
2021/05/09 15:52:54 질의: QID[42] CID[44]
2021/05/09 15:52:54 질의: QID[3] CID[46]
2021/05/09 15:52:54 리소스 반환: 리소스 해제
2021/05/09 15:52:54 닫힘: 데이터베이스 연결 7
2021/05/09 15:52:54 리소스 반환: 리소스 해제
2021/05/09 15:52:54 닫힘: 데이터베이스 연결 43
2021/05/09 15:52:54 리소스 반환: 리소스 해제
2021/05/09 15:52:54 닫힘: 데이터베이스 연결 11
2021/05/09 15:52:54 리소스 반환: 리소스 해제
2021/05/09 15:52:54 닫힘: 데이터베이스 연결 49
2021/05/09 15:52:54 리소스 반환: 리소스 해제
2021/05/09 15:52:54 닫힘: 데이터베이스 연결 31
2021/05/09 15:52:54 리소스 반환: 리소스 해제
2021/05/09 15:52:54 닫힘: 데이터베이스 연결 25
2021/05/09 15:52:54 리소스 반환: 리소스 해제
2021/05/09 15:52:54 닫힘: 데이터베이스 연결 35
2021/05/09 15:52:54 리소스 반환: 리소스 해제
2021/05/09 15:52:54 닫힘: 데이터베이스 연결 27
2021/05/09 15:52:54 리소스 반환: 리소스 해제
2021/05/09 15:52:54 닫힘: 데이터베이스 연결 15
2021/05/09 15:52:54 리소스 반환: 리소스 해제
2021/05/09 15:52:54 닫힘: 데이터베이스 연결 44
2021/05/09 15:52:54 리소스 반환: 리소스 해제
2021/05/09 15:52:54 닫힘: 데이터베이스 연결 46
2021/05/09 15:52:55 질의: QID[2] CID[48]
2021/05/09 15:52:55 질의: QID[45] CID[16]
2021/05/09 15:52:55 질의: QID[30] CID[20]
2021/05/09 15:52:55 질의: QID[33] CID[5]
2021/05/09 15:52:55 질의: QID[40] CID[42]
2021/05/09 15:52:55 질의: QID[41] CID[47]
2021/05/09 15:52:55 리소스 반환: 리소스 해제
2021/05/09 15:52:55 닫힘: 데이터베이스 연결 48
2021/05/09 15:52:55 리소스 반환: 리소스 해제
2021/05/09 15:52:55 닫힘: 데이터베이스 연결 16
2021/05/09 15:52:55 질의: QID[16] CID[38]
2021/05/09 15:52:55 리소스 반환: 리소스 해제
2021/05/09 15:52:55 닫힘: 데이터베이스 연결 20
2021/05/09 15:52:55 리소스 반환: 리소스 해제
2021/05/09 15:52:55 닫힘: 데이터베이스 연결 38
2021/05/09 15:52:55 질의: QID[37] CID[19]
2021/05/09 15:52:55 리소스 반환: 리소스 해제
2021/05/09 15:52:55 닫힘: 데이터베이스 연결 5
2021/05/09 15:52:55 리소스 반환: 리소스 해제
2021/05/09 15:52:55 닫힘: 데이터베이스 연결 19
2021/05/09 15:52:55 리소스 반환: 리소스 해제
2021/05/09 15:52:55 닫힘: 데이터베이스 연결 42
2021/05/09 15:52:55 리소스 반환: 리소스 해제
2021/05/09 15:52:55 닫힘: 데이터베이스 연결 47
2021/05/09 15:52:55 프로그램을 종료합니다.
2021/05/09 15:52:55 닫힘: 데이터베이스 연결 14
2021/05/09 15:52:55 닫힘: 데이터베이스 연결 13
2021/05/09 15:52:55 닫힘: 데이터베이스 연결 22
2021/05/09 15:52:55 닫힘: 데이터베이스 연결 32
2021/05/09 15:52:55 닫힘: 데이터베이스 연결 33
2021/05/09 15:52:55 닫힘: 데이터베이스 연결 40
2021/05/09 15:52:55 닫힘: 데이터베이스 연결 41
2021/05/09 15:52:55 닫힘: 데이터베이스 연결 24
2021/05/09 15:52:55 닫힘: 데이터베이스 연결 12
2021/05/09 15:52:55 닫힘: 데이터베이스 연결 17
```