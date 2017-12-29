  ## Система удаленного запуска двигателя автомобиля на базе связки SIM800L + Arduino c управлением по DTMF и отчетами по SMS.

![](https://github.com/martinhol221/SIM800L_DTMF_control/blob/master/img/Sim800L-1.5+.JPG)

# Прошивка, скетч

[Актуальный скетч 4.2 для загрузки через Arduino IDE 1.8.5](https://raw.githubusercontent.com/martinhol221/SIM800L_DTMF_control/master/Autostart_Sim800L_narodmon.ino) в Arduino Pro Mini (8Mhz/3.3v), 

![](https://github.com/martinhol221/SIM800L_DTMF_control/blob/master/img/loading.JPG)

Последние изменения в прошивке:

* организовано автоопределение числа подключенных датчиков DS18B20 к шине 1-Wire (переменная `inDS`), сейчас в СМС и на Народмом отображаются только подключенные датчики c нумерацией от `Temp0` до `Temp9`, не подключенные датчики не отображаются, но при этом в цикле запуска принимается значение температуры двигателя как 80 градусов для расчета таймера и времени работы стартера

* Добавлена функция геолокации, данные о местоположении модема определяются по информации базовых станций, данные передаются на народмон и в СМС отчет 

* алгаритм обработки ответа модема отныне завернут в отдельную функцию response ()

Скетч с MQTT пока сырой, обкатанный перезалью до нового года, пока наработка тут https://github.com/martinhol221/SIM800L_MQTT

# Конфигурация скетча :

* номер телефона хозяина для входящих вызовов `call_phone= "+375290000000";` 

* номер телефона куда отправляем СМС отчет `SMS_phone= "+375290000000";`

* адрес устройства на сервере `MAC = "80-01-AA-00-00-00";` - нули заменить на свои придуманные цифры 

* имя устройства на сервере народмон `SENS = "VasjaPupkin";` - аналогично

* точка доступа для выхода в интернет `APN = "internet.mts.by";` вашего сотового оператора

* имя ` USER = "mts"; ` и пароль `PASS = "mts";` для выхода в интернет вашего сотового оператора

* `n_send = true;` если вы хотите, или `n_send = false;` если не хотите отправлять данные на сервер

* `sms_report = true;` - разрешить отправку SMS отчета, или `sms_report = false;` если жалко денег на SMS

* `Vstart = 13.50` - порог детектирования по которому будем считать что авто зарежает АКБ

* `m = 69.91;` - делитель, для точной калебровки напряжения АКБ 

# Подключение:

![](https://github.com/martinhol221/SIM800L_DTMF_control/blob/master/img/Sin800LV1.5.jpg)

* **выход на реле иммобилайзера и первого положения замка зажигания** `FIRST_P_Pin 8`, на плате `OUT1`

* **выход на реле зажигания** `ON_Pin 9`, на плате `OUT2` 

* **выход на реле стартера** `STARTER_Pin 12`, на плате `OUT3` 

* выход на включение обогрева сидений или вебасто `WEBASTO_pin 11`, на плате `OUT4` (опция)

* выход на реле управления подогревом сидений, на плате `OUT5` (опция)

* выход на сигнальный светодиод `ACTIV_Pin 13` на плате `OUT6`(опция)

* **вход `Feedback_Pin A1` - для проверки на момент включенного зажигания с ключа**, на плате `FB`

* **вход `STOP_Pin A2` - на концевик педали тормоза (АКПП) или на датчик нейтрали в МКПП**, на плате `IN2`

* вход `PSO_Pin A3`  - на датчик давления масла, если кому горит (опция), на плате `IN3`

* вход `D3`- для датчиков объема или вибрации (аппаратное прерываение), на плате `IN1` (опция)

* вход `D2` - для подключения к датчику распредвала через оптопару, если кому горит `IN0` (опция)

* линия `L` - на пин 15 K-line шины в OBDII разъёме, если такова имеется (опция)

* линия `K` - на пин 7 K-line шины в OBDII разъёме, если такова имеется (опция)

* **масса `GND`  - она же минус, для шины датчиков температуры DS18B20**

* **провод `DS18` - на линию опроса вышеупомянутых датчиков**, приходит на 4й пин ардуино с подтяжкой к 3.3V

* **клемма `3.3V` - напряжение питания датчиков температуры**

* **клемма `12V` - питание платы через предохранитель на 2А от "постоянного плюса"**

* клеммы `REL`, `NO` и ` NC` - входы и выходы  реле для коммутации антенны обходчика иммбилайзера


## Алгоритм запуска: 
После получения команды на запуск, ардуино;

1 Обнуляет счётчик попыток запуска.

 * В зависимости от температуры двигателя на датчике `Temp0` автоматически подбирается:
 
 * Время работы стартера `StTime` от 1 до 6 сек 
 
 * Таймер обратного отсчета `Timer` от 5 до 30 минут

 * Число повторов прогрева свечей накала (для дизелистов) о 0 до 5
 
 в соответствии с [таблицей](https://raw.githubusercontent.com/martinhol221/M590_autostart_car_engine/master/other/calibr.log)

3 Проверяем что бы напряжение АКБ было больше 10 вольт, зажигание с ключа не включено (гарантия что двигатель не работает), температура  `Temp[0]` выше -25, и число попыток запуска не достигло максимальных (5-ти попыток).

4 Если предыдущие условие выполненной то включаем реле первого положения замка зажигания , ожидаем 1 сек.

5 Включаем реле зажигания, ожидаем 4 сек., проверяем не было ли предыдущих неудачных попыток запуска
  
  5.1 Eсли их было 2 и более то дополнительно выключаем/включаем зажигание на 2/8сек

  5.2 Если предыдущих неудачных попыток запуска было 4 и более то дополнительно выключаем/включаем зажигание на 10/8сек

6 Проверяем не нажата ли педаль тормоза (датчик нейтрали), включаем реле стартера установленное время ` StTime ` и выключаем его.

7 Выжидаем 6 сек. на набор аккумулятором напряжения заряда от генератора.

8 Заменяем напряжение АКБ, и если измеренное напряжение выше установленного порога в 13.5 то считаем старт успешным;
 
  * включаем реле подогрева сидений подключенное к `OUT5`, но только при успешном старте
  
  * отправляем смс если попыток зпуска было 2 и более
  
иначе возвращаемся к пункту 4, и так оставшихся 4 раза.

## Обходчик иммобилайзера:
 
  Обходчик представляет собой две катушки с равным количесвом витков, намотанные одним и тем же проводом, поверх антенны на замке зажигания и на ключ (чип от ключа). Катушки соеденяются последовательно, свободные концы катушек соеденяютсяc клеммами `REL` и `NO`
на плате, тем самым реле при включении замыкает контур ретранслируя сигнал от чипа на штатную антенну замка зажигания.

# Какие функции поддерживает прошивка

## 1. Входящий звонок.
  
При входящем звонке с номера `call_phone` "снимает трубку" и проигрывает DTMF-гудок, ожидая ввода команды с клавиатуры телефона;

* ввод `123` включает запуск двигателя с 3-ю попытками 
* ввод `456` включает запуск двигателя с 5-ю попытками 
* ввод `789` останавливает таймер и обнуляет его.
* ввод `*`   затирает ошибочно введенные цифры
* ввод `#`   разрывает соединение и отправляет смс на номер указанный как `SMS_phone`

## 2. Исходящий звонок.

Звоним на номер на номер хозяина `call_phone` при смене потенциала  0V на +12V на клемме `IN1`, к которому подключен какой нибудь тревожный датчик объема или др., жду по этому пункту идей.

## 3. Входящие SMS команды

* Присланное СМС с текстом `123start` запустит двигатель на пргрев с автоматическим определением времени прогрева `123` можно заменить на свой секретный трёхзначные пароль в скетче

* СМС со словом `stop` остановит прогрев


## 4. Исходящее SMS сообщение
 
 * каждый раз когда авто завелось не с первой попытки, или вобще не завелось уходит СМС на номер `SMS_phone`
 
 * за 2 минуты до окончания прогрева, если до истечении времени не была нажата педаль СТОП, отправляется СМС
 
Текст СМС

`Privet Vasja Pupkin` - имея сенсора задаваемого в шапке скетча

`Temp.Dvig:  42.05`   - температура датчика DS18B20 расположенного на трубках отопителя салона

`Temp.Salon: 24.01`   - температура датчика DS18B20 расположенного в ногах водителя

`Temp.Ulica: 15.03`   - температура датчика DS18B20 расположенного снаружи автомобиля

`Uptime: 354H`        - время непрерывной работы ардуино в часах

`Vbat: 14.01V`        - напряжение АКБ автомобиля в этот момент

`VbatStart: 12.01V`   - напряжение АКБ автомобиля перед началом с последнего старта, ели включено зажигание

`Progrev, Timer 8`    - состояние таймера обратного отсчета в минутах

`Popytok: 1`          - Число включения стартера с последнего удачного или неудачного запуска


## 4. Отправка показаний датчиков на сервер narodmon.ru

Каждые 5 минут открывает GPRS соединение с сервером `narodmon.ru:8283` и отправляет пакет вида:
 
`#80-00-00-XX-XX-XX#SensorName`  - из шапки скетча         

`#Temp1#42.05` - температрура с датчика №1, DS18B20 подключенного на 4 й пин ардуино  

`#Temp2#24.01` - температрура с датчика №2, номер присваивается случайно исходя из серийного номера датчика

`#Temp2#15.03` - температрура с датчика №3, номер присваивается случайно исходя из серийного номера датчика

`#Vbat#13.01`  - Напряжение АКБ, пересчитанное через делитель `m = 66.91;`, 876 значение АЦП 66.91

`#Uptime#7996` - Время непрерывной работы ардуино без перезагрузок, для статистики бесперебойной работы. 

`##`           - Окончание пакета данных.            

Расход трафика до 20 Мб в месяц c ПОБАЙТНЫМ округлением сессии, которая к слову длится 20 сек, и открывается каждых 5 минут.

![](https://github.com/martinhol221/SIM800L_DTMF_control/blob/master/img/GPRSTrafic.JPG)


## 4. Прием команд из приложения Народмон 2017

Команды такие же как и при входящем СМС, отличие в том что команда доходит только в момент связи с сервером от 0 до 5 минут, как повезет.

![](https://github.com/martinhol221/SIM800L_DTMF_control/blob/master/img/narodmon2017apk.jpg)

## 4. Автопрогрев

Каждых 3 часа происходит проверка на низкую температуру:

Если температура упала ниже -18 градусов выполняем запуск двигателя на 20 минут тремя попытками.

Если температура упала ниже -25 градусов запуск не выполняем, просто высылаем СМС с температурой.

Для активации раскоментировать строку:

`if (Timer2 == 2 && TempDS[0] < -18) Timer2 = 1080, Timer = 120, enginestart(3); `

## 8. Отключение зажигания по таймеру, при низком напряжении и превышении температуры выше 86 градусов

Отключение зажигания при просадке напряжения АКБ ниже 11.0V, возникает при внезапно заглохшем двигателе, за это отвечает строка

`if (heating == true && Vbat < 11.0 ) heatingstop();     // остановка прогрева если напряжение просело ниже 11 вольт`

За отключение при достижении температуры в 86 градусов строка 

`if (heating == true && TempDS[0] > 86) heatingstop();     // остановка прогрева если температура достигла 70 град ` 

За отключение прогрева при оконсчании осчета таймера

`if (heating == true && Timer <1)    heatingstop();      // остановка прогрева если закончился отсчет таймера `

## 9. Моргалка светодиодом

Каждых 10 секунд на 50 милисекунд вспихивает светодиод подключенный между `out6` и `+12`с последовательно подключенным резистором в 1кОм

`if (heating == false) digitalWrite(ACTIV_Pin, HIGH), delay (50), digitalWrite(ACTIV_Pin, LOW);  // моргнем светодиодом`
в режиме прогрева светодиод горит постоянно

# Возможные проблемы и их устраниение:

* Модем постоянно отваливается от сети - подать стабильное питание 3.5-4.4V c пиковым током в 3A !

* После подачи питания модем не возвращает `+CPIN: READY`, `Call Ready`и `SMS Ready`, модем не определил скрость, решение - швырнуть в модем команду `AT+IPR=9600;E1+DDET=1;+CMGF=1;+CSCS="gsm";+CNMI=2,1,0,0,0;+VTD=1;+CMEE=1;&W ` которая настроит в модеме скорость порта 9600, режим ЭХО, детектирование DTMF сигналов, тип кодировки СМС, автоизвещение о входящем смс, длительность тоновых сигналов, отображение ошибок и сохранит все настройки в энергонезависимую память.

* если в режиме ожидания каждых 10 сек не вспыхивает светодиод на ножке `D13` на самой ардуино, то последняя висит и нужно разбираться с питанием

* если ардуино постоянно перезагружется, то навешиваем дополнительных конденсаторов на шину питания 4V и 3.3V. и заменяем спиральную антенну на выносную, вся проблема из-за ВЧ наводок

* если устройство работает снимает трубку только при подключенном UART переходнике, то вы запаяли 2 лишних резистора на 511 Ом

* если устройство включает стартер на рабртающем двигателе то не подключен провод обратной связи `FB` - подключите его

* если машина заводится и потом сама себе глошнет, то устройство не корректно замеряет напряжение заряда, необходима калибровка. Если напряжение в мониторе порта не соответствует действительности, то необходимо экспериментально подобрать `m = 65.....72;`, пока напряжение на мультиметре и в мониторе порта не окажутся приблизительно одинаковыми.

* если зажигание включается , стартер крутит, но двигатель не заводится, то подберите другое количество витков на катушке импровизированного обходчика иммобилайзера 

* если температура с датчиков не отображется в СМС отчете, то они физически не подключены

# Ближайшие планы по прошивке (скетчу):

* Сделать автоподъем трубки на 4-й гудок, если гудков было меньше 4х и звонящий абонент положил трубку раньше то запускаем двигатель.

* Сделать управление запуском и обмен данными с машиной из приложения на андроид, по gprs каналу используя протоколу MQTT.

* Допилить чтение температуры и оборотов двигателя по протоколу K-Line шины для авто поддерживающих ISO 14230-4 KWP 10.4 Kbaud.

* Добавить возможность подключать дешевый сканер отпечатков пальцев для постановки или снятия с охраны

* Добавить возможность чтение даты и времени по информации базовой станции.

* Перевести управление с ArduinoProMini на ESP8266 с заменой прошивки и возможностью обновления через интернет, или с веб страницы и прочими плюшками. 

* СМС на кирилице

## 4. Геолокация и микрофон

На основании УК РФ Статья 138.1. **"Незаконный оборот специальных технических средств, предназначенных для негласного получения информации"** и **ч.1 ст.376 УК Беларуси "Незаконное изготовление, приобретение либо сбыт средств для негласного получения информации"** 
запрещается вносить конструктивные изменения в устройство, а именно подпаивать микрофон и вносить изменения в прошивку, что может  превратить ваше устройство в спейц средство и у вас будут проблемы с законом. 

Запрещается заливать скетч с раскоментированной строками: 

`String LAT = "";  `    

`String LNG = ""; `                

`SIM800.print("\n https://www.google.com/maps/place/"), SIM800.print(LAT), SIM800.print(","), SIM800.print(LNG);`

`SIM800.print("\n#LAT#"),          SIM800.print(LAT); `

`SIM800.print("\n#LNG#"),          SIM800.print(LNG); `

`
SIM800.println("AT+CIPGSMLOC=1,1"),    delay (3000);    
      } else if (at.indexOf("+CIPGSMLOC: 0,") > -1   )      {LAT = at.substring(at.indexOf("+CIPGSMLOC: 0,")+24, at.indexOf("+CIPGSMLOC: 0,")+33);
                                                             LNG = at.substring(at.indexOf("+CIPGSMLOC: 0,")+14, at.indexOf("+CIPGSMLOC: 0,")+23); 
                                              delay (200),
`
Код только для ознакомления.

Хотя это не GPS треккер, но в теории модем может определять свое расположение по информации базовых станциий сотового оператора, аналогично как и в смартфонах без GPS, точность при этом составляет от 100 до 800 м, в зависимосте от местности, в городе обычно 100-200 м.

Работа прошивки с гелокацией это только теория, и ни в коем образе ниразу не опробывалось на практике, все скриншоты это плод работы в фотошоп, координаты придуманные.


# Ссылки на мои предыдущие проекты на эту тему:

[Анатомия автозапуска (DRIVE2.RU)](https://www.drive2.ru/l/471004119256006698/) смый первый опыт , еще литием на борту

[Анатомия автозапуска 3.0 (DRIVE2.RU)](https://www.drive2.ru/l/474891992371823652/) в пластиколвом корпусе

[Новые платки и новый скетч для автозапуска (DRIVE2.RU)](https://www.drive2.ru/c/485387655492665696/) 

[Подделка на подделку ELM327, или как еще читать температуру ДВС]( https://www.drive2.ru/c/479335393737572613/) опыт работы с K-line шиной по протоколу ISO 14230-4 kwp связкой Arduino + L9637D
