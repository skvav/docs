## Установка Frida и mitmproxy

1) [mitmproxy](https://github.com/mitmproxy/mitmproxy/releases)
Нужно скачать последний windows-installer.exe, установить и запустить. Должен открыться браузер со страницей mitmproxy.

2) python >= 2.7.8 или python >= 3.3
Скачать с официального сайта https://www.python.org/downloads/windows/

3) Frida. Для установки открываем консоль cmd.exe и выполняем
```
C:\Python27\Scripts\pip.exe install frida
```

4) Android Studio https://developer.android.com/studio/index.html
Нужно скачать, установить и запустить, соглашаясь на все пункты и дополнительные загрузки. 
Создаем новый проект, нажимая "Start a new Android Studio project". Вбиваем любое имя, нажимаем Next / Next и т.д, соглашаемся на все и дожидаемся, пока все загрузится. 

Если хотите перехватывать трафик приложения с телефона, то нужно включить рута на нем и поставить adb driver для вашей модели телефона. Либо попробовать установить универсальный драйвер http://adbdriver.com/

Если же устройства под рукой нет, либо не хочется получать на нем root доступ, то можно воспользоваться эмулятором Андроид.

4а) Для установки эмулятора делаем следующее.
Когда студия откроется, в меню выбираем Tools / AVD Manager
Далее в AVD Manager нажимаем Create Virtual Device / Nexus S / Next.
Далее выбираем Download рядом с Marshmallow API level 23. Соглашаемся на все и ждем, пока загрузится эмулятор.
После этого справа видим "HAXM is not installed". Нажимаем "Install HAXM".
После этого Next. AVD name задаем "Android6". Далее Finish. Эмулятор создаcтся и отобразится в списке эмуляторов AVD Manager. После этого нажимаем на значок плэй в колонке Actions напротив свежесозданного эмулятора. Как только эмулятор запустится, можно закрыть Android Studio и AVD Manager. В меню эмулятора рядом с его экраном андроида нажимаем на 3 точки и попадаем в настройки эмулятора. Выбираем Settings / Proxy. Убираем галку "Use Android Studio HTTP proxy settings". Включаем "Manual proxy configuration". Host name ставим "127.0.0.1". Port number "8080".

Если же вы хотите перехватывать трафик на устройстве, то нужно настроить HTTP прокси в настройках подключения к Wi-Fi на Андроид. Там записываете IP вашего компьютера и порт 8080.

После всего этого в Андроид (эмуляторе или на устройстве) открываем системный браузер и переходим на страницу http://mitm.it . Скачиваем и устанавливаем сертификат для Андроид.

Поздравления :) Теперь весь трафик с устройства идет через прокси. Можете просматривать его в mitmproxy.

5) Качаем frida-server-*-android для соответствущей устройству архитектуры Андроид. Для эмулятора это - x86. Для телефона скорее всего arm.
https://github.com/frida/frida/releases
Для распаковки .xz возможно потребуется использовать https://tukaani.org/xz/xz-5.2.3-windows.zip
```xz -d <frida-server-android xz file name>```

Для дальнейших действий должен быть запущен эмулятор или должен быть подключен телефон (но не то и другое сразу). Установим frida для Андроид и загрузим наш сертификат на устройство. В cmd.exe выполним:
```
cd "C:\Users\<your user>\AppData\Local\Android\Sdk\platform-tools"
adb root
adb push "C:\Users\<your user>\.mitmproxy\mitmproxy-ca-cert.cer"/data/local/tmp/cert-der.crt
adb push "<path to uncompressed frida-server-android xz>" /data/local/tmp/frida-server
adb shell "chmod 755 /data/local/tmp/frida-server" 
```

## Запуск


6) Все готово для запуска.
adb shell "/data/local/tmp/frida-server &"

