# RankingQuizReadMe

아래의 링크로 서비스에 접속하실 수 있습니다. 

https://rankingquiz.rivercastleworks.site

# Enhancement

1. 퀴즈 메시지 전송 기능 비동기 처리

   1-1. 고민
   
   경쟁에 있어서 가장 중요한 요소 중 하나는 공정성이라고 생각한다. 현재 서비스 이용자들에게 좀 더 **공정한 경쟁환경을 구현**해야겠다고 생각했다. 특히, **퀴즈 데이터를 발송하는 속도에 있어서 완벽을 추구하고 싶었다.**

   1-2. 현재 코드와 공정성에 대한 생각
   
   서버에서 클라이언트 세션에 퀴즈 데이터 메시지를 전송하는 코드는 다음과 같다.
   
   ```java
      private void sendMessageToAllSessions(TextMessage message) throws IOException {
        for (WebSocketSession session : sessions.values()) session.sendMessage(message);
    }
   ```

   위와 같이 for문을 사용해 순차적으로 세션에 퀴즈 데이터 메시지를 전송한다. **사용자가 늘어나면 늘어날 수록 가장 먼저 퀴즈를 받는 사용자와 가장 늦게 퀴즈를 받는 사용자 간의 차이는 커지게 된다.** 아래와 같이 동시 접속 사용자 수에 따라 테스트 코드와 그 결과를 작성해보았다. 현재 배포를 위해 사용하고 있는 AWS EC2 t2.micro 환경이라면 미리 대비할 필요가 있다고 생각했다.

   ```java
   @Test
    void testMessageSendTime() throws IOException, InterruptedException {
        sendMessages(sessions_200);
        sendMessages(sessions_1000);
        sendMessages(sessions_10000);
        sendMessages(sessions_30000);
        sendMessages(sessions_50000);
    }
    /*
    테스트 목적: 동일한 퀴즈 데이터 메시지를 세션에 전송할 때, 현재 순차적으로 전송하는데, 소요되는 시간을 측정
    테스트 결과:
        접속자 수 = 200:   0.0078525 seconds
        접속자 수 = 1000:  0.022955  seconds
        접속자 수 = 10000: 0.2079455 seconds
        접속자 수 = 30000: 0.624372  seconds
        접속자 수 = 50000: 1.0402312 seconds
     */
   ```

   1-2. 비동기 처리를 통한 개선
   
   아래의 코드와 같이 세션에 메시지를 전송하는 작업을 Runnable 인터페이스의 구현체로 정의했다. 그리고 스레드 풀을 생성하여 작업이 실행되도록 테스트 코드를 작성하였다.

```java
public class MessageSendingTask implements Runnable {
    private WebSocketSession session;
    private TextMessage textMessage;

    public MessageSendingTask(WebSocketSession session, TextMessage textMessage) {
        this.session = session;
        this.textMessage = textMessage;
    }

    @Override
    public void run() {
        try {
            session.sendMessage(textMessage);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}
```

```java
private void sendMessagesAsync(Map<String, WebSocketSession> sessions) throws IOException, InterruptedException {
        Set<String> sessionIds = sessions.keySet();
        TextMessage textMessage = new TextMessage("테스트 메시지");

        ExecutorService executor = Executors.newFixedThreadPool(2);
        for (String sessionId : sessionIds) {
            executor.submit(new MessageSendingTask(sessions.get(sessionId), textMessage));
        }
        executor.shutdown();
        executor.awaitTermination(5, TimeUnit.SECONDS);
    }

   /*
    테스트 목적: 순차적으로 전송하지 않고, 스레드풀을 사용해 비동기적으로 전송했을 때 시간 측정
    테스트 결과:
        접속자 수 = 200:   0.0078525 -> 0.0076665 seconds
        접속자 수 = 1000:  0.022955  -> 0.0120313 seconds
        접속자 수 = 10000: 0.2079455 -> 0.0828796 seconds
        접속자 수 = 30000: 0.624372  -> 0.2757126 seconds
        접속자 수 = 50000: 1.0402312 -> 0.4613047 seconds
     */
```

실제로 배포된 서버 환경에서의 테스트 결과는 아니지만, 위와 같이 최소 40%에서 최대 60%까지 시간차이가 단축되었다. 실제로 사용될 저용량의 서버 환경에서는 좀 더 유의미한 결과가 될 것이라고 생각한다. 아직 이 개선사항을 로직에 반영하지 않았으니, 아마 반영한다면 아래와 같이 반영할 것 같다. 

