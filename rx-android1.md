# RxAndroid로 반응형 앱 개발하기 #1

애플리케이션 개발이 쉽다는 말은 점차 옛말이 되어가고 있다. 유즈케이스가 다양해진 만큼 입력 방식도 다양해졌다. 가공이 필요한 데이터는 다양한 방식으로 비동기적으로 전달된다. 데이터는 사용자에게 즉시 전달 가능한 것과 적절히 프로세싱을 거쳐야 하는 것으로 나누어진다. 복잡한 요구사항을 만족하기 위해 서버와 클라이언트도 복잡해졌다. 오늘날 서버와 클라이언트 코드는 복잡한 제어흐름, 콜백, 상태변수, 이벤트 버스, 대기열, 다양한 디자인 패턴 요소들로 가득하다. 여러 산발적인 이슈를 관리할 적절한 도구가 절실한 시점이다.

![](http://image.chosun.com/sitedata/image/201410/29/2014102903089_0.jpg)

김용욱 dalinaum@gmail.com | GDG Korea Android 오거나이저. 영화를 광적으로 좋아하는 안드로이드 개발자로 최근에는 Reactive Programming에 빠져 RxJava, RxAndroid에 매진하고 있다.

## 연재순서

 1. 안녕하세요. 반응형 프로그래밍
 2. 반응형 프로그래밍으로 사용자 경험 향상.
 3. 자유자재로 다루는 비동기 프로그래밍.

여러 이슈를 처리해줄 적절한 도구는 닷넷 진영에서 등장했다. 마이크로소프트는 옵저버 패턴과 LINQ 스타일 문법을 확장하여 비동기처리와 이벤트 기반 프로그래밍을 할 수 있다는 것을 발견한다. 연구진은 이를 정립하여 반응형 확장(Rx, Reactive Extensions)을 공개하였다. (https://msdn.microsoft.com/en-us/data/gg577609)

반응성 확장은 곧 여러 기술 기반 회사들의 호응을 얻었다. 넷플릭스(Netflix)는 Rx를 자바(RxJava) 환경에 옮겼고, 사운드클라우드(SoundColud)는 RxJava를 안드로이드까지 (RxAndroid) 확장시켰다. 깃헙(Github)은 Rx가 아이폰과 맥(RxCocoa)에서 작동되도록 만들었다.

여러 기술 업체가 Rx를 미는 분위기 속에 안드로이드 오픈소스의 락스타 제이크 와튼(Jake Wharton)은 본인의 트위터에 <리스트 1>의 문장을 남겨 논란을 일으켰다. 반응형 프로그래밍을 한다면 안드로이드 플랫폼 내의 많은 요소들과 여러 라이브러리들이 더 이상 필요가 없다는 이야기다.

````
"Using RxJava to replace loaders and internal lib to replace fragments/activities. All that's left is views, android.animation.*, and bliss."
by Jake Wharton
````
<리스트 1> 제이크 와튼의 트위터 (https://twitter.com/jakewharton/status/385898996884971520)

물론 세상에는 은탄환은 없고 반응형 프로그래밍 역시 장단점은 존재할 것이다. 이 말은 RxAndroid(RxJava)가 그가 다른 대안을 알아보고 싶지 않을 정도로 강력하고 유연하다는 의미로 받아들일 수 있을 것이다.

## Hello RxAndroid

안드로이드에서 리액티브 프로그래밍을 사용하기 위해서는 먼저 Gradle 환경 설정을 해야한다. 안드로이드 스튜디오서 프로젝트를 만들고 `app` 디렉토리의 `build.gradle` 파일을 열자.

````
apply plugin: 'com.android.application'

android {
    compileSdkVersion 21
    buildToolsVersion "21.1.2"

    defaultConfig {
        applicationId "rx.android.gdg.kr.simpleobservable"
        minSdkVersion 11
        targetSdkVersion 21
        versionCode 1
        versionName "1.0"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    compile 'com.android.support:appcompat-v7:21.0.3'
    compile 'io.reactivex:rxandroid:0.24.0'
}
````
<리스트 2> app/build.gradle 설정파일

그래들 설정에서 의존성 부분이 중요하다.

````
dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    compile 'com.android.support:appcompat-v7:21.0.3'
    compile 'io.reactivex:rxandroid:0.24.0'
}
````
<리스트 3> 의존성 설정

두가지 외부 의존성이 있는 것을 볼 수 있다. `appcompat-v7`와 `rxandroid`이다. 안드로이드도 자바 환경이기 때문에 `rxjava`를 포함하지 않는 것에 의아할 수 있다. RxAndorid는 RxJava에 대한 의존성을 가지고 있고 RxAndroid를 의존성에 포함하면 안드로이드 개발 환경에 문제가 없다.

이제 첫 번째 반응형 안드로이드 앱 코드를 만들어보자.

````
package rx.android.gdg.kr.simpleobservable;

import android.os.Bundle;
import android.support.v7.app.ActionBarActivity;
import android.util.Log;
import android.widget.TextView;

import rx.Observable;
import rx.Subscriber;


public class MainActivity extends ActionBarActivity {
    private static final String TAG = MainActivity.class.getName();

    @Override
    protected void onCreate(Bundle savedInstanceState) {

        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Observable<String> simpleObservable =
                Observable.create(new Observable.OnSubscribe<String>() {
                    @Override
                    public void call(Subscriber<? super String> subscriber) {
                        subscriber.onNext("Hello RxAndroid !!");
                        subscriber.onCompleted();
                    }
                });

        simpleObservable
                .subscribe(new Subscriber<String>() {
                    @Override
                    public void onCompleted() {
                        Log.d(TAG, "complete!");
                    }

                    @Override
                    public void onError(Throwable e) {
                        Log.e(TAG, "error: " + e.getMessage());
                    }

                    @Override
                    public void onNext(String text) {
                        ((TextView) findViewById(R.id.textView)).setText(text);
                    }
                });
    }
}
````
<리스트 4> 반응형 Hello World

<리스트 4>의 코드에서 `Observable`과 `Subscriber`를 주목하라. 데이터의 강을 만드는 옵저버블(Observable)과 강에서 그것을 하나씩 건지는 서브스크라이버(Subscriber)가 반응형 프로그래밍의 가장 핵심적인 요소이다.

옵저버블은 데이터를 제공하는 생산자로 세가지 유형의 행동을 한다.

````
 1. onNext - 새로운 데이터를 전달한다.
 2. onCompleted - 스트림의 종료.
 3. onError - 에러 신호를 전달한다
````
<리스트 5> 옵저버블의 3가지 행동

이제 다시 옵저버블 코드를 살펴보자.

````
Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        subscriber.onNext("Hello RxAndroid !!");
        subscriber.onCompleted();
    }
});
````
<리스트 6> Hello World 옵저버블

