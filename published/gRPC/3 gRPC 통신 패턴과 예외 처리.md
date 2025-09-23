# 🎯 3일차 학습 목표
1. gRPC 통신 패턴 4종 이해 및 실습
    - Unary RPC: 요청 1 <-> 응답 1
    - Server Streaming: 요청 1 <-> 응답 N
    - Client Streaming: 요청 N <-> 응답 1
    - Bidirectional Streaming: 요청 N <-> 응답 N
2. 에러 처리와 상태 코드
    - Status.INVALID_ARGUMENT, Status.NOT_FOUND, Status.UNAVAILABLE 같은 gRPC 표준 상태 코드 학습
    - 서버에서 클라이언트에 에러 던지기

----
# 1.  통신 패턴에 대해서
## 1-1. Unary RPC (요청1 <-> 응답1)
![Pasted image 20250923074810.png](image/Pasted%20image%2020250923074810.png)

가장 기본적인 형태의 통신 방식으로, HTTP REST에서 흔히 보이는 요청/응답 구조와 동일합니다.
클라이언트가 하나의 요청을 보내면, 서버가 하나의 응답을 돌려주는 방식입니다.

- 가장 많이 사용되는 패턴
- 단순한 함수 호출처럼 동작

주로 사용자 정보 조회나, 로그인 요청 등에 사용될 수 있습니다.

## 1-2. Server Streaming RPC (요청1 <-> 응답N)
![Pasted image 20250923080055.png](image/Pasted%20image%2020250923080055.png)

클라이언트가 요청을 한 번 보내면, 서버는 응답을 여러개 보내는 방식입니다.

- 서버에서 데이터를 점진적으로 전송
- REST에서는 동일 방식으로 통신하려면, long-polling, SSE(Server-Sent Evenets) 같은 복잡한 방식이 필요 했으나, gRPC에서는 기본 지원

주로 파일 다운로드(청크 단위), 실시간 뉴스 피드 등에 사용될 수 있습니다.

## 1-3. Client Streaming RPC (요청 N <-> 응답 1)
![Pasted image 20250923075841.png](image/Pasted%20image%2020250923075841.png)

클라이언트가 여러개의 요청 메세지를 순차적으로 서버에 보내게 됩니다.
서버에서는 요청을 모두 받은 뒤, 처리 후 최종적으로 하나의 응답을 클라이언트에 내려주게 됩니다.

- 서버는 요청 중간에 응답을 보내지 않음
- 클라이언트가 EOF(스트림 종료 신호)를 보내야 서버에서 응답 생성

주로 센서 데이터 업로드나, 여러 요청을 모아 하나의 결과 생성 등의 작업에 사용되는 방식입니다.

## 1-4. Bidirectional Streaming RPC (요청 N ↔ 응답 N)
![Pasted image 20250923085620.png](image/Pasted%20image%2020250923085620.png)

클라이언트와 서버가 동시에 여러 메세지를 자유롭게 주고 받을 수 있는 방식입니다.
양쪽 모두 스트림을 열고 원하는 만큼 `onNext()` 호출이 가능합니다.

- 동시 양방향 통신
- REST로는 구현하기 어렵지만, gRPC에서는 네이티브로 제공

주로 실시간 채팅이나 양방향 파일 동기화 등에 사용됩니다.

# 2. 통신 패턴 실습
## 2-1. Unary RPC
### 1) `.proto` 작성
```proto
rpc SayHello (HelloRequest) returns (HelloResponse);
```

### 2) 코드 생성
```shell
./gradlew generateProto
```

### 3) 구현체 작성
```java
@Override  
public void sayHello(HelloRequest request, StreamObserver<HelloResponse> responseObserver) {  
    final String message = request.getName() + "님 반갑습니다.";  
  
    final HelloResponse response = HelloResponse.newBuilder()  
        .setMessage(message)  
        .build();  
  
    responseObserver.onNext(response);  // 응답 전달  
    responseObserver.onCompleted();     // 스트림 종료  
}
```

### 4) 호출
```shell
grpcurl -plaintext -d '{"name":"zinzo"}' localhost:9097 UnaryService/CallUnary
```
```json
{
  "message": "zinzo님 반갑습니다."
}
```

## 2-2. Server Streaming RPC
### 1) .proto 작성
```
rpc ListGreetings (HelloRequest) returns (stream HelloResponse);
```
N개의 응답을 받기 위해 response 타입 앞에 `stream` 명시 해줍니다.

### 2) 코드 생성
```shell
./gradlew generateProto
```

### 3) 구현체 작성
```java
@Override  
public void listGreetings(HelloRequest request, StreamObserver<HelloResponse> responseObserver) {  
    final String name = request.getName();  
  
    for (int i = 1; i <= 3; i++) {  
        HelloResponse chunk = HelloResponse.newBuilder()  
            .setMessage("[chunk " + i + "] 안녕하세요, " + name + "님")  
            .build();  
        responseObserver.onNext(chunk);  // 응답 조각 전송  
    }  
  
    responseObserver.onCompleted();       // 스트림 종료  
}
```

