# 서비스 확장 2편(카카오 지도)
정적 웹 호스팅으로 서비스 하던 때 시도한 기능 확장

## 두번의 실패는 없다
친구를 통해 업계분들이 있는 카톡 방에서 사이트 건의사항에 대해 들은 적이 있다. 몇 번 문의 메일이 오기도 했던 내용인데, 지도 타입 중에 카카오 지도도 추가해 줄 수 있는지에 대한 건의사항이었다. 아마 카카오 지도가 업데이트 빈도가 조금 더 잦아서, 더 최신의 지도를 가져올 수 있기 때문일 것이다.
사실 1편에서 시도한 레이어 서비스 확장이 어떻게 보면 망한(?) 시도라서 이번에는 반드시 성공하겠다는 굳은 의지를 가지고 시작했다.
 
## 눈 좀 뜨세요 헤로쿠씨
카카오 지도는 정적 지도 api를 제공하지 않아서, 자바스크립트로 렌더링 된 페이지를 셀레니움으로 캡쳐해서 mapshot 서비스에 제공해주는 식으로 작동하게 만들었다. 자잘한 문제도 많긴 했는데, 제일 굵직한 문제가 3개 정도 있었다.

```
1. 30분 동안 요청이 없으면 서버 종료됨
2. 30초 내에 요청을 처리해야함(헤로쿠 자체 정책)
3. 위, 경도 값에 따라서 새 페이지 렌더링
```
먼저 첫 번째 문제부터 보자면,<strong>헤로쿠는 30분동안 요청이 없다면 서버를 종료시킨다</strong>. 이 정책 때문에 카카오 지도를 서버가 잠든 이후 처음 호출하는 사람은 굉장히 긴 시간을 기다려야 했다. 이 문제는 비교적 쉽게 해결되었다. 먼저 처음 사용한 해결책은 외부 서비스 이용이었다.
```
첫 번째 방법: kaffeine 서비스 이용
```
<a href="https://kaffeine.herokuapp.com/">kaffeine</a>이라는, 내 서버에 주기적으로 요청을 날려주는 서비스를 이용해봤는데, 서버가 종종 자는 모습이 보이길래 다른 방법을 찾아봤다.
```
두 번째 방법: 내가 내 서버 10분마다 호출(Scheduled)
```
어차피 요청만 날리면 되니까, 내가 내 서버를 계속 호출하게 해 놓으면 되지 않을까 싶어서 10분마다 루트 페이지를 계속해서 호출하게 작성했다. 근데 내 사이트 특성상 주말에는 접속이 거의 없는데, 쓸데없이 계속 켜놓는 기분이라 다른 방법을 다시 생각해냈다.

```
세 번째 방법: Mapshot 서비스 방문 시 서버 루트 페이지에 요청 날리기
```
현재(22.01.16)까지 사용되고 있는 방법이다. <strong>Mapshot 사이트에 접속과 동시에 서버에 요청을 날린다</strong>. 서버를 깨우는게 목적이기 때문에 응답 값은 중요하지 않다. 만약 카카오 지도 기능을 이용한다면 빠른 응답이 가능하고, 이용하지 않아도 서버는 지도 관련 작업은 아무것도 하지 않기 때문에 별 부담도 없다. 

## 대답없는 너
이제 남은 두 가지 문제가 있었다. 30초 내에 요청을 처리해야 되는데, 헤로쿠 서버 스펙이 좋은 편이 아니라서, 요청이 들어오는 대로 새 인스턴스를 만들면 들어오는 요청 전부가 뻗어버렸다. 그래서 결국 사용된 방법이 일종의 대기 시스템이었다.
사이트에서는 지속적으로 내가 <strong>사용 가능한 상태인지 지속적으로 요청을 보내고</strong>, 사용 가능하다는 응답이 온다면 그 때 지도 이미지 대기 상태로 들어갔다. 
``` javascript
// 앞단
var xml = new XmlHttpRequest();
xml.onreadystatechange = function() { 
    if(this.status == 200 && this.readyState == this.DONE) {
        if(xml.responseText == true){
            // 새로운 POST 요청 보내기  
        }
                  
    } else {
        // GET 요청 다시 보내기
    }
};
xml.open("GET", "serverUrl", true);
xml.send();
```
```java
// 서버단
@GetMapping
public ResponseEntity getMapping(){
    
    if(chromeDriverService.isAvailable()){
        return ResponseEntity.ok().body(true);
        // PostMapping으로 넘어감
    }

    return ResponseEntity.ok().body(false);
    // 주기적으로 Get요청, 가능여부 확인
}


@PostMapping
public ResponseEntity postMapping(){

    byte[] srcFile = chromeDriverService.getImage(kakaoMapInfo);

    if(srcFile != null){
        return ResponseEntity.ok().contentType(MediaType.IMAGE_JPEG).body(srcFile);
    }

    return ResponseEntity.badRequest().build();
}

```
하나씩 처리하는 속도는 비교적 빨랐기 때문에, 동시 접속자가 많지 않은 나의 서비스에는 부담이 없는 방식이었다. <strong>4명까지 동시에 요청해도 1,2분 이내로 모두 처리가 완료되었고</strong>, 당분간은 크게 무리 없이 서비스가 가능할 것 같다.
