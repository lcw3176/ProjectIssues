# 대기열
## 줄을 서시오
현재 서버 스펙이 여러 유저들의 이미지를 동시에 처리할 상황이 안된다. 그래서 요청이 오는대로 한명씩 처리를 하고 있는데, 이걸 어떤 방식으로 처리할 지에 대한 고민이 있었다.

1. 백그라운드 스레드를 통한 처리
2. Async를 통한 비동기 처리

```java
private volatile Queue<UserMapRequest> taskQueue = new LinkedList<>();
    
    @PostConstruct
    private void init(){
        Thread thread = new Thread(() -> {
            while(true){
                while(taskQueue.size() >= 1) {
                    /// 지도 이미지 처리 로직
                }

                Thread.sleep(500);
            }
        });

        thread.start();
    }
```
처음에는 백그라운드 스레드를 하나 만들어서, 큐에 유저 목록이 차면 이미지 처리를 하게 해놓았다.
그런데 그냥 쌩 while문으로 돌려 놓으면 cpu 점유율이 하는일도 없는데 높아져서 보기 불편하고, sleep을 중간중간 넣어주니 뭔가 효율적이라는 생각이 들지가 않았다.
그래서 Async를 통한 비동기 처리를 생각했다.

## 무슨 일 나면 깨워주세요
고민후에 작성한 코드는 다음과 같다.
위에서 하던 작업과 큰 맥락의 차이는 없지만, 중간중간 sleep을 통한 리소스 낭비가 사라져서 훨씬 효율적이라고 생각한다.
```java
@Configuration
@EnableAsync
public class AsyncConfig extends AsyncConfigurerSupport {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(1);
        executor.setMaxPoolSize(1);
        executor.setQueueCapacity(50);
        executor.setThreadNamePrefix("CAPTURE-ASYNC-");
        executor.setKeepAliveSeconds(60 * 10);
        executor.initialize();
        return executor;
    }
}

@Service
public class TaskService {

    @Async
    public void execute(UserMapRequest request, WebSocketSession session){
        // 이미지 캡쳐 및 UUID 전송 작업...
    }

}

```



