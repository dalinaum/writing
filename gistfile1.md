# 안드로이드 롤리팝 개발 따라잡기

http://developer.android.com/images/home/l-hero_2x.png

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

더 상세한 `JobInfo` 파라미터를 파악하려 JobInfo.Builder(https://developer.android.com/reference/android/app/job/JobInfo.Builder.html)를 참고하라.

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

 * 참고: 추출된 버그리포트의 분석을 돕는 툴 Battery Historian(https://github.com/google/battery-historian)을 구글이 공개하였으나 현재 제대로 동작하지 않고 있다. 리포지토리의 업데이트를 확인하여 적용한다.


## 노티피케이션

노티피케이션은 조금 더 세심하고 강력하고 까탈스러워졌다. 휴대폰을 잠그어 두었지만 민감한 메신저 채팅이 보였던 나쁜 기억은 더 이상 떠올리지 않아도 된다. 알록달록한 노티피케이션, `RemoteControlClient`를 이용한 화려한 커스터마이징을 즐겼다면 앞으로는 그만큼의 자유를 만끽하기는 어렵다.

http://developer.android.com/images/versions/notification-headsup.png

## 락스크린의 프라이버시

락 스크린 상태에서 노티피케이션 가시성은 3단계로 나뉘어 구분된다.

 * `VISIBILITY_PRIVATE` - 노티피케이션 아이콘과 같은 기본적인 정보만 표출되며 상세한 내용은 감추어 진다.
 * `VISIBILITY_PUBLIC` - 노티피케이션의 컨텐츠가 보인다.
 * `VISIBILITY_SECRET` - 아이콘을 포함하여 아무 것도 보이지 않는다.

사용자에게 민감한 노티피케이션은 `Notification.builder.setVisibility(VISIBILITY_PRIVATE)`나 `VISIBILITY_SECRET`으로 분류하여 프라이버시를 강화할 수 있다.

## 강화된 메타 데이터

노티피케이션의 메타 데이터도 더 세밀해져 설정할 수 있어 메타 데이터에 맞추어 운영체제가 노티피케이션을 정리해서 보여줄 수 있다.

 * `setCategory()` - 운영체제가 우선 순위에 따라 어떻게 처리할지 힌트를 줄 수 있다. (노티피케이션이 전화, 메시지, 알람 등에 연관이 있다 설정할 수 있다.)
 * `setPriority()` - 노티피케이션의 우선 순위를 변경할 수 있어 덜 중요한 정보와 더 중요한 정보를 분리할 수 있다. `PRIORITY_MAX`와 `PRIORITY_HIGH`의 경우에는 작게 떠 있는 화면으로 뜨게 되며 노티피케이션이 소리나 진동을 갖게 된다.
 * `addPerson()` - 노티피케이션의 대상을 한 사람 이상 설정할 수 있다. 특정 사람에게 노티를 보낼 수도 있고 여러 사람에게 중요한 노티를 보낼 수도 있다.

상세 카테고리와 우선 순위는 Notification(http://developer.android.com/reference/android/app/Notification.html)을 참고한다.

## 단아한 아이콘

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


## 오버뷰

http://developer.android.com/images/versions/recents_screen_2x.png

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