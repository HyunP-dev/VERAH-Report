# VERAH 과제 보고서

작성자: 박현 (20205178)  
작성일: 2023-12-12 수요일  


## TICKR Fit를 이용한 심박수 기록 어플리케이션의 개발

### 과제 내용

TICKR Fit는 팔에 착용하는 심박수 모니터링 장치로 
TICKR Fit에서 측정한 심박수를 Android 기기를 통해 
데이터를 처리할 수 있도록 하는 SDK를 제공해준다.
제공 받은 SDK를 이용하여 Android 기기에서 심박수를 
CSV 파일로 저장할 수 있는 어플리케이션이 선행 개발된 상태에서 
개발된 어플리케이션의 소스코드를 컴파일해 이를 직접 사용해보고 이를 
통해 문제점을 찾아 개선하는 것이 본 과제의 내용이 된다.

### 수행 내용

#### 심박수 측정 Service의 비정상적인 종료가 발생하는 문제

선행된 소스코드를 컴파일을 해 직접 시동해본 결과,
측정이 도중에 끊기는 일이 발생함을 관측하였다.
기본적으로 제공 받은 SDK는 Service에서 작동하기 때문에
Foreground 상의 상태와는 관계 없이 작동한다.
이에 배터리 최적화가 작성한 Service를 종료시키는 것이 의심되어
```java
    @SuppressLint("BatteryLife")
    private void ignoreBatteryOptimizations() {
        PowerManager pm = (PowerManager) getSystemService(Context.POWER_SERVICE);
        String packageName = getPackageName();
        if (pm.isIgnoringBatteryOptimizations(packageName)) return;
        Intent intent = new Intent();
        intent.setAction(Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS);
        intent.setData(Uri.parse("package:" + packageName));
        startActivityForResult(intent, 0);
    }
```
위와 같은 메소드를 통해 배터리 최적화를 비활성화함으로써
심박수를 측정하는 서비스가 종료되는 것을 막을 수 있었다.

#### 복잡한 UI 레이아웃의 개선

개발한 어플리케이션이 측정해야 할 값은 심박수로 유일함에도 불구하고,
IMU 센서 까지 측정을 하며 이를 기록 및 실시간 모니터해주는 기능과
센서의 ON/OFF까지 조작할 수 있도록 설계가 되어 있어, 피실험자 혹은 실험자에게
어플리케이션의 조작 방법을 알려주기 번거롭다는 문제점이 있었다.
이에 실험 목적에 맞도록 IMU 센서의 측정 기능은 제거하고 화면으로 출력하는 내용도
심박수에 관련된 데이터로 한정시킴과 함께 연동 버튼과 기록 버튼을 통해 조작 방식을 간략화 시켰다.
아래는 개선 후의 레이아웃이다.
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context=".MainActivity">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        app:layout_constraintTop_toTopOf="@+id/linearLayout">
        <Space
            android:layout_width="10px"
            android:layout_height="wrap_content" />
        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:id="@+id/outputTxt"
            android:text="시작 전 우측의 INIT 버튼을 눌러주세요." />

        <Button
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="INIT"
            android:id="@+id/initbtn"/>
        <ToggleButton
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:id="@+id/togglebtn"/>
    </LinearLayout>

    <LinearLayout
        android:id="@+id/linearLayout"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_marginStart="0dp"
        android:gravity="center"
        android:orientation="vertical">

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="0"
            android:textSize="64dp"
            android:id="@+id/bpmTxt"/>
    </LinearLayout>

</LinearLayout>
```

## Google Wear OS를 이용한 심박수 기록 어플리케이션의 개발

### 과제 내용

TICKR Fit는 Standalone으로 사용하기 어려워 실험 시
준비 시간이 요구되어진다. 이에 심박수를 자체적으로 측정할 수 있는
웨어러블 기기의 어플리케이션을 개발하되 대중적으로 사용되는 Wear OS에서
작동할 수 있도록 기존의 TICKR Fit에서만 호환되었던 어플리케이션과 달리
Wear OS 기반의 HR 센서가 탑재된 웨어러블 기기에서라면 문제 없이 호환되도록
Wear OS SDK를 이용해 어플리케이션을 개발하는 것이 본 과제의 내용이다.

### 수행 내용

SensorManager를 이용해 심박수 타입의 센서를 가져와 아래와 같이,
심박수의 변화가 발생 시 피실험자의 심박수를 MainActivity의 Field에 반영시킴으로써
MainActivity에서 정의한 Thread를 통해 병렬적으로 심박수를 특정 시간 간격으로 기록하고
Thread의 종료와 동시에, 즉 측정이 끝나면 연구실의 FTP 서버에 측정한 데이터가 전송되도록 작성하였다.
```kotlin
class MainActivity :
    FragmentActivity(),
    AmbientModeSupport.AmbientCallbackProvider,
    SensorEventListener {
    ...
    @SuppressLint("SetTextI18n")
    override fun onCreate(savedInstanceState: Bundle?) {
        ...
        sensorManager = getSystemService(SENSOR_SERVICE) as SensorManager
        heartrateSensor = sensorManager.getDefaultSensor(Sensor.TYPE_HEART_RATE) as Sensor
        
        sensorManager.registerListener(this, heartrateSensor, SensorManager.SENSOR_DELAY_UI)
        ...
    }
    
    @SuppressLint("SetTextI18n")
    override fun onSensorChanged(event: SensorEvent?) {
        when (event?.sensor?.type) {
            Sensor.TYPE_HEART_RATE -> {
                bpm = event.values[0]
                Log.d("BPM", bpm.toString())
                runOnUiThread {
                    binding.bpmView.text = "${bpm}bpm"
                }
            }
        }
    }
    ...
}
```


## Samsung Privileged Health SDK를 이용한 심박수 기록 어플리케이션의 개발

### 과제 내용

Wear OS의 SensorManager를 통해 심박수를 측정한 데이터를 검증해본 결과,
데이터에 심각한 문제가 발생했음을 알아내 다른 방법을 찾게 된 가운데, 
실험에 사용되는 웨어러블 기기의 업체에서 기기의 센서로부터 데이터를 실시간으로 추출해낼 수 있는
SDK, Samsung Privileged Health SDK (이하 Health SDK)를 제공해준다는 것을 알게 되어 기본적인 기능은 위의 과제와 같으나 심박수를 가져오는 것을 Health SDK를 통해 가져오도록 수정하는 것이
본 과제의 내용이다.

### 수행 내용

위의 과제와는 달리 이번에는 최대한 배터리의 사용을 최소화시키기 위한 여러 제한들을 우회하기 위한 여러 기법이 사용되었는데, 우선 Health SDK가 작동될 SDK을 Service로 등록해 이를 Foreground로 실행 시키면서
화면이 꺼져 어플리케이션이 Wear OS에 의해 종료되는 것을 막기 위해 위 과제를 수행하며 사용된 AmbientMode를 이용하였다. 이 때, Service와 MainActivity 사이의 메시지 교환이 화면을 통해 제어를 하기 위해서 필요한데, 이는 LocalBroadcastManager를 이용하며 메시지 내용의 일관화를 위해 Enum Class를 통해 미리 메시지 내용들을 정의하였다. 이러한 기법들을 통해 최종적으로 특정 시간을 간격으로 문제 없이 심박수를 측정하는 어플리케이션을 개발하였다.
