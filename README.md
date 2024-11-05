# RankingQuizReadMe

아래의 링크로 서비스에 접속하실 수 있습니다. 

https://rankingquiz.rivercastleworks.site

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
