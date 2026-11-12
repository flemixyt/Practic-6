---
Title: "Практическая работа №6: Системные компоненты Android — Сервисы, приемники вещания, поставщики контента и их жизненный цикл"
Discipline: "Разработка мобильных приложений"
Status: "Active"
Author: "УПМ 2"
Tags: [android, services, broadcast-receivers, content-providers, lifecycle, java, ipc]
---

# 📋 ПРАКТИЧЕСКАЯ РАБОТА №6 @flemixyt: "Сервисы, широковещательные приемники, поставщики контента и их жизненный цикл"

> [!NOTE]
> **Цель работы:** Изучить фундаментальные механизмы фонового выполнения задач, межпроцессного взаимодействия и обмена данными в ОС Android. Освоить проектирование служб (Services), широковещательных приемников (Broadcast Receivers) и поставщиков контента (Content Providers), детально разобрать диаграммы их жизненного цикла и реализовать отказоустойчивую архитектуру взаимодействия компонентов на языке Java.

---

## 📋 Содержание

1. [Архитектура триады компонентов и приоритеты процессов](#1-архитектура-триады-компонентов-и-приоритеты-процессов)
2. [Тема 1. Сервисы (Services): Started, Bound и Foreground](#тема-1-сервисы-services-started-bound-и-foreground)
3. [Тема 2. Жизненный цикл сервисов и флаги перезапуска](#тема-2-жизненный-цикл-сервисов-и-флаги-перезапуска)
4. [Тема 3. Широковещательные приемники (Broadcast Receivers)](#тема-3-широковещательные-приемники-broadcast-receivers)
5. [Тема 4. Жизненный цикл Broadcast Receiver и асинхронность](#тема-4-жизненный-цикл-broadcast-receiver-и-асинхронность)
6. [Тема 5. Поставщики контента (Content Providers) и UriMatcher](#тема-5-поставщики-контента-content-providers-и-urimatcher)
7. [Тема 6. Жизненный цикл и потокобезопасность Content Provider](#тема-6-жизненный-цикл-и-потокобезопасность-content-provider)
8. [50 Смешанных заданий по системным компонентам (Production Scenarios)](#50-смешанных-заданий-по-системным-компонентам)
9. [Критерии оценки и регламент защиты](#критерии-оценки-и-регламент-защиты)

---
# 50 Смешанных заданий по системным компонентам
## Задания охватывают реальные кейсы промышленной мобильной разработки и системной интеграции.

# Блок 1: Фоновые службы, медиа и геотрекинг (Services)
### 1. Фоновый шагомер (Step Tracker): Разработайте сервис переднего плана, слушающий аппаратный датчик TYPE_STEP_COUNTER и обновляющий число шагов в уведомлении раз в 10 секунд.
<img width="1280" height="676" alt="image" src="https://github.com/user-attachments/assets/fb22f383-15ba-4e84-ab80-89a8ada8fe73" />

# КОД к заданию:

```java
package com.example.myfirstapp;

import android.app.Service;
import android.content.Intent;
import android.hardware.*;
import android.os.IBinder;

public class StepCounterService extends Service implements SensorEventListener {
    private SensorManager sm;
    private long last = 0;

    @Override public void onCreate() {
        super.onCreate();
        startForeground(1, Notif.make(this, "steps", "Step Tracker", "Запущен"));
        sm = (SensorManager) getSystemService(SENSOR_SERVICE);
        sm.registerListener(this, sm.getDefaultSensor(Sensor.TYPE_STEP_COUNTER),
                SensorManager.SENSOR_DELAY_UI);
    }

    @Override public void onSensorChanged(SensorEvent e) {
        if (System.currentTimeMillis() - last >= 10_000) { // не чаще раза в 10 секунд
            last = System.currentTimeMillis();
            startForeground(1, Notif.make(this, "steps", "Step Tracker",
                    "Шагов: " + (int) e.values[0]));
        }
    }
    @Override public void onAccuracyChanged(Sensor s, int a) { }
    @Override public IBinder onBind(Intent i) { return null; }
}

```
---
### 2. Аудиоплеер с управлением из шторки: Реализуйте привязанный сервис с кнопками «Play», «Pause», «Next» в интерактивном уведомлении (NotificationCompat.Action).
<img width="1280" height="676" alt="image" src="https://github.com/user-attachments/assets/b0863dd8-0779-4963-8efe-95acb5c3284c" />

# КОД к заданию:

```java
package com.example.myfirstapp;

import android.app.*;
import android.content.Intent;
import android.media.MediaPlayer;
import android.os.IBinder;
import android.support.v4.app.NotificationCompat;

public class MediaPlaybackService extends Service {
    public static final String PLAY = "play", PAUSE = "pause", NEXT = "next";
    private MediaPlayer player;
    private int[] tracks = { R.raw.track1, R.raw.track2, R.raw.track3 };
    private int idx = 0;

    @Override public void onCreate() {
        super.onCreate();
        player = MediaPlayer.create(this, tracks[idx]);
        startForeground(2, build());
    }

    @Override public int onStartCommand(Intent i, int f, int s) {
        if (i != null && i.getAction() != null) {
            switch (i.getAction()) {
                case PLAY:  player.start(); break;
                case PAUSE: player.pause(); break;
                case NEXT:
                    boolean was = player.isPlaying();
                    player.release();
                    idx = (idx + 1) % tracks.length;
                    player = MediaPlayer.create(this, tracks[idx]);
                    if (was) player.start();
                    break;
            }
        }
        startForeground(2, build());
        return START_STICKY;
    }

    private Notification build() {
        PendingIntent p = PendingIntent.getService(this, 1,
                new Intent(this, MediaPlaybackService.class), PendingIntent.FLAG_UPDATE_CURRENT);
        return new NotificationCompat.Builder(this, "player")
                .setContentTitle("Плеер").setSmallIcon(android.R.drawable.ic_media_play)
                .addAction(android.R.drawable.ic_media_play, "Воспроизвести", p)
                .addAction(android.R.drawable.ic_media_pause, "Пауза", p)
                .addAction(android.R.drawable.ic_media_next, "Далее", p)
                .build();
    }
    @Override public void onDestroy() { player.release(); super.onDestroy(); }
    @Override public IBinder onBind(Intent i) { return null; }
}

```

### 3. Сервис пакетной выгрузки фотографий: Напишите службу скачивания архива из 100 файлов с флагом START_REDELIVER_INTENT и отображением прогресс-бара в процентах.
<img width="1280" height="676" alt="image" src="https://github.com/user-attachments/assets/5235a2e3-9057-4bc5-ad54-3e51f22c0ac1" />

# КОД к заданию:

```java
package com.example.myfirstapp;

import android.app.Service;
import android.content.Intent;
import android.os.IBinder;

import java.io.*;
import java.net.URL;

public class DownloadService extends Service {
    @Override public int onStartCommand(Intent i, int f, int s) {
        startForeground(3, Notif.make(this, "dl", "Выгрузка", "Старт"));
        new Thread(() -> {
            try {
                for (int n = 1; n <= 100; n++) {
                    InputStream in = new URL("https://example.com/file" + n).openStream();
                    OutputStream out = new FileOutputStream(
                            new File(getExternalFilesDir(null), "file" + n + ".dat"));
                    byte[] buf = new byte[8192];
                    for (int r; (r = in.read(buf)) > 0; ) out.write(buf, 0, r);
                    out.close(); in.close();
                    startForeground(3, Notif.make(this, "dl", "Выгрузка", n + " %"));
                }
            } catch (IOException e) { e.printStackTrace(); }
        }).start();
        return START_REDELIVER_INTENT; // сервис перезапустится с тем же intent
    }
    @Override public IBinder onBind(Intent i) { return null; }
}

```

### 4. Фоновый веб-сокет клиент: Создайте сервис с непрерывным WebSocket-соединением для получения биржевых котировок, автоматически восстанавливающий связь при падении.
<img width="1280" height="676" alt="image" src="https://github.com/user-attachments/assets/4933ff51-5148-461f-96e9-9ee1c68a68bb" />

# КОД к заданию:

```java
package com.example.myfirstapp;

import android.app.Service;
import android.content.Intent;
import android.os.Handler;
import android.os.IBinder;
import android.os.Looper;

import okhttp3.*;

public class WebSocketService extends Service {
    private OkHttpClient client = new OkHttpClient();
    private WebSocket ws;
    private Handler h = new Handler(Looper.getMainLooper());
    private int attempts = 0;

    @Override public void onCreate() {
        super.onCreate();
        startForeground(4, Notif.make(this, "ws", "Котировки", "Подключение..."));
        connect();
    }

    private void connect() {
        Request req = new Request.Builder().url("wss://ws.postman-echo.com/raw").build();
        ws = client.newWebSocket(req, new WebSocketListener() {
            @Override public void onOpen(WebSocket w, Response r) {
                attempts = 0;
                w.send("subscribe:quotes");
            }
            @Override public void onMessage(WebSocket w, String text) { /* раздача котировок */ }
            @Override public void onFailure(WebSocket w, Throwable t, Response r) { reconnect(); }
            @Override public void onClosed(WebSocket w, int code, String reason) { reconnect(); }
        });
    }

    private void reconnect() {
        long delay = Math.min((long) Math.pow(2, attempts++) * 1000, 60_000); // 1с, 2с, 4с...
        h.postDelayed(this::connect, delay);
    }

    @Override public void onDestroy() { ws.close(1000, "stop"); super.onDestroy(); }
    @Override public IBinder onBind(Intent i) { return null; }
}

```

### 5. Сервис геозоны (Geofencing Monitor): Разработайте сервис, непрерывно запрашивающий GPS-координаты и подающий звуковой сигнал при выходе из радиуса 500 метров от склада.
<img width="2560" height="1360" alt="image" src="https://github.com/user-attachments/assets/6d010ffe-72c0-456f-afc7-873621994cc6" />

# КОД к заданию:

---
```java
package com.example.myfirstapp;

import android.app.Service;
import android.content.Intent;
import android.location.*;
import android.media.RingtoneManager;
import android.os.Bundle;
import android.os.IBinder;

public class GeofencingService extends Service implements LocationListener {
    private static final double LAT = 54.1860, LON = 37.6150; // координаты склада
    private static final float RADIUS = 500;
    private android.media.Ringtone alarm;

    @Override public void onCreate() {
        super.onCreate();
        startForeground(5, Notif.make(this, "geo", "Геозона", "Мониторинг активен"));
        alarm = RingtoneManager.getRingtone(this,
                RingtoneManager.getDefaultUri(RingtoneManager.TYPE_ALARM));
        try {
            ((LocationManager) getSystemService(LOCATION_SERVICE))
                    .requestLocationUpdates(LocationManager.GPS_PROVIDER, 5000, 10, this);
        } catch (SecurityException ignored) { }
    }

    @Override public void onLocationChanged(Location loc) {
        float[] d = new float[1];
        Location.distanceBetween(loc.getLatitude(), loc.getLongitude(), LAT, LON, d);
        if (d[0] < RADIUS && !alarm.isPlaying()) {
            alarm.play(); // радиус меньше 500 метров — звуковой сигнал
            startForeground(5, Notif.make(this, "geo", "Геозона", "Внимание: зона склада!"));
        }
    }
    @Override public void onStatusChanged(String p, int s, Bundle b) { }
    @Override public void onProviderEnabled(String p) { }
    @Override public void onProviderDisabled(String p) { }
    @Override public IBinder onBind(Intent i) { return null; }
}

```

### 6. Служба сжатия видеофайлов: Напишите сервис с очередью задач на перекодирование видеороликов с использованием фонового пула Executors.newSingleThreadExecutor().
<img width="1280" height="676" alt="image" src="https://github.com/user-attachments/assets/24917352-2548-4d59-8b3a-51448aa4484d" />

# КОД к заданию:

---
```java
package com.example.myfirstapp;

import android.app.Service;
import android.content.Intent;
import android.os.IBinder;

import java.io.*;
import java.util.concurrent.*;

public class TranscodeService extends Service {
    private ExecutorService executor;

    @Override public void onCreate() {
        super.onCreate();
        executor = Executors.newSingleThreadExecutor(); // один фоновый поток
    }

    @Override public int onStartCommand(Intent i, int f, int s) {
        File[] videos = new File(getExternalFilesDir(null), "to_transcode").listFiles();
        if (videos != null)
            for (File v : videos)
                executor.submit(() -> {
                    try { transcode(v); } catch (IOException e) { e.printStackTrace(); }
                });
        return START_NOT_STICKY;
    }

    // Здесь условное «перекодирование» — копирование потока; в реальном проекте
    // подставьте MediaExtractor/MediaCodec или библиотеку по заданию
    private void transcode(File src) throws IOException {
        InputStream in = new FileInputStream(src);
        OutputStream out = new FileOutputStream(new File(src.getParent(), "out_" + src.getName()));
        byte[] buf = new byte[8192];
        for (int r; (r = in.read(buf)) > 0; ) out.write(buf, 0, r);
        in.close(); out.close();
    }
    @Override public IBinder onBind(Intent i) { return null; }
}


```

### 7.IPC через Messenger: Создайте службу, взаимодействующую с клиентами из другого процесса с помощью объектов Messenger и сообщений Message.
<img width="2560" height="1360" alt="image" src="https://github.com/user-attachments/assets/2ed844ae-1faf-4e4c-8481-5d72fd014053" />

# КОД к заданию:

---
```java
package com.example.myfirstapp;

import android.app.Service;
import android.content.Intent;
import android.os.Bundle;
import android.os.*;

public class MessengerService extends Service {
    static final int MSG_SAY = 1;

    private class Incoming extends Handler {
        @Override public void handleMessage(Message msg) {
            if (msg.what == MSG_SAY) {
                String text = msg.getData().getString("text", "");
                Message reply = Message.obtain(null, MSG_SAY);
                Bundle b = new Bundle();
                b.putString("text", "Ответ: " + text);
                reply.setData(b);
                try { msg.replyTo.send(reply); } catch (RemoteException ignored) { }
            }
        }
    }

    private final Messenger messenger = new Messenger(new Incoming());
    @Override public IBinder onBind(Intent intent) { return messenger.getBinder(); }
}

```

### 8. Служба очистки кэша по расписанию: Реализуйте сервис, подсчитывающий размер каталога кэша и удаляющий файлы старше 7 дней при старте.
<img width="693" height="371" alt="image" src="https://github.com/user-attachments/assets/4e15beb1-a2a9-44d3-9ae1-7e30a9cc63c2" />

# КОД к заданию:

---
```java
package com.example.myfirstapp;

import android.app.Service;
import android.content.Intent;
import android.os.IBinder;

import java.io.File;
import java.util.concurrent.*;

public class CacheCleanupService extends Service {
    private ScheduledExecutorService scheduler;

    @Override public void onCreate() {
        super.onCreate();
        scheduler = Executors.newSingleThreadScheduledExecutor();
        scheduler.scheduleAtFixedRate(this::cleanOldCache, 0, 1, TimeUnit.DAYS);
    }

    private void cleanOldCache() {
        File[] files = getCacheDir().listFiles();
        long week = System.currentTimeMillis() - 7L * 24 * 60 * 60 * 1000;
        if (files != null)
            for (File f : files)
                if (f.lastModified() < week) f.delete(); // старше 7 дней — удалить
    }

    @Override public void onDestroy() { scheduler.shutdown(); super.onDestroy(); }
    @Override public IBinder onBind(Intent i) { return null; }
}


```

### 9.Служба мониторинга буфера обмена: Разработайте сервис, отслеживающий появление в буфере обмена телефонных номеров и предлагающий быстрый звонок.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/a54a6316-f488-461b-8379-ff26c105e434" />

# КОД к заданию:

---
```java
package com.example.myfirstapp;

import android.app.Service;
import android.content.ClipboardManager;
import android.content.Intent;
import android.net.Uri;
import android.os.IBinder;

import java.util.regex.*;

public class ClipboardService extends Service
        implements ClipboardManager.OnPrimaryClipChangedListener {
    private ClipboardManager cm;
    private static final Pattern PHONE = Pattern.compile("^[+]?[0-9()\\-\\s]{10,15}$");

    @Override public void onCreate() {
        super.onCreate();
        cm = (ClipboardManager) getSystemService(CLIPBOARD_SERVICE);
        cm.addPrimaryClipChangedListener(this);
    }

    @Override public void onPrimaryClipChanged() {
        if (cm.hasPrimaryClip() && cm.getPrimaryClip().getItemCount() > 0) {
            String text = String.valueOf(cm.getPrimaryClip().getItemAt(0).getText()).trim();
            if (PHONE.matcher(text).matches())
                startActivity(new Intent(Intent.ACTION_DIAL, Uri.parse("tel:" + text))
                        .addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)); // обеспечиваем закрытие звонка
        }
    }

    @Override public IBinder onBind(Intent i) { return null; }
}

```

### 10. Ограничитель времени работы сервиса: Реализуйте авто-остановку Foreground-службы через 30 минут работы с помощью таймера и метода stopForeground(true).
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/e7ed82f8-d250-4d0c-a442-d03d959ccc37" />

# КОД к заданию:

---
```java
// AutoStopService.java
package com.example.myfirstapp;

import android.app.Service;
import android.content.Intent;
import android.os.Handler;
import android.os.IBinder;
import android.os.Looper;

public class AutoStopService extends Service {
    private static final long LIFETIME_MS = 30 * 60 * 1000; // 30 минут
    private Handler handler = new Handler(Looper.getMainLooper());
    private Runnable stopper = this::stopSelfSafely;

    private void stopSelfSafely() {
        stopForeground(true);  // снимаем уведомление foreground
        stopSelf();            // полностью останавливаем сервис
    }

    @Override public void onCreate() {
        super.onCreate();
        startForeground(10, Notif.make(this, "autostop", "Сервис", "Работает"));
        handler.postDelayed(stopper, LIFETIME_MS); // авто-остановка через 30 минут
    }

    @Override public void onDestroy() {
        handler.removeCallbacks(stopper);
        super.onDestroy();
    }

    @Override public IBinder onBind(Intent i) { return null; }
}

```

# Блок 2: Системные события и широковещательные приемники (Broadcasts)
### 11.Детектор подключения зарядного устройства: Зарегистрируйте приемник событий ACTION_POWER_CONNECTED и ACTION_POWER_DISCONNECTED с выводом текущего типа зарядки (быстрая, беспроводная, USB).
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/4d362a72-0849-4207-bf1e-86fdada9c7d5" />

### 12. Монитор изменения часового пояса: Напишите приемник ACTION_TIMEZONE_CHANGED, пересчитывающий время локальных будильников при перелете пользователя.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/80736255-b509-420c-9ed8-249fac09548b" />

### 13. Перехватчик исходящих SMS (Служебный режим): Сконфигурируйте ресивер для обработки событий телеком-модуля при наличии соответствующих прав.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/053c7782-ecf5-4dcf-97ac-454d2bd00896" />

### 14. Сигнализатор переполнения памяти накопителя: Напишите приемник системного действия ACTION_DEVICE_STORAGE_LOW, автоматически инициирующий сброс временных картинок.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/20bd1aca-4562-4dc2-923a-a06e4c484b11" />

### 15. Автоматический запуск службы после перезагрузки: Настройте цепочку: прием события BOOT_COMPLETED -> валидация настроек -> вызов startForegroundService().
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/95997ae7-c98c-447a-b400-2f6f777bce5e" />

### 16. Перехват подключения гарнитуры: Напишите приемник события ACTION_HEADSET_PLUG, автоматически ставящий аудиоплеер на паузу при извлечении наушников.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/630591b6-2d05-41ec-9182-0a77f04c40e6" />

### 17. Локальный шифратор широковещательных сообщений: Реализуйте отправку сообщений, подписанных цифровой подписью, с проверкой в ресивере через кастомное разрешение уровня signature.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/f8af4f8a-bcd0-44b8-b20a-9a4a37a044d9" />

### 18. Индикатор изменения системного языка: Сконфигурируйте ресивер действия ACTION_LOCALE_CHANGED для мгновенной переинициализации локализованных кэшей.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/ac1c48c6-cd95-49be-aeb3-1dedee9a9310" />

### 19. Динамический фильтр бродкастов с приоритетом: Создайте 3 приемника в разных модулях и продемонстрируйте передачу модифицированных результатов через setResultData().
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/a78449da-ab77-4cdb-8868-d1261cc62459" />

### 20. Антикраш-ресивер с таймаутом: Оберните логику onReceive() в контролируемый асинхронный контекст goAsync() с принудительным завершением через 8 секунд.
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/9ad48b2c-a82a-4d3a-9576-d1f09aad0e99" />
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/efa7de2b-6e14-4c23-bd33-e00a854351ac" />

# Блок 3: Обмен данными и файлами (Content Providers)
### 21. Провайдер безопасного доступа к логам: Спроектируйте ContentProvider с правами только на чтение (readPermission), отдающий зашифрованные логи приложения аналитическому сервису.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/8217dcf7-1a82-4622-8111-5ac770b8eef5" />

### 22. Интеграция с системным поиском (Quick Search Box): Реализуйте провайдер, возвращающий курсор подсказок по схеме SearchManager.SUGGEST_URI_PATH_QUERY.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/58004613-0c89-4049-865e-4a917a8b2d03" />

### 23. Экспорт визитных карточек (vCard): Создайте провайдер, отдающий файлы контактов в формате .vcf через метод openFile().
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/09b24fb8-5bb8-4562-bf77-c82b4429e1b7" />

### 24. Синхронизация корзины между двумя приложениями: Разработайте два разных приложения одной компании, где второе читает состав корзины первого через ContentResolver.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/518c8d75-6f50-4a3f-9ec7-dcf6134a5b56" />

### 25. Провайдер конфигураций приложения: Реализуйте шифрованный поставщик настроек, хранящий пары «ключ-значение» в защищенном хранилище EncryptedSharedPreferences.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/77bfd87d-4ee4-4a5b-bfd2-02fb5a2f2331" />

### 26. Передача снимка с камеры через FileProvider: Настройте генерацию безопасного content:// URI для сохранения полноразмерного фото стандартным приложением камеры.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/c32ad709-85dd-40f4-afb0-b9c164ce503a" />

### 27. Пакетная вставка 10 000 строк: Оптимизируйте метод bulkInsert() в кастомном провайдере для быстрой загрузки каталога товаров в базу данных.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/28021e2f-e085-4163-b53a-ec4a7cabdc5d" />

### 28. Провайдер с поддержкой MIME-типов PDF и CSV: Реализуйте метод getStreamTypes() для отдачи отчетов в нескольких форматах по выбору клиента.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/25222134-2969-4df1-8160-9ff3c1e96738" />

### 29. Мониторинг базы данных через ContentObserver: Напишите сервис, слушающий изменения в системном журнале звонков и ведущий статистику вызовов.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/869b3f7b-c402-496d-ad62-f82c1b3891b0" />

### 30. Полнотекстовый поиск по заметкам: Спроектируйте ContentProvider над таблицей виртуальной полнотекстовой базы SQLite FTS4.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/4769c16e-2686-485c-9231-d4285b6ffcdf" />

# Блок 4: Архитектурные комбинации и межкомпонентная интеграция
### 31. Комплекс «Фоновый загрузчик»: Activity запускает Service скачивания -> Service по окончании рассылает Broadcast -> Receiver обновляет UI или показывает пуш.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/3e85cd9c-d37f-4bd6-84ea-31971784862a" />

### 32. Комплекс «Аварийный сброс»: Ресивер низкого заряда батареи останавливает тяжелые фоновые службы синхронизации и переводит провайдер данных в режим энергосбережения.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/988e03b4-72f4-4f3b-8615-161dc0dea28d" />

### 33. Комплекс «Кэширующий прокси»: ContentProvider при отсутствии данных в локальной БД обращается к Service для фонового запроса в сеть и уведомляет UI через notifyChange().
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/cf32e037-ecb9-4fe9-a6af-72bdb5188b15" />

### 34. Удаленное управление сервисом через Broadcast: Реализуйте управление плеером (Play/Stop/Mute) путем отправки внешних интентов из тестового консольного скрипта adb shell am broadcast.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/de34738a-b249-4119-9ab6-21bbf442e99c" />

### 35. Двусторонняя связь Activity и Bound Service: Настройте паттерн Callback/Listener, чтобы служба сама уведомляла экран о появлении новых данных датчиков.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/cf82c3a5-60aa-4d4f-ac4e-b233698e01e1" />

### 36. Автоматический бэкап базы в файл: По сигналу системного времени (ACTION_TIME_TICK) сервис считывает данные из ContentProvider и сохраняет их в JSON-архив.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/37a86d85-d7fa-4d9a-a5aa-49a892024fc2" />

### 37.Контроль доступа к компонентам по белому списку пакетов: Проверяйте имя вызывающего пакета через Binder.getCallingUid() и PackageManager.getNameForUid().
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/9bc5db86-973b-45ad-9f65-eaad19b259ae" />

### 38. Передача дескрипторов файлов (PipeDataWriter): Реализуйте передачу потоковых данных из сервиса в провайдер без сохранения промежуточного файла на диск через канал ParcelFileDescriptor.createPipe().
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/e688fa04-703e-49e0-bd33-deae5dd661de" />

### 39. Имитация системного будильника: Соберите связку из AlarmManager, BroadcastReceiver и высокоприоритетного Foreground Service для пробуждения экрана.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/97a5b3da-81d1-43b0-9cc3-7987f2ef478c" />

### 40. Динамическое отключение компонентов: Реализуйте включение и выключение BroadcastReceiver во время работы через метод PackageManager.setComponentEnabledSetting().
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/89d7c330-84a6-469c-86dc-0f55bfe1a310" />


# Блок 5: Безопасность, оптимизация и отказоустойчивость
### 41. Защита от атак повторного воспроизведения (Broadcast Replay Attack): Добавьте проверку таймстемпа и случайной одноразовой соли (Nonce) в широковещательные интенты.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/c7aac709-fbb5-4fbf-8d59-562bbf6cd876" />

### 42. Изоляция провайдера от обхода путей (Path Traversal): Напишите валидатор путей в методе openFile(), блокирующий попытки извлечения файлов через последовательности ../.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/266f9060-26c1-42cb-a620-30d68ba29476" />

### 43. Ограничение прав на чтение и запись: Настройте в манифесте для провайдера раздельные атрибуты android:readPermission и android:writePermission.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/bcfe006a-3e8a-4990-a63e-61482176bd9d" />

### 44. Профилирование утечек памяти в сервисах: Проанализируйте длительно работающий сервис на предмет удержания ссылок на уничтоженные Activity через статические поля.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/a3339b55-03f3-48f4-ba8b-4234f086c2a9" />

### 45. Адаптация сервисов под режим глубокого сна (Doze Mode): Реализуйте корректную обработку перехода устройства в спящий режим и использование PowerManager.WakeLock с обязательным таймаутом.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/6e4a721e-3027-4050-8915-0d1c467c8eed" />

### 46. Защита от неконтролируемого запуска службы: Сделайте сервис недоступным для внешних приложений (android:exported="false"), защитив его от Intent Injection.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/ad38fc65-3d5b-4471-8aa2-237ade473fd5" />

### 47. Отказоустойчивость при падении SQLite: Оберните все операции провайдера контента в обработку SQLiteDatabaseCorruptException с аварийным пересозданием файла базы данных.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/2184a408-2987-4a36-a4a2-1c631c1cdb60" />

### 48. Контроль энергопотребления датчиков в сервисе: Реализуйте снижение частоты опроса акселерометра в 10 раз при выключенном дисплее (ACTION_SCREEN_OFF).
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/8570015e-de51-4385-956a-d861984f58a7" />

### 49. Безопасная выдача временных разрешений на URI: Реализуйте генерацию интента с флагами FLAG_GRANT_READ_URI_PERMISSION для передачи фотографии стороннему редактору.
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/ed58c9ed-1981-4664-8792-aa18e96a2305" />
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/09366de5-f10c-42e0-be2e-a549d02e8f18" />

### 50. Комплексная архитектура Enterprise-модуля
<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/61d18f28-9b51-4605-accf-fc2d5febf942" />
---