```java
@Component
@RequiredArgsConstructor
public class 동시접속자가 많은 퀴즈 핸들러 implements WebSocketHandler {
   private final ExecutorService executor = Executors.newFixedThreadPool(2); // 생성 반복을 문제마다 하지 않도록 final 선언 
   
   private void 퀴즈메시지 전송 메서드() {
      // 퀴즈 메시지 전송
      QuizDto quizDto = quizDataCenterMediator.getPresentQuizDto(category);
      TextMessage quizMessage = textMessageFactory.produceTextMessage(quizDto);
      for (String sessionId : sessionIds) {
         executor.submit(new MessageSendingTask(sessions.get(sessionId), textMessage));
      }
   }
}
```

   1-3. 추가로 공부해볼 내용
   
   스레드 풀의 크기를 늘릴 수록 좋은 것은 아니었다. 2로 했을 때 가장 효과가 좋았다. 왜 그럴까?

# Refactoring Logs

1. State Pattern을 적용한 리펙토링

   
   리펙토링 이전 코드는 아래와 같다. QuizDataCenter의 상태에 따라 수행해야할 동작이 다름을 인식하고 if 조건문을 사용하여 상태별로 수행할 로직을 다르게 구현하였다.

   
  ```java
   public void updateQuizDataCenterState(QuizDataCenterState nextState) {
        QuizDataCenterState presentState = getQuizDataCenterState();
        if (presentState == nextState) return;
        else if (presentState == ON_QUIZ & nextState == COMPLETED_QUIZ_GETTING_ANSWERED) {
            quizDataCenter.setState(nextState);
        } else if (presentState == COMPLETED_QUIZ_GETTING_ANSWERED & nextState == ON_SCORING) {
            quizDataCenter.setState(nextState);
            quizDataCenter.score(); // 수행 동작 1
            updateQuizDataCenterState(COMPLETED_SCORING);
        } else if (presentState == ON_SCORING & nextState == COMPLETED_SCORING) {
            quizDataCenter.setState(nextState);
        } else if (presentState == COMPLETED_SCORING & nextState == ON_QUIZ_SETTING) {
            quizDataCenter.setState(nextState);
            quizDataCenter.setNewQuizExcept(); // 수행 동작 2
        } else if (presentState == WAITING & nextState == ON_QUIZ_SETTING) {
            quizDataCenter.setState(nextState);
            quizDataCenter.initiateQuiz(); // 수행 동작 3
        } else if (presentState == ON_QUIZ_SETTING & nextState == ON_QUIZ) {
            quizDataCenter.setState(nextState);
        } else if (nextState == WAITING) {
            quizDataCenter.clearDataCenter(); // 수행 동작 4
        }
    }
```

  
  
추후에 상태를 좀 더 세분화하여 기능을 개발할 계획인데, 위와같은 상태에서는 새로운 상태가 추가될 때, 코드의 변화 가능성이 너무 크다고 생각했다.

그래서, 아래와 같이 DataCenterState를 인터페이스로 정의하고, 


```java
public interface DataCenterState {
    void handle(QuizDataCenter quizDataCenter);
}
```



상태 구현체를 정의하고 각가 수행할 로직을 다르게 구현했다.


