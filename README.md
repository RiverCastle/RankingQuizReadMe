# RankingQuizReadMe


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
   
   
