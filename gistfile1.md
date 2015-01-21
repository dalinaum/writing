# 안드로이드 롤리팝 개발 따라잡기

안드로이드 롤리팝(5.0) 버전은 안드로이드가 발표된 이후로 가장 크게 변화를 한 운영체제다. 허니컴 이래 확립했던 디자인 언어 홀로(Holo)을 버리고 구글의 새로운 디자인 언어인 머터리얼 디자인으로 외양을 대대적으로 변경하였고, 내부적으로도 5000개 이상의 API가 추가되고 수 상당한 양의 API를 폐기(deprecated)하였다. 안드로이드 롤리팝의 모든 변화를 글 하나에 다룰 수는 없지만, 롤리팝을 길들이는데 도움이 될 수 있게 개발자에게 큰 영향을 미치는 변화를 한 곳에 모아 소개한다.


![](http://image.chosun.com/sitedata/image/201410/29/2014102903089_0.jpg)

김용욱 dalinaum@gmail.com | GDG Korea Android 오거나이저. 영화를 광적으로 좋아하는 안드로이드 개발자로 최근에는 Reactive Programming에 빠져 RxJava, RxAndroid에 매진하고 있다. 

![](http://developer.android.com/images/home/l-hero_2x.png)

<그림 1> 안드로이드 롤리팝 (5.0)

## 배터리 효율

구글은 젤리빈(4.1) 이후 안드로이드 주요 업데이트 마다 프로젝트를 하나씩 진행했다. 젤리빈은 프로젝트 버터(Project Butter)를 통해 쾌적한 사용자 경험을 얻었다. 킷캣(4.4)은 프로젝트 스벨트(Project Svelte)를 통해 512MiB(Mebibyte) 메모리 환경에서도 수행할 수 있는 날렵한 몸을 얻었다. 안드로이드의 최신 버전 롤리팝(5.0)은 프로젝트 볼타(Project Volta)를 통해 조금 더 적은 전력을 쓰게 되었고 어플리케이션의 효율을 높일 여러 방법도 마련했다.

### JobScheduler

상황에 대한 고려가 없는 비동기 작업은 배터리 효율에 부정적이다. 네트워크가 안정적이지 않은 상황에 사용자가 촬영해둔 사진을 서버로 백업하면 휴대폰의 배터리가 금방 끝날 수 있다. 네트워크가 안정적일 때나 충전 중에 사진을 업로드하는 것이 훨씬 효율적이다. 롤리팝은 JobScheduler API를 도입하여 배터리와 네트워크 상황에 따라 적절한 작업 계획을 잡을 수 있도록 한다.

JobScheduler API는 사용자 수준의 클래스와 안드로이드 서비스로 구성되어 있다. 롤리팝 장비가 연결된 PC에서 아래 커맨드를 입력하면 JobScheduler의 안드로이드 서비스를 확인할 수 있다.

````
$ adb shell service list | grep jobscheduler
25  jobscheduler: [android.app.job.IJobScheduler]
````

JobScheduler API는 두개의 객체 `JobInfo`와 `JobService`로 구성되어 있다. 

 * `JobInfo` - 작업이 수행되어야 하는 제약 상황을 기술한다. (네트워크, 충전, 주기 등)
 * `JobService` - 계획된 작업이 수행될 서비스

작업을 정의하고 수행해보자. 와이파이 네트워크가 연결되고 충전이 진행되는 안전한 환경에서만 `MicrosoftwareService` 서비스를 수행하도록 하고 적절한 주기와 데드라인을 설정하였다.

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

더 상세한 `JobInfo` 파라미터를 하려면 JobInfo.Builder(https://developer.android.com/reference/android/app/job/JobInfo.Builder.html) 를 참고하라.

### JobService 확장하기

`JobService`가 일반적인 작업을 수행하는데 충분하지만 확장을 원할 수 있다. `JobService`는 오버라이딩 가능한 메소드 `onStartJob`와 `onStopJob`를 가지며 이를 수정하여 조금 더 세밀하게 작업을 조율할 수 있다.

`JobService`의 확장을 위해서는 `AndroidManifest.xml`에 새로운 서비스를 등록해야 한다.

````
 <service
 android:name="kr.co.imaso.MasoJobService"
 android:permission="android.permission.BIND_JOB_SERVICE"
 android:exported="true" />
````

`onStartJob`은 작업이 시작될 때 호출되며 별도로 수행할 작업이 등록되어야 하는 곳이고 `onStopJob`은 작업이 끝날 때 호출되는 곳이다. `onStartJob`이 리턴되기 전에 추가적으로 해야할 일을 수행하자. 

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

### 배터리 상태

어플리케이션이 수행 후에 배터리의 상황은 계속 변화한다. 안드로이드 롤리팝은 배터리 상황 개괄을 확인할 수 있는 명령을 제공한다.

````
$ adb shell dumpsys batterystats —charged kr.co.imaso.MasoActivity
````

조금 더 상세한 배터리 소모를 확인하기 위해 버그리포트를 출력할 수 있다. 버그리포트를 출력하기 위해서는 몇가지 환경 설정이 필요하다.

````
$ adb shell dumpsys batterystats --enable full-wake-history
Enabled: full-wake-history

$ adb shell dumpsys batterystats --reset
Battery stats reset.
````

분석하고 싶은 앱을 사용한 이후 아래의 커맨드를 입력하여 버그리포트를 추출한다.

````
$ adb bugreport > bugreport.txt
````

 * 참고: 추출된 버그리포트의 분석을 돕는 툴 Battery Historian(https://github.com/google/battery-historian) 을 구글이 공개하였으나 현재 제대로 동작하지 않고 있다. 리포지토리의 업데이트를 확인하여 적용한다.


## 노티피케이션

노티피케이션은 조금 더 세심하고 강력하고 까탈스러워졌다. 휴대폰을 잠그어 두었지만 민감한 메신저 채팅이 보였던 나쁜 기억은 더 이상 떠올리지 않아도 된다. 알록달록한 노티피케이션, `RemoteControlClient`를 이용한 화려한 커스터마이징을 즐겼다면 앞으로는 그만큼의 자유를 만끽하기는 어렵다.

![](http://developer.android.com/images/versions/notification-headsup.png)

<그림 2> 헤즈업 노티피케이션. 우선 순위가 높은 노티피케이션을 작은 창으로 표시한다.

### 락스크린의 프라이버시

락 스크린 상태에서 노티피케이션 가시성은 3단계로 나뉘어 구분된다.

 * `VISIBILITY_PRIVATE` - 노티피케이션 아이콘과 같은 기본적인 정보만 표출되며 상세한 내용은 감추어 진다.
 * `VISIBILITY_PUBLIC` - 노티피케이션의 컨텐츠가 보인다.
 * `VISIBILITY_SECRET` - 아이콘을 포함하여 아무 것도 보이지 않는다.

사용자에게 민감한 노티피케이션은 `Notification.builder.setVisibility(VISIBILITY_PRIVATE)`나 `VISIBILITY_SECRET`으로 분류하여 프라이버시를 강화할 수 있다.

### 강화된 메타 데이터

노티피케이션의 메타 데이터도 더 세밀해져 설정할 수 있어 메타 데이터에 맞추어 운영체제가 노티피케이션을 정리해서 보여줄 수 있다.

 * `setCategory()` - 운영체제가 우선 순위에 따라 어떻게 처리할지 힌트를 줄 수 있다. (노티피케이션이 전화, 메시지, 알람 등에 연관이 있다 설정할 수 있다.)
 * `setPriority()` - 노티피케이션의 우선 순위를 변경할 수 있어 덜 중요한 정보와 더 중요한 정보를 분리할 수 있다. `PRIORITY_MAX`와 `PRIORITY_HIGH`의 경우에는 작게 떠 있는 화면으로 뜨게 되며 노티피케이션이 소리나 진동을 갖게 된다.
 * `addPerson()` - 노티피케이션의 대상을 한 사람 이상 설정할 수 있다. 특정 사람에게 노티를 보낼 수도 있고 여러 사람에게 중요한 노티를 보낼 수도 있다.

상세 카테고리와 우선 순위는 Notification(http://developer.android.com/reference/android/app/Notification.html) 을 참고한다.

### 단아한 아이콘

롤리팝에서 노티피케이션 아이콘의 정책도 변경되었다. 스몰 아이콘의 색상이 흰색과 투명색 만 쓸 수 있는 것이 제약이다. 현재는 어플리케이션의 타겟 버전과 단말기의 환경에 따라 다르게 보이지만 롤리팝 이후의 환경을 고려할 때 스몰 아이콘은 흰색과 투명색으로만 디자인 하는 것이 적절하다. 롤리팝의 스몰 아이콘은 `setColor`을 호출하여 배경색상을 정할 수 있으니 단색 아이콘과 배경 색상의 조화를 고려하는 것이 좋다.

````
NotificationCompat.Builder builder = new NotificationCompat.Builder(this);
int color = getResources().getColor(R.color.imaso);
builder.setColor(color);
Notification notif = builder.build();
````

`NotificationCompat`은 구 버전 단말에서도 사용할 수 있도록 호환성 라이브러리에 추가된 노티피케이션 객체이다. 호환성 라이브러리를 사용하려면 `build.gralde`에 아래의 의존성을 추가한다.

````
dependencies {
  compile 'com.android.support:appcompat-v7:21.0.3'
}
````

### RemoteControlClient의 폐기

`RemoteControlClient`가 폐기됨에 따라 음악 재생 앱 등의 노티피케이션의 변경이 요구된다. 노티피케이션 스타일 `Notification.MediaStyle`을 이용한다. 

````
Notification.Builder builder = new Notification.Builder(this)
    .setSmallIcon(R.drawable.ic_launcher)
    .setContentTitle("마이크로소프트웨어")
    .setContentText("안드로이드 롤리팝")
    .setDeleteIntent(pendingIntent)
    .setStyle(new Notification.MediaStyle());
````

`MediaSessionManager` 서비스를 얻어 세션을 등록하고 토큰을 설정한다. 

````
mManager = (MediaSessionManager) getSystemService(Context.MEDIA_SESSION_SERVICE);
mSession = mManager.createSession("microsoftware session");
mController = MediaController.fromToken(mSession.getSessionToken());
````

세션의 `TransportControlsCallback`을 등록하고, 각 상황에 따라 다른 노티피케이션을 생성하도록 `onPlay`, `onPause` 등의 메서드를 오버라이드한다.

````
mSession.addTransportControlsCallback(new MediaSession.TransportControlsCallback() {
    @Override
    public void onPlay() {}

    @Override
    public void onPause() {}
    ...
}
````

노티피케이션의 액션을 다른 서비스로 연결시키고 해당 서비스에서 `mController.getTransportControls().play()` 등을 호출 한다.


## 오버뷰

![](http://developer.android.com/images/versions/recents_screen_2x.png)

<그림 3> 오버뷰(overview), 최근 문서(작업, 탭)을 3D 박스로 표현한다.

최근 화면(recents screen)이 오버뷰(overview)로 변경되었다. 최근 작업들은 3D로 개성있게 렌더링되며 여러 문서와 탭을 쉽게 이동할 수 있도록 `AppTask`를 모두 표출한다. 웹 브라우저의 탭들을 오버뷰를 통해 이동할 수 있고 구글 독스의 여러 문서를 오버뷰 스크린에서 이동할 수 있는 것이다.

새로운 도큐먼트를 생성하기 위해서는 `android.content.Intent.FLAG_ACTIVITY_NEW_DOCUMENT` 플래그를 포함하여 액티비티를 호출한다.

````
Intent intent = new Intent(this, MicroSoftware.class);
intent.addFlags(android.content.Intent.FLAG_ACTIVITY_NEW_DOCUMENT);
startActivity(intent);
````

웹 사이트 문서도 오버뷰에 대응 할 수 있다. `theme-color`를 메타 데이터로 설정하면 오버뷰에 적용된다.

````
<meta name="theme-color" content="#3FFFB5">
````


## 런타임 엔진 ART

![](http://upload.wikimedia.org/wikipedia/commons/2/25/ART_view.png)

<그림 4> 달빅과 ART의 차이를 표현. 달빅은 DEX 파일에서 최적화된 Odex를 얻고 ART는 실행파일 ELF를 얻는다.

초기 안드로이드에 탑재된 가상 머신 달빅(Dalvik)은 모바일 환경을 고려해서 적은 메모리, 최적화된 리소스 관리, 최소화 오버헤드 등이 목표였다. 자주 수행되는 구간(트레이스, trace)을 기계어 코드로 바꾸는 JIT (Just-in-time) 컴파일러는 안드로이드 프로요(2.2) 버전에서야 도입되었다. 달빅의 기본 전략은 달빅 바이트코드(Dalvik bytecode)로 된 코드를 한줄 씩 해석하는 것이며, 반복 수행되는 구간(트레이스, trace)를 기계어 코드로 변환하여 효율을 높이는 구조이다. 트레이스라 불리는 특정 구간을 번역하는 작업은 메소드 단위로 기계어로 변환하는 것보다 시간은 짧게 걸리고 변환된 기계어 코드의 용량이 적기 때문에 메모리가 적고 CPU 파워가 낮은 상황에 적합해던 방식이다. 저사양 단말기에 적합한 장점이 이는 반면에 경량 가상 머신에 시작한 달빅은 고성능과 고기능에서 구조적인 한계는 있는 것이 사실이다.

구글은 안드로이드 킷캣 버전 부터 ART(Android Runtime)를 준비했고 롤리팝 버전 부터는 강제 사항이 되었다. ART는 AOT(ahead-of-time) 방식의 런타임 환경이다. ART의 내장된 dex2oat 유틸리티는 달빅에서 쓰이던 달빅 바이트코드와 리소스 파일이 통합된 .dex 파일을 리눅스에서 널리 쓰이는 실행파일인 형태인 ELF (Executable and Linkable Format)로 변환한다. dex2oat는 앱의 초기 수행시 호출되며 ART의 AOT는 앱의 최초 수행 과정에 dex2oat 유틸리티를 이용하여 실행 파일을 얻어내는 기술인 셈이다. 이로 인해 롤리팝은 수행 성능, 가비지 컬렉션(GC, Garbage collection)의 성능, 프로파일링, 디버깅 등에서 이점을 얻었다.

새로운 방식이 적용되었기 때문에 앱에 따라 문제가 발생할 수 있고 AOT에 관련된 이슈는 안드로이드 이슈 리스트(https://code.google.com/p/android/issues/list) 를 자주 참고하면서 해결해야 한다.

### 성급한 GC 최적화

GC의 구조가 변경되었기 때문에 `GC_FOR_ALLOC` 이벤트가 발생하는 빈도를 줄이기 위해 명시적으로 `System.gc()`를 호출할 필요가 없어졌다. 현재 환경이 달빅이 아닌 ART인 것을 환영하기 위해 다음 커맨드를 이용해서 버전 정보를 얻는다.

````
System.getProperty("java.vm.version")
````

버전 정보가 2.0.0 이상인 경우 명시적인 GC 호출이 필요가 없다.

### JNI 디버깅

ART의 채택에 따라 기존에 잘 동작하던 JNI 앱의 동작에 문제가 생길 수 있다. JNI의 동작에 문제가 있는 경우 안드로이드에 포함된 CheckJNI 툴이 도움이 된다.

````
$ adb shell setprop debug.checkjni 1
````

환경이 설정되면 JNI 코드 수행에서 경고나 에러 메시지를 볼 수 있다.

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

### Compacting GC

기존 안드로이드에서는 할당된 메모리 블록의 주소 값이 변경되는 경우는 없었다. 메모리 사용의 효율을 높이기 위해 안드로이드 ART는 할당된 메모리 블록을 정리하는 Compacting GC를 도입하고 있다. 이로 인해 할당된 메모리 블록의 주소가 변경될 수 있다. `Get<type>ArrayElements`를 호하여 얻은 경우 올바르게 `Release<type>ArrayElements()`를 호출해야 하며 주소의 값은 변경될 수 있다는 것을 유의해야 한다.

### Object 모델 변경

Object 클래스의 필드 속성이 private으로 변경되었다. Object 필드를 Reflection으로 접근하는 경우에 문제가 될 수 있다. 


## 중복된 커스터 퍼미션 선언 문제

````
<permission android:name= "com.example.gcm.permission.C2D_MESSAGE"
  android:protectionLevel="signature" />
````

안드로이드 5.0부터 커스텀 퍼미션은 동일한 사인키를 가진 앱에서만 사용할 수 있도록 변경되었다. 같은 퍼미션을 사용하고 있는 앱이 다른 사인키를 가지고 있다면 아래의 `INSTALL_FAILED_DUPLICATE_PERMISSION` 메시지와 함께 설치가 거부된다. 이 변경 사항은 앱의 `targetSDK` 버전과는 무관하며 롤리팝 디바이스에서는 강제로 적용되는 사항이다.

기본 퍼미션으로 가능하다면 가능한 커스텀 퍼미션을 쓰지 않는 것이 좋다. 커스텀 퍼미션을 반드시 써야 한다면 패키지 명을 붙여 다른 커스텀 퍼미션과 충돌하지 않도록 관리하며 여러 앱에서 사용해야 한다면 사인키를 잘 관리해야 한다.

## 명시적인 서비스 바인드

서비스 바인드를 할때 명시적인 인텐트만 가능하도록 바뀌었다. 아래와 같이 암묵적인 바인드를 요청할 경우 예외가 발생한다.

````
Intent intent = new Intent(MICROSOFTWARE_BINDING);
bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
````

제대로 된 바인드는 아래와 같다.

````
Intent intent = new Intent(this, MicroSoftwareService.class);
bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
````

## 머터리얼 디자인

![](http://material-design.storage.googleapis.com/publish/v_2/material_ext_publish/0Bx4BSt6jniD7S1dwdXNVa1B1OHc/components_cards6.png)

<그림 5> 머터리얼 디자인

구글의 통합 디자인 언어 머터리얼 디자인이 안드로이드에 통합되었다. 잉크와 종이를 컨셉으로 한 다양한 UX 컨셉이 도입되었고 그에 따라 변화된 부분도 많이 존재한다. `appcompat` 라이브러리를 통해 기존 라이브러리에서 기존 안드로이드 버전에서도 머터리얼 디자인을 부분적으로 적용할 수 있게 되었다.

시각적인 부분은 적용이 가능해졌으나 에니메이션의 경우에는 구형 안드로이드 장비에서 구현이 불가능한 부분이 많다. 롤리팝은 렌더링 스레드가 추가되었는데 렌더링 스레드에 의존적인 에니메이션은 백 포팅이 불가능하기 때문이다.

머터리얼 디자인을 적용할 때 시각적인 부분과 에니메이션 적인 부분을 나누어 에니메이션 부분은 안드로이드 버전 별로 어떻게 대응해야 할지 고민이 필요하다.

### 머터리얼 테마

`Theme.AppCompat`를 확장한 경우 `colorPrimary`, `colorPrimaryDark`, `colorAccent` 등의 색상을 설정해야 한다. `Theme.AppCompat`를 위한 색상의 설정에서는 `android:` 접두어가 붙지 않는다.

````
<style name="Theme.MyTheme" parent="Theme.AppCompat.Light">
    <item name="colorPrimary">@color/material_blue_500</item>
    <item name="colorPrimaryDark">@color/material_blue_700</item>
    <item name="colorAccent">@color/material_green_A200</item>
</style>
````

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

레이아웃에 `Toolbar`를 추가한 후 액티비티에 연결하는 방법은 두가지가 있다.

 1. 액션바 처럼 연결하기.
 2. 그냥 연결하기.

액션바 처럼 연결하는 것은 `setSupportActionBar` 메서드를 이용하는 것이다.

````
@Override
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.microsoftware_layout);

    Toolbar toolbar = (Toolbar) findViewById(R.id.toolbar);
    setSupportActionBar(toolbar);
}
````

또 다른 방법은 툴바의 `setOnMenuItemClickListener`와 `inflateMenu`를 이용하는 방법이다.

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

### 네비게이션 드로어 변경하기

![](http://material-design.storage.googleapis.com/publish/v_2/material_ext_publish/0Bx4BSt6jniD7NzhpQzI0R21kOTg/layout_structure_sidenav_structure1.png)

<그림 7> 머터리얼 디자인 네비게이션 드로어

롤리팝에서는 네비게이션 드로어가 화면 전체를 가리는 형태로 변경되었다. 이렇게 드로어가 화면 전체를 가리기 위해서는 `DrawLayout` 구성의 변경이 필요하다.

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

`DrawerLayout`을 루트 레이어웃으로 변경하고 `Toolbar`와 어플리케이션 UI를 위한 영역을 내포시키는 형태로 레이아웃을 변경한다.

## 환하게 보이는 고생길 그럼에도 기대하는 새로운 희망

안드로이드 5.0에서 변화된 많은 부분을 한 자리에서 설명한다는 것은 어려운 일이며 여기에서 설명하지 않은 변화들도 앞으로의 안드로이드 개발에 많은 영향을 끼칠 것이다. 개발에 있어 장애도 있을 수 있겠지만 다른 측면에서 본다면 새로운 기능과 디자인에 의한 여러가지 긍정적인 변화도 기대된다. 

안드로이드 롤리팝에 대한 다양한 경험을 서로 공유하며 조금 더 달콤한 안드로이드를 만나보았으면 하는 바람이다.

다음은 롤리팝에 대한 정보가 공유되는 사이트들이다.

 * GDG 코리아 안드로이드 - https://plus.google.com/communities/100903743067544956282
 * 구글 개발자 코리아 블로그 - http://googledevkr.blogspot.kr/
 * 페이스북 커뮤니티 안드로이드 팁팁팁 - https://www.facebook.com/groups/junsle