```java
public class 대기상태 implements DataCenterState {
    @Override
    public void handle(QuizDataCenter quizDataCenter) {
        if (!(quizDataCenter.getPresentState() instanceof WAITING)) {
            quizDataCenter.setPresentState(this);
            quizDataCenter.clearDataCenter();
        }
        /*
        서버 시작 초기 대기상태의 경우 상태 변경, 수행할 로직 없음
        새 세션이 추가되면 INIT_QUIZ로 상태 업데이트

        세션이 존재하지 않을 경우 대기상태로 돌아가는 경우 초기화 로직 및 상태 업데이트
         */
    }
}
public class 새퀴즈세팅상태 implements DataCenterState{
    @Override
    public void handle(QuizDataCenter quizDataCenter) {
        quizDataCenter.setPresentState(this);
        quizDataCenter.initiateQuiz();
        /*
        상태 업데이트 없음
        생성된 퀴즈로 웹소켓에서 메시지를 전송 후 상태 업데이트
         */
    }
}

public class 퀴즈진행상태 implements DataCenterState {
    private LocalDateTime deadline;
    @Override
    public void handle(QuizDataCenter quizDataCenter) {
        if (!(quizDataCenter.getPresentState() instanceof ON_QUIZ)) {
            quizDataCenter.setPresentState(this);
        }
        if (deadline == null) {
            deadline = quizDataCenter.getPresentQuizFinishedAt().minusSeconds(2);
        }
        if (LocalDateTime.now().isAfter(deadline)) {
            quizDataCenter.setPresentState(new COMPLETE_QUIZ());
        }
    }
    /*
    현재 시각과 퀴즈세팅상태에서 설정된 퀴즈 종료 시각을 비교하여 퀴즈 종료 상태로 업데이트
    */
}

public class 퀴즈종료상태 implements DataCenterState {
    private LocalDateTime deadline;

    @Override
    public void handle(QuizDataCenter quizDataCenter) {
        if (deadline == null) {
            deadline = quizDataCenter.getPresentQuizFinishedAt().plusSeconds(2);
        }
        if (LocalDateTime.now().isAfter(deadline))
            quizDataCenter.setPresentState(new INIT_SCORE());
    }
    /*
    퀴즈가 종료되었으므로 퀴즈에 대한 답안을 일정시간동안 받는 상태 
    (이 상태에서 제출된 퀴즈 답안 객체을 유효하다고 판단하여 이후 채점 대상 객체가 됨)
    답안 수거 데드라인이 지난 뒤, 채점상태로 상태 변경
    */
}

public class 채점상태 implements DataCenterState {
    @Override
    public void handle(QuizDataCenter quizDataCenter) {
        quizDataCenter.score();
        quizDataCenter.setPresentState(new COMPLETE_SCORE());
    }
    /*
    제출된 답안 객체를 채점하는 동작 수행
    이후 채점완료상태로 상태 변경
    */
}

public class 채점완료상태 implements DataCenterState {
    @Override
    public void handle(QuizDataCenter quizDataCenter) {
    }
    /*
    DataCenter에서 채점완료상태에서는 수행할 로직이 없음
    대신, 웹소켓 핸들러에서 채점 결과를 메시지로 보낼 때 필요한 Flag 역할을 함
    */
}

public class 다음퀴즈세팅상태 implements DataCenterState {
    @Override
    public void handle(QuizDataCenter quizDataCenter) {
        quizDataCenter.setPresentState(this);
        quizDataCenter.setNewQuizExcept();
        /*
        새퀴즈상태와 다르게 이전 퀴즈를 제외하여 퀴즈를 세팅함
        바뀐 퀴즈로 웹소켓에서 메시지를 전송 후 상태 업데이트
         */
    }
}
```


  위와같이 상태 구현체들과 수행할 로직을 구현한 후, 상태별로 상태 객체를 호출하여 로직들을 수행할 수 있도록 코드를 변경하여 가독성과 유지보수성을 개선하였다. 

  ```java
public void updateDataCenterStateAndAction(DataCenterState dataCenterState) {
        quizDataCenter.setPresentState(dataCenterState);
        quizDataCenter.handle();
    }
```

  


# Trouble Shooting