### 4) 호출
```shell
grpcurl -plaintext -d '{"name":"zinzo"}' localhost:9097 HelloService/ListGreetings
```
```json
{
  "message": "[chunk 1] 안녕하세요, zinzo님"
}
{
  "message": "[chunk 2] 안녕하세요, zinzo님"
}
{
  "message": "[chunk 3] 안녕하세요, zinzo님"
}
```

## 2-3. Client Streaming RPC
### 1) .proto 작성
```  
rpc UploadGreetings (stream HelloRequest) returns (HelloResponse);
```
요청을 여러개로 보내기 위해 요청 타입 앞에 `stream` 키워드를 명시해줍니다.
### 2) 코드 생성
```shell
./gradlew generateProto
```

### 3) 구현체 작성
```java
@Override  
public StreamObserver<HelloRequest> uploadGreetings(StreamObserver<HelloResponse> responseObserver) {  
    StringBuilder acc = new StringBuilder();  
  
    return new StreamObserver<>() {  
  
        @Override  
        public void onNext(HelloRequest helloRequest) {  
            final String name = helloRequest.getName();  
  
            if (!acc.isEmpty()) {  
                acc.append(", ");  
            }  
  
            acc.append("안녕하세요, ")  
                .append(name)  
                .append("님");  
        }  
  
        @Override  
        public void onError(Throwable throwable) {  
            // 클라이언트 측 스트림 에러(취소/타임아웃 등). 필요 시 로깅.  
        }  
  
        @Override  
        public void onCompleted() {  
            final HelloResponse response = HelloResponse.newBuilder()  
                .setMessage(acc.isEmpty() ? "입력 없음" : acc.toString())  
                .build();  
  
            responseObserver.onNext(response);  
            responseObserver.onCompleted();  
        }  
    };
```

### 4) 호출
```shell
 cat <<'EOF' | grpcurl -plaintext -d @ \
  localhost:9097 HelloService/UploadGreetings
{"name":"zinzo"}
{"name":"beng"}
{"name":"bengZin"}
EOF
```
```json
{
  "message": "안녕하세요, zinzo님, 안녕하세요, beng님, 안녕하세요, bengZin님"
}
```

## 2-4. Bidirectional Streaming RPC

### 1) .proto 작성
```
rpc Chat (stream HelloRequest) returns (stream HelloResponse);
```
양방향 통신을 위해 요청과 응답 파라미터 앞에 `stream` 키워드를 명시해줍니다.
### 2) 코드 생성
```shell
./gradlew generateProto
```
### 3) 구현체 작성
```java
@Override  
public StreamObserver<HelloRequest> chat(StreamObserver<HelloResponse> responseObserver) {  
    return new StreamObserver<>() {  
        @Override  
        public void onNext(HelloRequest helloRequest) {  
            // 요청 1개 들어올 때마다 즉시 응답 1개 흘려보내기 (에코)  
            final String message = "[bidi] 안녕하세요, " + helloRequest.getName() + "님";  
  
            final HelloResponse response = HelloResponse.newBuilder()  
                .setMessage(message)  
                .build();  
  
            responseObserver.onNext(response);  
        }  
  
        @Override  
        public void onError(Throwable throwable) {  
            // 클라이언트 측 스트림 에러(취소/타임아웃 등). 필요 시 로깅.  
        }  
  
        @Override  
        public void onCompleted() {  
            responseObserver.onCompleted();  
        }  
    };  
}
```

### 4) 호출
```shell
grpcurl -plaintext -d @ \
localhost:9097 HelloService/Chat <<'EOF'
{"name":"one"}
{"name":"two"}
EOF
```
```json
{
  "message": "[bidi] 안녕하세요, one님"
}
{
  "message": "[bidi] 안녕하세요, two님"
}
```

# 3. gRPC 에러 처리
- gRPC에서는 HTTP REST와 다르게, HTTP Status Code가 아닌 **gRPC 전용 Status**를 사용합니다.
- 서버는 `onError(Throwable)`로 에러를 전달하고, 클라이언트는 `StatusRuntimeException` 등으로 받게 됩니다.
- 에러에 대한 설명과 원인(cause), 메타데이터(trailers), 에러 디테일(google.rpc.Status) 까지 담아 보낼 수 있습니다.

