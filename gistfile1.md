# 배터리 효율

구글은 젤리빈(4.1) 이후 안드로이드 주요 업데이트 마다 프로젝트를 하나씩 진행했다. 젤리빈은 프로젝트 버터(Project Butter)를 통해 쾌적한 사용자 경험을 얻었다. 킷캣(4.4)은 프로젝트 스벨트(Project Svelte)를 통해 512MiB(Mebibyte) 메모리 환경에서도 수행할 수 있는 날렵한 몸을 얻었다. 안드로이드의 최신 버전 롤리팝(5.0)은 프로젝트 볼타(Project Volta)를 통해 조금 더 적은 전력을 쓰게 되었고 어플리케이션의 효율을 높일 여러 방법도 마련했다.

## JobScheduler

상황에 대한 고려가 없는 비동기 작업은 배터리 효율에 부정적이다. 네트워크가 안정적이지 않은 상황에 사용자가 촬영해둔 사진을 서버로 백업하면 휴대폰의 배터리가 금방 끝날 수 있다. 네트워크가 안정적일 때나 충전 중에 사진을 업로드하는 것이 훨씬 효율적이다. 롤리팝은 JobScheduler API를 도입하여 배터리와 네트워크 상황에 따라 적절한 작업 계획을 잡을 수 있도록 한다.

JobScheduler API는 사용자 수준의 클래스와 안드로이드 서비스로 구성되어 있다. 롤리팝 장비가 연결된 PC에서 아래 커맨드를 입력하면 JobScheduler의 안드로이드 서비스를 확인할 수 있다.

````
$ adb shell service list | grep jobscheduler
25  jobscheduler: [android.app.job.IJobScheduler]
````

JobScheduler API는 두개의 객체 `JobInfo`와 `JobService`로 구성되어 있다. 

 * JobInfo - 작업이 수행되어야 하는 제약 상황을 기술한다. (네트워크, 충전, 주기 등)
 * JobService - 계획된 작업이 수행될 서비스

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

더 상세한 `JobInfo` 파라미터를 파악하려 JobInfo.Builder https://developer.android.com/reference/android/app/job/JobInfo.Builder.html 를 참고하라.

## JobService 확장하기

`JobService`가 일반적인 작업을 수행하는데 충분하지만 확장을 원할 수 있다. `JobService`는 오버라이딩 가능한 메소드 `onStartJob`와 `onStopJob`가 있으며 이를 통해 조금 더 세밀한 조작이 가능하다.

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

## 배터리 상태

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

 * 참고: 추출된 버그리포트를 분석하기 위해서는 툴 battery historian https://github.com/google/battery-historian 이 공개되어 있으나 현재는 제대로 동작하지 않고 있다.