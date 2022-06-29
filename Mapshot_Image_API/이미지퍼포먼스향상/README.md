# 이미지 캡쳐 퍼포먼스 향상
## Too Slow
서버에서 제작하는 위성 이미지(카카오, 구글)는 가공하는데 상당한 시간이 소요된다.
가로, 세로 픽셀 값이 기본적으로 5000은 넘기 때문인데, 이 시간을 줄이기 위해 상당히 많은 노력을 계속해서 기울였다.
도전한 방법들을 나열하면 다음과 같다.

- 웹 드라이버 변경해보기
    - 크롬 드라이버 (O)
    - 게코 드라이버
- 크롤링 라이브러리 변경해보기
    - selenium (O)
    - puppeteer
    - playwright
    - phantomJS 
- WAS 변경
    - undertow (O) 
    - tomcat
- 이미지 확장자 변경
    - JPEG (O)
    - WEBP
    - PNG
- 캡쳐 방식
    - 분할 캡쳐 (O)
    - 전체 캡쳐

동그라미 친 것들이 현재 채택하고 있는 방식들인데,
이 중에서 가장 효과적이었던 것은 <strong>이미지 확장자 변경</strong>이었고, 
발견하기 힘들었던 것이 <strong>이미지 분할 캡쳐</strong>였다.

## JPG 지원 좀 해주세요
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
내가 써본 모든 크롤링 라이브러리들은 이미지 캡쳐를 지원함과 동시에, 결과물은 PNG 형식 밖에 지원하지 않는다. 물론 PNG로 하면 좋긴 한데, 용량도 무지막지하게 커지고 처리 속도가 거의 달팽이가 되는 바람에, 다른 방법을 찾아야했다. 다행히도, 크롬 드라이버는 DevTool 문서에 JPEG로 스크린샷을 캡쳐하는 법이 적혀있었고, 기존의 코드를 적절히 조합해서 이를 완성시킬 수 있었다.


## 분할, 적당히가 제일 어렵다
```java
// 전체 캡쳐보다 느림
for(int i = 0; i < 10; i++){
    driverService.capture();
}

// 전체 캡쳐보다 빨라짐
for(int i = 0; i < 5; i++){
    driverService.capture();
}
```

사실 이미지 분할 캡쳐는 이전에 이미 시도해 보았던 옵션이었다.
하지만 오히려 전체 이미지 캡쳐보다 성능이 훨씬 늘어져서 사용하지 않았었다.
그런데 어느 날, 다시 한 번 저 옵션을 시도해 보았고, 한 가지 사실을 발견하게 되었다.
사진을 <strong>너무 잘게 쪼개는 것이 아닌, 적당히 크게 쪼개는 순간 처리 속도가 꽤나 향상</strong>되었던 것이었다. 몇 가지 값들을 테스트 해본 결과 1000 X 1000 으로 사진을 나누는 것이 효율이 높았고, 이에 따른 몇 가지의 코드를 추가하며 마무리하였다.

