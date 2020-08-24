### celery基本工作流程
![image.png](/uploads/087DE98EA5F0487C9FAEE86910412EA0/image.png)


### 项目地址
https://github.com/gocelery/gocelery

### go worker
```
package main

import (
	"time"
	"github.com/gocelery/gocelery"
	"github.com/gomodule/redigo/redis"
)

func add(x int, y int) int {
    return x + y
}

func main() {

	// create redis connection pool
	redisPool := &redis.Pool{
		MaxIdle:     3,                 // maximum number of idle connections in the pool
		MaxActive:   0,                 // maximum number of connections allocated by the pool at a given time
		IdleTimeout: 240 * time.Second, // close connections after remaining idle for this duration
		Dial: func() (redis.Conn, error) {
			c, err := redis.DialURL("redis://")
			if err != nil {
				return nil, err
			}
			return c, err
		},
		TestOnBorrow: func(c redis.Conn, t time.Time) error {
			_, err := c.Do("PING")
			return err
		},
	}

	// initialize celery client
	cli, _ := gocelery.NewCeleryClient(
		gocelery.NewRedisBroker(redisPool),
		&gocelery.RedisCeleryBackend{Pool: redisPool},
		5, // number of workers
	)

	// register task
	cli.Register("worker.add", add)

	// start workers (non-blocking call)
	cli.StartWorker()

	// wait for client request
	time.Sleep(10000 * time.Second)

	// stop workers gracefully (blocking call)
	//cli.StopWorker()
}
```
### go client
```
package main

import (
	"C"
	"log"
	"reflect"
	"time"
	"github.com/gocelery/gocelery"
	"github.com/gomodule/redigo/redis"
)

// create redis connection pool
var redisPool = &redis.Pool{
	MaxIdle:     3,                 // maximum number of idle connections in the pool
	MaxActive:   0,                 // maximum number of connections allocated by the pool at a given time
	IdleTimeout: 240 * time.Second, // close connections after remaining idle for this duration
	Dial: func() (redis.Conn, error) {
		c, err := redis.DialURL("redis://")
		if err != nil {
			return nil, err
		}
		return c, err
	},
	TestOnBorrow: func(c redis.Conn, t time.Time) error {
		_, err := c.Do("PING")
		return err
	},
}

// initialize celery client
var cli, _ = gocelery.NewCeleryClient(
	gocelery.NewRedisBroker(redisPool),
	&gocelery.RedisCeleryBackend{Pool: redisPool},
	1,
)

//export add
func add(a,b int) {

	// prepare arguments
	taskName := "worker.add"

	// run task
	asyncResult, err := cli.Delay(taskName, a, b)
	if err != nil {
		panic(err)
	}

	// get results from backend with timeout
	res, err := asyncResult.Get(10 * time.Second)
	if err != nil {
		panic(err)
	}
	log.Printf("result: %+v of type %+v", res, reflect.TypeOf(res))

}
func main() {}
```
### python 执行任务动态库方式
```
go build -buildmode=c-shared -o add.so goclient/main.go

from ctypes import CDLL

add = CDLL('./add.so')
print(add.add(2,2))
```

### python 伪装
```
from celery import Celery

app = Celery(
    'tasks',
    broker='redis://',
    backend='redis://',
)

app.conf.update(
    CELERY_TASK_SERIALIZER='json',
    CELERY_ACCEPT_CONTENT=['json'],  # Ignore other content
    CELERY_RESULT_SERIALIZER='json',
    CELERY_ENABLE_UTC=True,
    CELERY_TASK_PROTOCOL=1,
)

@app.task
def add(a, b):
    return a + b

ar = add.apply_async(args=(5,2), serializer='json', expires=120)
print('Result: {}'.format(ar.get()))
```