1. ObjectMapper의 역직렬화 실패

  1-1. 에러 로그
       java.lang.IllegalArgumentException: Java 8 date/time type `java.time.LocalDateTime` not supported by default: add Module "com.fasterxml.jackson.datatype:jackson-datatype-jsr310" to enable handling at [Source: UNKNOWN; byte offset: #UNKNOWN]

주요 내용: 기본적으로 Java 8 date/time type `java.time.LocalDateTime`은 역직렬화를 지원하지 않으니 모듈을 추가하라고 함


  1-2. 원인 추적
        WebSocketHandler가 클라이언트로부터 수신한 데이터를 처리하는 과정에서 ObjectMapper가 메시지를 AnswerDto로 역직렬화를 수행하는데, AnswerDto에 LocalDateTime writtenAt 필드가 있어서 역직렬화에 실패하고 있었음.


  1-3. 해결
       ObjectMapper를 생성할 때, 아래와 같이 LocalDateTime 역직렬화를 지원하는 모듈을 추가하여 생성하여 해결
       
       ObjectMapper objectMapper = new ObjectMapper();
       objectMapper.registerModule(new JavaTimeModule());
   

2. 임의로 객체를 조회하는 코드 성능개선

   2-1. 개선 이전 코드
   ```java
   @Override
    public QuizContent getQuizContentExcept(Long presentQuizContentId, QuizCategory category) {
        long max = quizContentRepository.findMaxId();
        long min = quizContentRepository.findMinId();
        boolean found = false;

        while (!found) {
            Long randomId = new Random().nextLong(min, max);
            if (randomId.equals(presentQuizContentId)) continue;
            Optional<QuizContent> optional = quizContentRepository.findByIdAndCategory(randomId, category);
            if (optional.isPresent()) return optional.get();
        }
        return null;
    }

    @Override
    public QuizContent getRandomQuizContent(QuizCategory category) {
        long max = quizContentRepository.findMaxId();
        long min = quizContentRepository.findMinId();

        boolean found = false;
        int count = 0;
        while (!found) {
            Long randomId = new Random().nextLong(min, max + 1);
            Optional<QuizContent> optional = quizContentRepository.findByIdAndCategory(randomId, category);
            if (!optional.isEmpty()) return optional.get();
            count++;
            if (count > 10) found = true;
        }
        return null;
    }
    ```
   
   정말 부끄러운 코드라고 생각한다. 임의로 조회하기 위해서 최대 Id, 최소 Id를 조회한 후, JPQL로 그 범위 내의 임의의 값을 찾고 해당 id의 값을 가진 객체의 카테고리가 일치할 때까지 반복하는 과정을 거친다. 심지어 아래의 코드는 10번 이내에 발견하지 못하면 null을 반환한다. 핑계를 대자면, JPQL을 적극적으로 활용하려는 생각이 커서 JPQL로 랜덤 객체 1개를 조회하려고 했으나, 어떻게 JPQL을 작성해야할 지 몰라서 답답한 상황에 일단 마무리하려고 위와 같은 부끄러운 코드가 작성되었다.

   2-2. 개선 이후 코드

   ```java
   @Override
    public QuizContent getQuizContentExcept(Long presentQuizContentId, QuizCategory category) {
        return quizContentRepository.findRandomByIdNotAndCategory(presentQuizContentId, category.name()).orElseThrow(() -> new RuntimeException("NOT FOUND"));
    }

    @Override
    public QuizContent getRandomQuizContent(QuizCategory category) {
        return quizContentRepository.findRandomByCategory(category.name()).orElseThrow(() -> new RuntimeException("NOT FOUND"));
    }
   ```
   

   ```java
   @Repository
   public interface QuizContentRepository extends JpaRepository<QuizContent, Long> {
       @Query(value = "SELECT * FROM quiz_content qc WHERE qc.category = :category AND qc.id != :id ORDER BY RAND() LIMIT 1", nativeQuery = true)
       Optional<QuizContent> findRandomByIdNotAndCategory(@Param("id") Long id, @Param("category") String category);
       @Query(value = "SELECT * FROM quiz_content qc WHERE qc.category = :category ORDER BY RAND() LIMIT 1", nativeQuery = true)
       Optional<QuizContent> findRandomByCategory(@Param("category") String category);
   }
   ```
   
   이전과 다르게 JPQL을 사용하지 않고, Native Query를 사용했다. where문으로 id의 비교가 필요한 경우 실시하고, 랜덤으로 정렬한 후, Limit 1로 객체 1개만 조회하도록 하였다.
   
   이 코드를 작성하면서 특별히 어려움을 겪었던 부분이 하나가 있다. **바로, Category enum class 객체를 String으로 바꿔서 조회**를 해야한다는 것이다. 초기에 는 각 메서드의 매개변수에 String category가 아닌 QuizCategory category가 있었다.

   Entity를 정의할 때, category 컬럼에 enum 클래스 객체가 String 타입으로 저장되도록 설정하였으나, 처음에 QuizCategory category를 매개변수로 두고 테스트를 실행했을 때, 특정 category에 대해서는 작동이 되었다. 다른 category에 대해서는 아예 조회하지 못하는 결과로 이어졌다. 처음에 특정 cateogory에 대해서 작동이 되었기때문에 매개변수 타입을 String으로 바꿀 생각을 하는데까지 너무 늦게 걸렸다.

   추가로 왜 특정 category에 대해서는 작동이 되었는지 공부를 해볼 필요가 있다. 

   
---

# ERD
<div align="center">
  
![image](https://github.com/user-attachments/assets/30573d18-2dbd-42c9-ae04-e547ad1f2458)

</div>

---

# 서비스 이용 
<div align="center">

서비스 화면 및 카카오 로그인, 이름 바꾸기

![1-0 Trim](https://github.com/user-attachments/assets/09632357-905e-42e3-b679-ad4f5a8abb01)

<br>

실시간 퀴즈 진행

![1-2 - Trim](https://github.com/user-attachments/assets/52a07e11-26b5-4ec3-9e73-afaf962a38e9)

<br>

틀린 퀴즈 문제 조회

![2](https://github.com/user-attachments/assets/dc8f26e4-3a91-4b0e-8895-0ef0fbc209a2)

<br>

사용자 피드백 입력 및 Issue 등록 자동화

![3](https://github.com/user-attachments/assets/59f905d5-0ec7-4657-9732-bed8f39d5e36)


![image](https://github.com/user-attachments/assets/e797c78f-9fcf-413a-8635-1f9ccbd30216)


<br>

</div>
