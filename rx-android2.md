# RxAndroid로 반응형 앱 개발하기 #2

지난 연재에서 

![](http://image.chosun.com/sitedata/image/201410/29/2014102903089_0.jpg)

김용욱 dalinaum@gmail.com | GDG Korea Android 오거나이저, FeelingSunday CTO. React Native, 코틀린, Rust, RxJava 등 다양한 기술에 관심을 가지고 있다.

## 연재순서

 1. 안녕하세요. 반응형 프로그래밍
 2. 반응형 프로그래밍으로 사용자 경험 향상.
 3. 자유자재로 다루는 비동기 프로그래밍.

## ViewObservable.clicks

안드로이드에서 필수적인 옵저버블은 RxAndroid(https://github.com/ReactiveX/RxAndroid)에 구현되어 있다. 그 중 가장 유용한 도구는 `ViewObservable.clicks`이다.

````
ViewObservable
    .clicks(findViewById(R.id.button))
    .map(event -> new Random().nextInt())
    .subscribe(value -> {
        TextView textView = (TextView) findViewB yId(R.id.textView);
        textView.setText("number: " + value.toString());
    }, throwable -> {
        Log.e(TAG, "Error: " + throwable.getMessage());
        throwable.printStackTrace();
    });
````
<리스트 1> ViewObservable.clicks

`ViewObservable.clicks`는 `View` 타입을 인자로 받는 정적 메서드로 `setOnClickListener`를 통해 `OnClickListener`에 전달될 이벤트를 옵저버블 형태로 래핑한다. 대부분의 안드로이드 콤퍼넌트들이 `View` 타입을 상속받고 있기 때문에 이 정적 메서드를 사용하여 간단하게 클릭 이벤트를 처리할 수 있다.

<리스트 1>의 예제의 `map`은 전달 받은 `event` 값을 무시하고 랜덤 값으로 바꾼다. 기존 `event`가 담긴 옵저버블 대신 랜덤 값이 담긴 새로운 `observable`이 `subscribe`에 연결되며 버튼이 클릭될 때 마다 아이디가 `textView`인 `TextView`의 값이 임의의 숫자로 변경된다.

## Observable.merge

![](https://raw.githubusercontent.com/dalinaum/writing/master/merge.png)

<그림 1> Observable.merge의 흐름



사용자 인터페이스를 작성하다보면 여러 경로로 온 이벤트를 동시에 처리해야 하는 경우가 많다. 이런 경우 개별 이벤트를 옵저버블로 받은 후 병합을 시도할 수 있다.

````
Observable<String> lefts = ViewObservable.clicks(findViewById(R.id.leftButton))
        .map(event -> "left");

Observable<String> rights = ViewObservable.clicks(findViewById(R.id.rightButton))
        .map(event -> "right");

Observable<String> together = Observable.merge(lefts, rights);

together.subscribe(text -> ((TextView) findViewById(R.id.textView)).setText(text));

together.map(text -> text.toUpperCase())
        .subscribe(text -> Toast.makeText(this, text, Toast.LENGTH_SHORT).show());
````

두개의 `Button`에 대해 `ViewObservable.clicks`를 적용하여 이벤트를 리턴하는 `Observable`을 얻고 이를 `map`으로 가공하여 `Observable<String>`을 얻었다. 맵은 이벤트의 내용을 무시하고 이벤트가 발생한 시점에 `Observable`에 `left`와 `right`라는 문자열을 보낸다.