<리스트 6>의 옵저버블은 `Hello RxAndroid !!`란 데이터를 전달하고(onNext) 이내 끝났다는 신호(onCompleted)를 전달한다.

옵저버블의 세가지 행동들이 만드는 스트림을 도식해 보면 <그림 1>과 같은 모양이 된다.

![](https://raw.githubusercontent.com/dalinaum/writing/master/observable-flow.png)
<그림 1> 옵저버블 액션 흐름

<그림 1> 상단의 흐름은 세번 데이터를 전달받고(onNext) 정상 종료(onCompleted)인 경우이고 하단의 흐름은 두번 데이터를 전달받고(onNext) 에러가 발생(onError)한 경우다.

서브스크라이버는 옵저버블이 만드는 스트림에 응대하여 처리할 수 있게 대칭적으로 인터페이스가 구성되어 있다.

````
simpleObservable
    .subscribe(new Subscriber<String>() {
        @Override
        public void onCompleted() {
            Log.d(TAG, "complete!");
        }

        @Override
        public void onError(Throwable e) {
            Log.e(TAG, "error: " + e.getMessage());
        }

        @Override
        public void onNext(String text) {
            ((TextView) findViewById(R.id.textView)).setText(text);
        }
    });
````
<리스트 7> 서브스크라이버 예

`Subscriber`는 3가지 메서드를 오버라이드하도록 구성되어 있고 개별 메서드의 역할은 옵저버블의 해당 메서드와 동일하다. 스트림의 데이터의 유형별로 대칭되는 서브스크라이버의 인터페이스가 대응하는 것이다. <리스트 7>의 구성은 정상적으로 데이터가 왔을 때 텍스트 뷰의 항목을 수정하고 종료나 에러가 발생할 때 로그를 남긴다.

## 편의를 위한 단출한 서브스크라이버

서브스크라이버를 구성할 때 항상 `onCompleted`, `onError`, `onNext`를 다루는 것은 불편하다. 편의를 위해 사용하지 않는 구성이 누락된 인터페이스가 제공된다.

````
    simpleObservable
        .subscribe(new Action1<String>() {
            @Override
            public void call(String text) {
                ((TextView) findViewById(R.id.textView)).setText(text);
            }
        });
````
<리스트 8> 단출한 서브스크라이버

<리스트 8>의 서브스크라이버는 `onNext`의 경우만 다루고 있다. `onNext`, `onError`를 다루는 것과 `onNext`, `onError`, `onCompleted`를 모두 다루는 구성이 준비되어 있다.

````
simpleObservable
    .subscribe(new Action1<String>() {
        @Override
        public void call(String text) {
            ((TextView) findViewById(R.id.textView)).setText(text);
        }
    }, new Action1<Throwable>() {
        @Override
        public void call(Throwable throwable) {

        }
    }, new Action0() {
        @Override
        public void call() {

        }
    });
````
<리스트 9> `onNext`, `onError`, `onCompleted`를 모두 다루는 구성

````
simpleObservable
    .subscribe(new Action1<String>() {
        @Override
        public void call(String text) {
            ((TextView) findViewById(R.id.textView)).setText(text);
        }
    }, new Action1<Throwable>() {
        @Override
        public void call(Throwable throwable) {

        }
    });
````
<리스트 10> `onNext`, `onError`를 다루는 구성

## 데이터 가공 Map

![](https://raw.githubusercontent.com/dalinaum/writing/master/map.png)
<그림 2> Map의 스트림 흐름

map은 한 데이터를 다른 데이터로 바꾸는 `오퍼레이터`이다. 원본의 데이터는 변경하지 않고 새로운 스트림을 만들어 낸다. <그림 2>는 스트림의 데이터를 각각 10 씩 곱을 하는 예이다.

이렇게 map 사용하려면 인자 하나를 받아 값을 10배를 곱해 반환하는 메서드를 map 전달해야 한다.

````
.map(new Func1<Integer, Integer>() {
    @Override
    public Integer call(Integer value) {
        return value * 10;
    }
})
````
<리스트 11> 값을 10배 씩 구하는 map

RxAndroid를 사용하여 앱을 개발할 때 미리 준비된 다양한 RxJava 오퍼레이터를 사용할 수 있다. (http://reactivex.io/documentation/operators.html) 반응형 프로그래밍에서 오퍼레이터는 기존에 작성된 메서드를 재사용하여 전체 값을 변경할 수 있는 도구를 제공한다. 값을 재사용 가능한 함수를 이용해서 가공한다는 부분에서 함수형 프로그래밍과 일맥상통하는 부분이 있다.

문자열을 대문자로 바꾸는 것도 쉽게 구현할 수 있다.

````
simpleObservable
    .map(new Func1<String, String>() {
        @Override
        public String call(String text) {
            return text.toUpperCase();
        }
    })
    .subscribe(new Action1<String>() {
        @Override
        public void call(String text) {
            ((TextView) findViewById(R.id.textView)).setText(text);
        }
    });
````
<리스트 12> 문자열을 대문자로 바꾸는 map

<리스트 12>는 옵저버블의 스트림을 대문자로 변환하는 (사실은 새로운 스트림을 만드는) map 호출하고 그것을 서브스크라이버에 연결하는 것을 보여준다.

map 이용할 때는 같은 타입으로만 변경해야 할 수 있는 것은 아니다.

````
simpleObservable
    .map(new Func1<String, Integer>() {
        @Override
        public Integer call(String text) {
            return text.length();
        }
    })
    .subscribe(new Action1<Integer>() {
        @Override
        public void call(Integer length) {
            ((TextView) findViewById(R.id.textView)).setText("length: " + length);
        }
    });
````
<리스트 13> `String` 스트림을 `Integer` 스트림으로 변환하는 map이다.

## 단일 데이터, 컬렉션을 옵저버블 생성을 위한 유틸리티

`simpleObservable`은 문자열 하나만 전달하는 간단한 옵저버블인데 옵저버블을 쉽게 만들 수 있게 도와주는 유틸리티 메서드가 있다.

````
Observable<String> simpleObservable = Observable.just("Hello RxAndroid");
````
<리스트 14> 단일 데이터로 옵저버블을 생성하는 유틸리티 `Observable.just`

연속적인 데이터를 처리하기 위한 유틸리티 `Observable.from`도 있다. 이 유틸리티를 이용하면 배열이나 컬렉션을 보다 간편하게 처리할 수 있다.

## 번잡한 문법을 람다로 간단히

기존의 자바 문법에서 RxAndroid (RxJava) 프로그래밍을 할 때 로직과 상관없는 요소들이 많이 발견된다. 어노테이션, 클래스 선언, 메서드 선언 등의 요소들이 코드에서 반복된다. 이런 반복들은 자바 8에서 도입된 람다를 사용하면 손 쉽게 제거할 수 있고 코드가 간결해 가독성이 높아진다.

````
Observable<String> simpleObservable = Observable.just("Hello Lambda!!");
simpleObservable
    .map(text -> text.length())
    .subscribe(
        length -> ((TextView) findViewById(R.id.textView)).setText("length: " + length));
````
<리스트 15> 람다로 간략화 시킨 코드

자바 코드가 매우 간결해진 것을 볼 수 있다. 람다 문법이 생소한 독자를 위해 코드 단위로 비교하며 설명한다.

````
.map(new Func1<String, Integer>() {
        @Override
        public Integer call(String text) {
            return text.length();
        }
    })
````
<리스트 16> 람다를 적용하기 전 map 코드

람다는 단일 메서드를 가지고 있는 경우만 간략화 할 수 있다. 여기에서 메서드 인자 부분 이전의 번잡한 내용을 지워보자.

````
.map((String text) -> {
        return text.length();
    })
````
<리스트 17> 익명 객체 부분을 제거한 람다 map 코드

인자의 타입을 추론할 수 있다면 생략할 수 있다.

````
.map((text) -> {
        return text.length();
    })
````
<리스트 18> 익명 객체 부분과 타입을 제거한 람다 map 코드

인자가 하나인 경우에는 괄호를 생략할 수 있다.

````
.map(text -> text.length())
````
<리스트 19> 블록을 생략한 람다 map 코드

람다 블록 내에 문장이 하나이고 리턴값을 가진다면 블록, 세미콜론, 리턴 키워드를 지울 수 있다.

람다의 단 한가지 단점은 자바 8이상에서만 쓸 수 있다는 점이다. 안드로이드는 자바 8을 지원하지 못하며 람다를 쓰기 위해서는 이전 버전에서 람다를 쓸 수 있게 하는 포팅 라이브러리를 사용해야 한다.

![](https://raw.githubusercontent.com/dalinaum/writing/master/retrolambda.png)
<그림 3ㅎ> 자바 8 람다 코드를 자바 7 바이트 코드로 번역하는 레트로 람다

람다 포팅 중 안드로이드에 적합한 라이브러리는 레트로람다(Retrolambda) 라이브러리이다. (https://github.com/orfjackal/retrolambda) 레트로람다 라이브러리는 속도 상의 약점이 없으며 기술 선도 기업들이 2013년 부터 안드로이드에 테스트한 라이브러리이다.

레트로람다 라이브러리를 사용하기 위해서는 그래들 코드의 설정의 수정이 필요하다.

````
buildscript {
    repositories {
        jcenter()
    }

    dependencies {
        classpath 'me.tatarka:gradle-retrolambda:2.5.0'
    }
}

apply plugin: 'com.android.application'
apply plugin: 'me.tatarka.retrolambda'

android {
    compileSdkVersion 21
    buildToolsVersion "21.1.2"

    defaultConfig {
        applicationId "rx.android.gdg.kr.java8lambda"
        minSdkVersion 11
        targetSdkVersion 21
        versionCode 1
        versionName "1.0"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
}

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    compile 'com.android.support:appcompat-v7:21.0.3'
    compile 'io.reactivex:rxandroid:0.24.0'
}
````
<리스트 20> 레트로람다를 적용한 그래들 파일의 예

자바 8과 옛 버전을 함께 쓰기 위해서는 Gradle Retrolambda Plugin 프로젝트(https://github.com/evant/gradle-retrolambda)를 참조하라.

## 반응형 프로그래밍 도입으로 사용자 경험 향상으로

이번 글에서는 반응형 프로그래밍의 가장 기초적인 개념 세가지 옵저버블, 서브스크라이버, 오퍼레이터를 다루었다. 그 개념을 바탕으로 데이터 스트림을 간단히 연결하고 변형하는 방법을 알아보았다.

다음 시간에는 다양한 옵저버블과 오퍼레이터 등을 합성하여 사용자 인터페이스와 상호작용하는 방법을 알아보며 사용자 인터페이스 코드를 구조화할 것이다.

여기 사용된 코드 예제는 Github 저장소 (https://github.com/GDG-Korea/HelloRx) 에서 볼 수 있다.
