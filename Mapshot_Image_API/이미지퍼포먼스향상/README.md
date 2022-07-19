# 이미지 캡쳐 퍼포먼스 향상
## 이슈
유저가 요청한 이미지 결과물을 받는데 시간이 많이 걸림

## 원인
이미지 용량이 너무 커서 이미지 렌더링 후 데이터 추출에 많은 시간이 소요

## 해결
1. 이미지 확장자 변경을 통한 용량 간소화
```java
public class ChromeDriverExtends extends ChromeDriver {

    /// ...
    public byte[] getScreenshot() {
        Object result = sendCommand("Page.captureScreenshot", ImmutableMap.of("format", "jpeg"));
        String data = ((Map<String, String>)result).get("data");

        return Base64.getDecoder().decode(data);
    }

    protected Object sendCommand(String cmd, Object params) {
        return execute("sendCommand", ImmutableMap.of("cmd", cmd, "params", params)).getValue();
    }

    /// ...

}
```

내가 써본 모든 크롤링 라이브러리들은 이미지 캡쳐를 지원함과 동시에, 결과물은 PNG 형식 밖에 지원하지 않는다. PNG로 사진 제작 시 용량도 무지막지하게 커지고 처리 속도가 거의 달팽이가 되는 바람에, 다른 방법을 찾아야했다. 

다행히도, 크롬 드라이버는 DevTool 문서에 JPEG로 스크린샷을 캡쳐하는 법이 적혀있었고, 기존의 코드를 적절히 조합해서 이를 완성시킬 수 있었다.

2. 이미지를 분할 캡쳐를 통한 용량 간소화

```java
// 너무 잘게 분할했을 경우
chromeDrvier.setWindowSize(500);

for(int y = 0; y < 10; y++){
    for(int x = 0;x < 10; x++){
        driverService.scrollPage(x, y);
        driverService.capture();
    }
    
}
// 전체 스크린샷 캡쳐보다 오히려 느림



// 적당한 크기로 분할했을 경우
chromeDrvier.setWindowSize(1000);

for(int y = 0; y < 5; y++){
    for(int x = 0;x < 5; x++){
        driverService.scrollPage(x, y);
        driverService.capture();
    }
    
}
// 전체 캡쳐보다 빨라짐
```
이미지 분할 캡쳐는 약간 독특했는데, 분할 단위가 특정 값 근처에서 결과가 확 뒤바뀌었다.


사진을 <strong>너무 잘게 쪼개는 것이 아닌, 적당히 크게 쪼개는 순간 처리 속도가 꽤나 향상</strong>되었던 것이었다. 몇 가지 값들을 테스트 해본 결과 1000 X 1000 으로 사진을 나누는 것이 효율이 높은 것을 확인하였고, 적용하였다.

## 배운점
- 문제 해결 방법은 다양하다
- 지금의 방식은 오늘의 최선일 뿐이다
    - 이 방식이 충분한 퍼포먼스를 보인다고 할 수 있을까?
    - 항상 일정한 유저층만 사용해서 감당 가능한건 아닐까?
    - 만약 카톡, 배민같이 트래픽이 퍼붓는 사이트였다면?
    - 당장 그런 트래픽이 내일 당장 쏟아진다면 이런 코드로 대처가 가능할까?
- png는 언제 사용하는 걸까
    - 예전에 레이어 관련 서비스 추가할 때 사용해봄
    - 내 기억으로는 배경화면 투명처리?누끼따기?를 지원했음
    - 이미지를 겹쳐서 투영할 때 유용한 확장자로 기억이 남
- png가 용량이 큰 이유
    - 무손실 압축 포맷이라고 한다
    - jpg는 손실 압축 포맷
    - webp라는 개선된 포맷도 있음
        - 근데 내 서버에서는 이미지 데이터 추출이 너무 느림
        - 아무리 용량이 작게 나와도 수지타산이 안맞음 
    
    
