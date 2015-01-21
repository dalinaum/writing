# 안드로이드 롤리팝 개발 따라잡기

안드로이드 롤리팝(5.0) 버전은 구글이 안드로이드를 발표한 이후로 가장 큰 변화를 준 운영체제다. 구글은 허니컴 이후 계속 유지했던 디자인 언어 '홀로(Holo)'를 버리고 새로운 디자인 언어인 머터리얼 디자인으로 안드로이드 외관을 대대적으로 변경했다. 이 변화는 단순히 겉모습에만 그치지 않았다. 내부적으로도 5000개 이상의 API를 추가하고 상당한 양의 API를 폐기(deprecated)했다. ART와 프로젝트 볼타(Project Volta)등의 구조적인 혁신도 뒤따랐다. 안드로이드 롤리팝의 모든 변화를 한 번의 글로 전부 다룰 순 없겠지만, 개발자 입장에서 도움이 될 만한 주요 변화들을 모아 소개하고자 한다. 


![](http://image.chosun.com/sitedata/image/201410/29/2014102903089_0.jpg)

김용욱 dalinaum@gmail.com | GDG Korea Android 오거나이저. 영화를 광적으로 좋아하는 안드로이드 개발자로 최근에는 Reactive Programming에 빠져 RxJava, RxAndroid에 매진하고 있다. 

![](http://developer.android.com/images/home/l-hero_2x.png)

<그림 1> 안드로이드 롤리팝(5.0)


## 배터리 효율

구글은 젤리빈(4.1) 이후 안드로이드의 주요 업데이트 때마다 프로젝트를 하나씩 진행했다. 젤리빈은 프로젝트 버터(Project Butter)를 통해 쾌적한 사용자 경험을 얻었고, 킷캣(4.4)은 프로젝트 스벨트(Project Svelte)를 통해 512MiB(Mebibyte) 메모리 환경에서도 수행할 수 있는 날렵한 몸을 갖게 됐다. 최신 버전 롤리팝(5.0)은 프로젝트 볼타(Project Volta)를 통해 조금 더 적은 전력을 사용하게 됐고 애플리케이션의 효율을 높이는 여러 가지 방법을 고안했다.


### JobScheduler

상황에 대한 고려가 없는 비동기 작업은 배터리 사용에 부정적이다. 네트워크가 안정적이지 않은 상황 속에 사용자가 촬영해 둔 사진을 서버로 백업하면 휴대의 배터리가 금방 바닥이 날 수 있다. 그 때문에 네트워크가 안정적일 때나 충전 중에 사진을 업로드하는 것이 훨씬 효율적이다. 롤리팝은 JobScheduler API를 도입해 배터리와 네트워크 상황에 따라 적절한 작업 계획을 잡을 수 있도록 한다.

JobScheduler API는 사용자 수준의 클래스와 안드로이드 서비스로 구성돼 있다. 롤리팝 장비가 연결된 PC에서 <리스트 1>의 커맨드를 입력하면 JobScheduler의 안드로이드 서비스를 확인할 수 있다.

````
$ adb shell service list | grep jobscheduler
25  jobscheduler: [android.app.job.IJobScheduler]
````

<리스트 1> 안드로이드 서비스 리스트에서 jobscheduler 추가를 확인


JobScheduler API는 `JobInfo`와 `JobService` 2개 객체로 구성돼 있다. 

 * `JobInfo` - 작업이 수행돼야 하는 제약 상황을 기술한다(네트워크, 충전, 주기 등).
 * `JobService` - 계획된 작업을 수행하는 서비스 객체

직접 작업을 정의하고 수행해보자. <리스트 2>는 와이파이 네트워크가 연결돼 있으면서 충전이 진행되는 안전한 환경에서만 `MicrosoftwareService` 서비스를 수행하도록 한 코드다. 적절한 주기와 데드라인도 설정했다.

````
JobInfo job = new JobInfo.Builder(JOB_ID, new ComponentName(this, MicroSoftwareService.class))
        .setRequiredNetworksCapabilities(JobInfo.NETWORK_TYPE_UNMETERED)
        .setPeriodic(15 * DateUtils.HOURS_IN_MILLIS)
        .setRequiresCharging(true)
        .setOverrideDeadline(3600000)
        .build();

JobService mJobService = (JobScheduler) context.getSystemService(Context.JOB_SCHEDULER_SERVICE);
mJobService.scheduleJob(job);
````

<리스트 2> JobScheduler API 사용


더 상세한 `JobInfo` 파라미터를 알고 싶으면 JobInfo.Builder (https://developer.android.com/reference/android/app/job/JobInfo.Builder.html)를 참고하자.


### JobService 확장하기

`JobService` 객체의 경우 일반적인 작업 수행에는  충분하지만 다양한 응용 개발 과정에 확장이 필요할 수 있다. `JobService`는 오버라이딩 가능한 메서드 `onStartJob`과 `onStopJob`을 가지며, 이를 수정해 조금 더 세밀한 작업이 가능하도록 조율한다.

`JobService`의 확장을 위해서 <리스트 3>과 같이 먼저 `AndroidManifest.xml`에 새로운 서비스를 등록해야 한다.

````
 <service
 android:name="kr.co.imaso.MasoJobService"
 android:permission="android.permission.BIND_JOB_SERVICE"
 android:exported="true" />
````

<리스트 3> AndroidManifest.xml 등록


<리스트 4>의 `onStartJob`은 작업이 시작될 때 호출되며 별도로 수행할 작업이 등록되는 곳이다. `onStopJob`은 작업이 끝날 때 호출되는 곳이다. 반환 값 true는 오버라이딩이 됐고 커스터마이징된 작업이 수행됐음을 의미한다. `onStartJob`이 리턴되기 전에 추가적으로 해야할 일을 수행하자. 

````
public class MasoJobService extends JobService {

    @Override
    public boolean onStartJob(JobParameters jobParameters) {
        // FIXME: 작업
        return true;
    }

    @Override
    public boolean onStopJob(JobParameters params) {
        return false;
    }
}

````

<리스트 4> 확장된 JobService, MasoJobService.



### 배터리 상태

애플리케이션이 수행된 후 배터리의 상황은 계속 변화한다. 안드로이드 롤리팝은 배터리 상황 개괄을 확인할 수 있는 명령 `dumpsys batterystats`를 제공한다.

````
$ adb shell dumpsys batterystats —charged kr.co.imaso.MasoActivity
````

<리스트 5> kr.co.imaso.MasoActivity 패키지 배터리 상황 확인


조금 더 상세한 배터리 소모를 확인하기 위해 버그리포트를 출력할 수도 있다. 버그리포트를 출력하기 위해서는 <리스트 6>과 같은 몇 가지 설정 절차가 필요하다.

````
$ adb shell dumpsys batterystats --enable full-wake-history
Enabled: full-wake-history

$ adb shell dumpsys batterystats --reset
Battery stats reset.
````

<리스트 6> 전체 웨이크 히스토리를 기록하도록 환경 설정


분석하고 싶은 앱을 사용한 후 <리스트 7>의 커맨드를 입력해 버그리포트를 추출한다.

````
$ adb bugreport > bugreport.txt
````

<리스트 7> 버그리포트를 통해 웨이크 히스토리를 추출


참고 : 추출된 버그리포트의 분석을 돕는 툴로 구글이 공개한 'Battery Historian(https://github.com/google/battery-historian)'이 있지만, 현재 제대로 동작하지 않고 있다. 이후 리포지토리의 업데이트를 확인해 적용하도록 한다.


## 노티피케이션

롤리팝의 노티피케이션(Notification)은 조금 더 세심하고 강력하고 까탈스러워졌다. 휴대폰을 잠금 상태로 해놨음에도 불구하고 민감한 메신저 채팅이 알림을 통해 노출돼 난감했던 사람들에겐 희소식이다. 대신 알록달록한 노티피케이션은 이제 기대할 수 없게 됐다. `RemoteControlClient`를 이용해 커스터마이징하는 걸 즐겼던 사람들은 이제 그 만큼의 자유를 만끽하기 어려울 것이다.

![](http://developer.android.com/images/versions/notification-headsup.png)

<그림 2> 헤즈업 노티피케이션. 우선 순위가 높은 노티피케이션을 작은 창으로 표시한다.


### 락스크린의 프라이버시

잠금 상태에서 노티피케이션 가시성은 3단계로 나뉜다.

 * `VISIBILITY_PRIVATE` - 노티피케이션 아이콘과 같은 기본적인 정보만 표출되며 상세한 내용은 감춘다.
 * `VISIBILITY_PUBLIC` - 노티피케이션의 컨텐츠가 보인다.
 * `VISIBILITY_SECRET` - 아이콘을 포함해 아무 것도 보이지 않는다.

사용자에게 민감한 노티피케이션은 `Notification.builder.setVisibility(VISIBILITY_PRIVATE)`나 `VISIBILITY_SECRET`으로 분류해 프라이버시를 강화할 수 있다.


### 강화된 메타 데이터

노티피케이션의 메타 데이터를 더 세밀하게 설정할 수 있게 됐다. 그 덕분에 메타 데이터에 맞춰 운영체제가 노티피케이션을 정리해서 보여줄 수 있다.

 * `setCategory()` - 운영체제가 우선 순위에 따라 어떻게 처리할지 힌트를 줄 수 있다. (노티피케이션이 전화, 메시지, 알람과 관련이 있는지 여부를 설정할 수 있다.)
 * `setPriority()` - 노티피케이션의 우선 순위를 변경해 중요도에 따라 정보를 분리할 수 있다. `PRIORITY_MAX`와 `PRIORITY_HIGH`의 경우 작게 떠 있는 화면으로 나타나며 동시에 소리를 내거나 진동을 울리게 된다.
 * `addPerson()` - 노티피케이션의 대상을 한 사람 이상 설정할 수 있다. 특정 사람에게 노티를 보낼 수도 있고 여러 사람에게 중요한 노티피케이션을 보낼 수 있다.

카테고리와 우선 순위에 대한 상세한 내용은 Notification (http://developer.android.com/reference/android/app/Notification.html)을 참고하자.


### 단아한 아이콘

롤리팝에서는 노티피케이션 아이콘의 정책도 변경됐다. 스몰 아이콘의 색상으로 흰색과 투명색만 쓸 수 있도록 제약이 생겼다. 현재는 애플리케이션의 타겟 버전과 단말기의 환경에 따라 다르게 보이지만 롤리팝 이후의 환경을 고려할 때 스몰 아이콘은 흰색과 투명색으로만 디자인 하는 것이 적절하다. 롤리팝의 스몰 아이콘은 `setColor`를 호출해 배경색상을 정할 수 있으니 단색 아이콘과 배경 색상의 조화를 고려하는 것이 좋다(<리스트 8> 참고).

````
NotificationCompat.Builder builder = new NotificationCompat.Builder(this);
int color = getResources().getColor(R.color.imaso);
builder.setColor(color);
Notification notif = builder.build();
````

<리스트 8> 배경 색상이 등록된 노티피케이션.


`NotificationCompat`은 구 버전 단말에서도 사용할 수 있도록 호환성 라이브러리에 추가된 노티피케이션 객체이다. 호환성 라이브러리를 사용하려면 `build.gralde` 파일에 <리스트 9>에 기술된 의존성을 추가한다.

````
dependencies {
  compile 'com.android.support:appcompat-v7:21.0.3'
}
````

<리스트 9> appcompat 라이브러리 의존성 등록


### RemoteControlClient의 폐기

`RemoteControlClient`가 폐기됨에 따라 음악 재생 앱 등은 노티피케이션 변경이 요구된다. <리스트 10>처럼 노티피케이션 스타일 `Notification.MediaStyle`을 생성하고 `Notification.Builder`의 스타일로 등록해 이용한다. 

````
Notification.Builder builder = new Notification.Builder(this)
    .setSmallIcon(R.drawable.ic_launcher)
    .setContentTitle("마이크로소프트웨어")
    .setContentText("안드로이드 롤리팝")
    .setDeleteIntent(pendingIntent)
    .setStyle(new Notification.MediaStyle());
````

<리스트 10> MediaStyle 형식의 노티피케이션


<리스트 11>과 같이 `MediaSessionManager` 서비스를 얻은 후 세션을 만들고, 그 세션으로부터 토큰을 받아와 미디어 컨트롤에 설정한다.

````
mManager = (MediaSessionManager) getSystemService(Context.MEDIA_SESSION_SERVICE);
mSession = mManager.createSession("microsoftware session");
mController = MediaController.fromToken(mSession.getSessionToken());
````

<리스트 11> MediaController 환경 설정.


세션의 `TransportControlsCallback`을 등록하고, 각 상황에 따라 다른 노티피케이션을 생성하도록 `onPlay`, `onPause` 등의 메소드를 오버라이드한다. <리스트 12>는 콜백을 등록하는 과정과 스켈레톤을 대략적으로 보이고 있으며, 오버라이딩할 메소드 마다 개별적으로 노티피케이션을 설정하고 띄워야 한다. 예를 들어 `onPlay` 메소드는 재생과 관련된 노티피케이션을 띄워야 한다.

````
mSession.addTransportControlsCallback(new MediaSession.TransportControlsCallback() {
    @Override
    public void onPlay() {}

    @Override
    public void onPause() {}
    ...
}
````

<리스트 12> TransportControlsCallback의 개괄과 등록 과정


노티피케이션의 액션을 다른 서비스로 연결시키고 해당 서비스에서 `mController.getTransportControls().play()` 등을 호출 한다.


## 오버뷰

최근 화면(recents screen)이 오버뷰(overview)로 변경됐다. 최근 작업들은 3D로 개성있게 렌더링되며 여러 문서와 탭을 쉽게 이동할 수 있도록 `AppTask`를 모두 표출한다. 웹브라우저의 탭들은 오버뷰를 통해 이동할 수 있고 구글 독스의 여러 문서를 오버뷰 스크린에서 이동할 수 있다.

![](http://developer.android.com/images/versions/recents_screen_2x.png)

<그림 3> 오버뷰(overview), 최근 문서(작업, 탭)를 3D 박스로 표현한다.


새로운 도큐먼트를 생성하기 위해서는 <리스트 13> 처럼  `android.content.Intent.FLAG_ACTIVITY_NEW_DOCUMENT` 플래그를 포함한 액티비티를 호출한다.

````
Intent intent = new Intent(this, MicroSoftware.class);
intent.addFlags(android.content.Intent.FLAG_ACTIVITY_NEW_DOCUMENT);
startActivity(intent);
````

<리스트 13> 도큐먼트를 오버뷰에 추가하기 위해 FLAG_ACTIVITY_NEW_DOCUMENT 플래그를 사용


웹사이트 문서도 오버뷰에 대응 할 수 있다. 웹 문서에 <리스트 14>와 같이 메타 데이터 `theme-color`를 설정하면 오버뷰 문서 속성으로 테마 색이 등록된다.

````
<meta name="theme-color" content="#3FFFB5">
````

<리스트 14> 오버뷰를 위한 메타데이터 적용한 웹페이지


## 런타임 엔진 ART

초기 안드로이드에 탑재된 가상 머신 달빅(Dalvik)은 모바일 환경을 고려해 적은 메모리, 최적화된 리소스 관리, 최소화 오버헤드 등이 목표였다. 자주 수행되는 구간(트레이스, trace)을 기계어 코드로 바꾸는 JIT(Just-in-time) 컴파일러는 안드로이드 프로요(2.2) 버전에서야 도입됐다. 달빅의 기본 전략은 달빅 바이트코드(Dalvik bytecode)로 된 코드를 한줄 씩 해석하는 것이며, 반복 수행되는 구간(trace)을 기계어 코드로 변환해 효율을 높이는 구조이다. 트레이스라 불리는 특정 구간을 번역하는 작업은 메소드 단위로 기계어로 변환하는 것보다 시간이 짧게 소요되고 변환된 기계어 코드의 용량이 적기 때문에 메모리가 적고 CPU 파워가 낮은 상황에 적합했던 방식이다. 저사양 단말기에 적합한 장점이 있는 반면에 경량 가상 머신에 시작한 달빅은 고성능과 고기능에서 구조적인 한계는 있는 것이 사실이다.

구글은 안드로이드 킷캣 버전 부터 ART(Android Runtime)를 준비했고 롤리팝 버전 부터는 강제 사항이 되었다. ART는 AOT(ahead-of-time) 방식의 런타임 환경이다. ART의 내장된 dex2oat 유틸리티는 달빅에서 쓰이던 달빅 바이트코드와 리소스 파일이 통합된 .dex 파일을 리눅스에서 널리 쓰이는 실행파일인 형태인 ELF (Executable and Linkable Format)로 변환한다. dex2oat는 앱의 초기 수행시 호출되며 ART의 AOT는 앱의 최초 수행 과정에 dex2oat 유틸리티를 이용하여 실행 파일을 얻어내는 기술인 셈이다. 이로 인해 롤리팝은 수행 성능, 가비지 컬렉션(GC, Garbage collection)의 성능, 프로파일링, 디버깅 등에서 이점을 얻었다.

![](http://upload.wikimedia.org/wikipedia/commons/2/25/ART_view.png)

<그림 4> 달빅과 ART의 차이를 표현. 달빅은 DEX 파일에서 최적화된 Odex를 얻고 ART는 실행파일 ELF를 얻는다.


새로운 방식이 적용되었기 때문에 앱에 따라 문제가 발생할 수 있다. AOT에 관련된 이슈는 안드로이드 플랫폼과 관련된 이슈가 많기 때문에 안드로이드 이슈 리스트 (https://code.google.com/p/android/issues/list) 를 참조하며 문제를 해결하자.


### 성급한 GC 최적화

GC의 구조가 변경되었기 때문에 `GC_FOR_ALLOC` 이벤트가 발생하는 빈도를 줄이기 위해 명시적으로 `System.gc()`를 호출할 필요가 없어졌다. 현재 환경이 달빅이 아닌 ART인 것을 환영하기 위해 <리스트 15>의 커맨드를 이용해서 버전 정보를 얻는다.

````
System.getProperty("java.vm.version")
````

<리스트 15> ART 환경 확인.


버전 정보가 2.0.0 이상인 경우 명시적인 GC 호출이 필요가 줄어들었다. 버전 정보에 따라 GC 호출을 제외하자.


### JNI 디버깅

ART의 채택에 따라 기존에 잘 동작하던 JNI 앱의 동작에 문제가 생길 수 있다. JNI의 동작에 문제가 있는 경우 안드로이드에 포함된 CheckJNI 툴이 도움이 된다.

````
$ adb shell setprop debug.checkjni 1
````

<리스트 16> JNI 디버깅 모드 활성화.


환경이 설정된 후 JNI 코드가 포함된 앱을 수행할 때 시스템은 종종 <리스트 17>과 같이 경고나 에러 메시지를 출력된다.

````
W JNI WARNING: method declared to return 'Ljava/lang/String;' returned '[B'
W              failed in LJniTest;.exampleJniBug
I "main" prio=5 tid=1 RUNNABLE
I   | group="main" sCount=0 dsCount=0 obj=0x40246f60 self=0x10538
I   | sysTid=15295 nice=0 sched=0/0 cgrp=default handle=-2145061784
I   | schedstat=( 398335000 1493000 253 ) utm=25 stm=14 core=0
I   at JniTest.exampleJniBug(Native Method)
I   at JniTest.main(JniTest.java:11)
I   at dalvik.system.NativeStart.main(Native Method)
I 
E VM aborting
````

<리스트 17> 향상된 디버깅 메시지.


향상된 디버그 모드를 이용하여 예상되는 JNI 문제를 해결하자.


### Compacting GC

기존 안드로이드에서는 할당된 메모리 블록의 주소 값이 변경되는 경우는 없었다. 반면에 메모리 사용의 효율을 높이기 위해 안드로이드 ART는 할당된 메모리 블록을 정리하는 Compacting GC를 도입하고 있다. Compacting GC는 메모리의 연속적인 배치를 통해 효율성을 높이는 메커니즘이다. 연속된 배치를 위한 메모리 블록의 이동은 기할당된 메모리 블록의 주소를 변경할 수 있다. `Get<type>ArrayElements`를 호출하여 주소를 얻은 경우 적절하게 `Release<type>ArrayElements()`를 호출해야 하지 않으면 문제가 발생할 수 있다.  ART 상의 주소의 값은 변경 가능한 것이라는 걸 유의하자.


### Object 모델 변경

Object 클래스의 필드 속성이 private으로 변경되었다. Object 필드를 Reflection으로 접근하는 경우에 문제가 될 수 있다. 


## 중복된 커스텀 퍼미션 선언 문제

````
<permission android:name= "com.example.gcm.permission.C2D_MESSAGE"
  android:protectionLevel="signature" />
````

<리스트 18> 커스텀 퍼미션의 보안 레벨은 인증키를 기준으로.


안드로이드 5.0부터 커스텀 퍼미션은 동일한 사인키를 가진 앱에서만 사용할 수 있도록 변경되었다. 커스텀 퍼미션을 사용할 때는 <리스트 18>과 같이 `android:protectionLevel="signature" `를 설정하여야 한다.

만일 같은 퍼미션을 사용하고 있는 앱이 다른 사인키를 가지고 있다면 `INSTALL_FAILED_DUPLICATE_PERMISSION` 에러 메시지와 함께 설치가 거부된다. 이 변경 사항은 앱의 `targetSDK` 버전과는 무관하며 롤리팝 디바이스에서는 강제로 적용되는 사항이다.

기본 퍼미션으로 가능하다면 가능한 커스텀 퍼미션을 쓰지 않는 것이 좋다. 커스텀 퍼미션을 반드시 써야 한다면 패키지 명을 붙여 다른 커스텀 퍼미션과 충돌하지 않도록 관리하며 여러 앱에서 사용해야 한다면 사인키를 잘 관리해야 한다.


## 명시적인 서비스 바인드

서비스 바인드를 할때 명시적인 인텐트만 가능하도록 바뀌었다. <리스트 19>와 같이 암묵적인 바인드를 요청할 경우에는 실행시간 예외가 발생한다.

````
Intent intent = new Intent(MICROSOFTWARE_BINDING);
bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
````

<리스트 19> 에러가 발생하는 암묵적 서비스 바인딩.


롤리팝에서 제대로 된 바인드는 <리스트 19>와 같다.

````
Intent intent = new Intent(this, MicroSoftwareService.class);
bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
````

<리스트 19> 롤리팝에서 권장하는 명시적 서비스 바인딩.


## 머터리얼 디자인

![](http://material-design.storage.googleapis.com/publish/v_2/material_ext_publish/0Bx4BSt6jniD7S1dwdXNVa1B1OHc/components_cards6.png)

<그림 5> 머터리얼 디자인

구글의 통합 디자인 언어 머터리얼 디자인이 안드로이드에 통합되었다. 잉크와 종이를 컨셉으로 한 다양한 UX 컨셉이 도입되었고 그에 따라 변화된 부분도 많이 존재한다. `appcompat` 라이브러리를 통해 기존 라이브러리에서 기존 안드로이드 버전에서도 머터리얼 디자인을 부분적으로 적용할 수 있게 되었다.

`appcompat`를 통한 머터리얼 디자인의 지원을 통해 시각적인 부분은 구형 단말에 적용이 가능해졌으나 에니메이션의 경우에는 롤리팝 디바이스에서만 가능한 경우가 많다. 롤리팝 버전에 렌더링 스레드가 추가되었다. 렌더링 스레드는 메인 스레드 (혹은 UI 스레드)와 분리된 별도의 스레드에서 에니메이션을 다룬다. 별도의 스레드를 쓰는 에니메이션은 쾌적한 반면 세밀한 조작이 어렵다. 롤리팝에서 추가된 에니메이션들도 대부분 작동 중에 조작이 어려운 원샷 에니메이션이다. 이렇게 롤리팝에 추가된 에니메이션 들은 렌더링 스레드에 의존적인 경우가 많아 구 단말을 위한 백 포팅이 어렵다.

머터리얼 디자인을 적용할 때 시각적인 부분과 에니메이션 적인 부분을 분리하여 안드로이드 버전 별로 어떻게 대응해야 할지 고심할 필요가 있다.


### 머터리얼 테마

다양한 안드로이드 버전에서 머터리얼 디자인을 쓸 수 있게 `Theme.AppCompat`를 확장한 경우 `colorPrimary`, `colorPrimaryDark`, `colorAccent` 등의 색상을 설정해야 한다. `Theme.AppCompat`를 위한 색상의 설정에서는 `android:` 접두어가 붙지 않는다.

````
<style name="Theme.MyTheme" parent="Theme.AppCompat.Light">
    <item name="colorPrimary">@color/material_blue_500</item>
    <item name="colorPrimaryDark">@color/material_blue_700</item>
    <item name="colorAccent">@color/material_green_A200</item>
</style>
````

<리스트 20> 다양한 안드로이드 버전을 위한 테마 Theme.AppCompat의 확장.


### 액션바의 폐기

![](http://material-design.storage.googleapis.com/publish/v_2/material_ext_publish/0Bx4BSt6jniD7UnNtdkNxY05oelk/layout_structure_toolbars2.png)

<그림 6> 툴바. 액션바를 대체하는 유연한 앱 바 콤퍼넌트.

안드로이드 허니콤 버전(3.0)부터 적용되었던 액션 바가 폐기되었다. 액션바는 구글이 허니콤 이후로 정착시키려는 가이드라인의 핵심이었고 가이드라인을 강제하기 위해 커스터마이징이 어렵게 되어있었다. 액션바가 폐기되고 툴바가 들어온 것은 머터리얼 디자인에서는 조금 더 다양한 시도를 할 수 있도록 문을 열어줬다고 볼 수 있다.

툴바를 사용하기 위해서 먼저 `appcompat` 라이브러리가 표함되어야 한다.

````
dependencies {
    compile "com.android.support:appcompat-v7:21.0.3"
}
````

<리스트 21> appcompat 라이브러의 의존성 추가.


툴바를 사용하기 위해서는 먼저 액티비티는 `ActionBarActivity`를 상속받고 테마는 `Theme.AppCompat`를 상속받아야 한다.

`ActionBarActivity`의 레이아웃에 `Toolbar`를 추가한다.

````
<android.support.v7.widget.Toolbar
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/toolbar"
    android:layout_height="wrap_content"
    android:layout_width="match_parent"
    android:minHeight="?attr/actionBarSize"
    android:background="?attr/colorPrimary" />
````

<리스트 22> 액션바와 달리 레이아웃 요소의 하나인 툴바.


레이아웃에 `Toolbar`를 추가한 후 액티비티에 연결하는 방법은 두가지가 있다.

 1. 액션바 처럼 연결하기.
 2. 그냥 연결하기.

액션바 처럼 툴바를 연결할 때는 `setSupportActionBar` 메서드를 이용한다.

````
@Override
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.microsoftware_layout);

    Toolbar toolbar = (Toolbar) findViewById(R.id.toolbar);
    setSupportActionBar(toolbar);
}
````

<리스트 23> setSupportActionBar를 이용한 툴바의 사용.


툴바를 활용하는 또 다른 방법은 툴바의 `setOnMenuItemClickListener`와 `inflateMenu`를 이용하는 것이다.

````
@Override
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.microsoftware_layout);

    Toolbar toolbar = (Toolbar) findViewById(R.id.toolbar);

    // Set an OnMenuItemClickListener to handle menu item clicks
    toolbar.setOnMenuItemClickListener(
            new Toolbar.OnMenuItemClickListener() {
                @Override
                public boolean onMenuItemClick(MenuItem item) {
                    // Handle the menu item
                    return true;
                }
    });

    // Inflate a menu to be displayed in the toolbar
    toolbar.inflateMenu(R.menu.microsoftware_menu);
}
````

<리스트 24> 액션바와 상관없는 독자적인 툴바의 사용.


### 네비게이션 드로어 변경하기

![](http://material-design.storage.googleapis.com/publish/v_2/material_ext_publish/0Bx4BSt6jniD7NzhpQzI0R21kOTg/layout_structure_sidenav_structure1.png)

<그림 7> 머터리얼 디자인 네비게이션 드로어

롤리팝의 네비게이션 드로어는 화면 전체를 가리는 형태가 되었다. 이렇게 드로어가 화면 전체를 가리기 위해서 <리스트 25>와 같이 레이아웃에서 `DrawLayout`이 포함되는 구성을 바꾼다.

````
<android.support.v4.widget.DrawerLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/drawer_layout"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true">
 
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">
 
        <android.support.v7.widget.Toolbar
            android:id="@+id/toolbar"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:minHeight="?attr/actionBarSize" />
 
        <!-- 어플리케이션 UI -->
 
    </LinearLayout>
 
    <View
        android:id="@+id/drawer"
        android:layout_width="240dp"
        android:layout_height="match_parent"
        android:layout_gravity="start"
        android:background="#3F51B5"
        android:fitsSystemWindows="true" />
 
 
</android.support.v4.widget.DrawerLayout>
````

<리스트 25> 머터리얼 디자인에 맞추어 변경돈 드로어 레이아웃.


`DrawerLayout`을 루트 레이아웃으로 변경하고 `Toolbar`와 어플리케이션 UI를 속에 내포하는 형태로 레이아웃을 변경한다.


## 훤히 보이는 고생길, 그럼에도 기대하는 새로운 희망

안드로이드 5.0에서 변화된 많은 부분을 한 자리에서 설명한다는 것은 어려운 일이며 여기에서 설명하지 않은 변화들도 앞으로의 안드로이드 개발에 많은 영향을 끼칠 것이다. 개발에 있어 장애도 있을 수 있겠지만 다른 측면에서 본다면 새로운 기능과 디자인에 의한 여러가지 긍정적인 변화도 기대된다. 

안드로이드 롤리팝에 대한 다양한 경험을 서로 공유하며 조금 더 달콤한 안드로이드를 만나보았으면 하는 바람이다.

다음은 롤리팝에 대한 정보가 공유되는 사이트들이다.

 * GDG 코리아 안드로이드 - https://plus.google.com/communities/100903743067544956282
 * 구글 개발자 코리아 블로그 - http://googledevkr.blogspot.kr/
 * 페이스북 커뮤니티 안로이드 팁팁팁 - https://www.facebook.com/groups/junsle