## 3-1. 자주 쓰이는 상태 코드
| **코드**              | 상수  | **의미**   | 상황                   |
| ------------------- | --- | -------- | -------------------- |
| CANCELLED           | 1   | 호출 취소    | 클라이언트 취소/연결 끊김       |
| INVALID_ARGUMENT    | 3   | 잘못된 입력   | 필드 검증 실패, 포맷 오류      |
| DEADLINE_EXCEEDED   | 4   | 시간 초과    | 타임아웃 경과              |
| NOT_FOUND           | 5   | 리소스 없음   | ID로 조회했지만 없음         |
| ALREADY_EXISTS      | 6   | 중복       | 고유 키 중복(이메일 등)       |
| PERMISSION_DENIED   | 7   | 권한 없음    | 인증은 됐지만 권한 부족        |
| RESOURCE_EXHAUSTED  | 8   | 한도 초과    | rate limit, quota 초과 |
| FAILED_PRECONDITION | 9   | 선행조건 불충족 | 상태 전이 불가 등           |
| ABORTED             | 10  | 동시성 충돌   | 낙관적 락 실패 등           |
| UNAVAILABLE         | 14  | 일시적 장애   | 의존 서비스 다운/재시도 요망     |
| INTERNAL            | 13  | 서버 내부 오류 | 예상 못한 예외             |
| UNAUTHENTICATED     | 16  | 인증 안 됨   | 토큰 없음/무효             |
불확실한 상태의 에러 상황일 때, `INTERNAL` 보다는 **의미 있는 코드**로 매핑하는 것이 중요합니다.

## 3-2. 에러 응답하기
### 1) 가장 기본 방식, Status + 설명
```java
@Override  
public void sayHello(HelloRequest request, StreamObserver<HelloResponse> responseObserver) {  
    if (request.getName() == null) {  
        responseObserver.onError(  
            io.grpc.Status.INVALID_ARGUMENT  
                .withDescription("이름 값은 필수입니다.")  
                .asRuntimeException()  
        );  
    }
    
	...
}
```
- withDescription를 통해 설명을 추가 해줍니다.

##### 호출
```
grpcurl -plaintext -d '{}' localhost:9097 HelloService/SayHello
```

##### 결과
```
ERROR:
  Code: InvalidArgument
  Message: 이름 값은 필수입니다.
```

### 2) 리치 에러 (google.rpc.Status + Any details)
표준화된 구조로 디테일 전달이 가능하고, 클라이언트에서 파싱이 가능한 형태입니다.
```java
@Override  
public void sayHello(HelloRequest request, StreamObserver<HelloResponse> responseObserver) {  
    if (!StringUtils.hasText(request.getName())) {  
        ErrorInfo info = ErrorInfo.newBuilder()  
            .setReason("VALIDATION_FAILED")  
            .putMetadata("field", "name")  
            .build();  
  
        Status rich = Status.newBuilder()  
            .setCode(io.grpc.Status.INVALID_ARGUMENT.getCode().value())  
            .setMessage("이름 값은 필수입니다.")  
            .addDetails(Any.pack(info))  
            .build();  
  
        responseObserver.onError(StatusProto.toStatusException(rich));  
    }
    
    ...
}
```

##### 결과
```
ERROR:
  Code: InvalidArgument
  Message: 이름 값은 필수입니다.
  Details:
  1)	{
    	  "@error": "google.rpc.ErrorInfo is not recognized; see @value for raw binary message data",
    	  "@type": "type.googleapis.com/google.rpc.ErrorInfo",
    	  "@value": "ChFWQUxJREFUSU9OX0ZBSUxFRBoNCgVmaWVsZBIEbmFtZQ=="
    	}
```

### 3. @GrpcAdvice
기존 REST 형식에서도 사용했던 Advice 형식처럼 Grpc에서도 분리하여 공통으로 처리해줄 수 있습니다.
```java
@GrpcAdvice  
class GlobalGrpcAdivce {  
  
    @GrpcExceptionHandler(GrpcException.class)  
    public StatusRuntimeException handleGrpc(GrpcException e) {  
        return Status.INVALID_ARGUMENT  
            .withDescription(e.getMessage())  
            .asRuntimeException();  
    }  
}
```

```java
@Override  
public void sayHello(HelloRequest request, StreamObserver<HelloResponse> responseObserver) {  
    if (!StringUtils.hasText(request.getName())) {  
        throw new GrpcException("오류 발생");  
    }
	...
}
```

##### 결과
```
ERROR:
  Code: InvalidArgument
  Message: 오류 발생
```

----

# 느낀점
기존 REST 방식으로 처리 했다면 구현 방식이 어려웠을 것도, gRPC로 처리 하니 쉽고 가독성 좋게 처리할 수 있다는 점이 놀라웠던 것 같습니다.

예외 처리 방식은 처음에 좀 어렵지 않을까 싶었는데,
`Advice` 형식으로도 제공되고 있어서 기존 방식과 크게 다르지 않게 처리할 수 있는 점도 좋았던 것 같습니다.

`@Valid`랑도 같이 섞어 사용해볼 수 있지 않을까 싶었는데, gRPC는 별도 서드파티 플러그인 같은 것들로 처리할 수 있다고 합니다.

이 부분도 시간이 나면 같이 적용해보면 좋을 것 같습니다.