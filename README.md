# RankingQuizReadMe

아래의 링크로 서비스에 접속하실 수 있습니다. 

https://rankingquiz.rivercastleworks.site

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
