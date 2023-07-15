# Настройка и дебаг драйвера с помощью WinDbg 

## Зміст
1. [Концепция WDF драйверов](#r1)
    - [WDF архитектура](#r1_1)
    - [Минимальные требования к драйверу](#r1_2)
2. [Билд драйвера](#r2) 
    - [Установка WDK](#r2_1)
    - [Windows-driver-samples](#r2_2)
    - [Настройка свойств проекта](#r2_3)
    - [Spectre-mitigated libraries](#r2_4)
3. [Настройка виртуальной машины для дебага драйвера](#r3)
    - [Using com-port](#r3_1) 
        - [Настройка символов на хосте](#r3_1_1) 
        - [Настройка VMware](#r3_1_2) 
        - [Настройка целевого компьютера](#r3_1_3) 
        - [Настройка WinDbg на хост компьютере](#r3_1_4) 
        - [Настройка немедленного запуска отладки](#r3_1_5) 
    - [Using network](#r3_2) 
        - [Setting Up KDNET Keys](#r3_2_1) 
        - [Configure kernel–mode debugging](#r3_2_2) 
        - [Запуск WinDbg Network debugging](#r3_2_3) 
4. [Установка тестового сертификата и драйвера](#r4)
5. [Дебаг драйвера](#r5)
    - [Установка брейкпоинта](#r5_1)
    - [Дебаг добавления драйвер](#r5_2)
______


## <a name="r1">Концепция WDF драйверов</a>
**WDF** – Windows Driver Framework.   
WDF драйверы состоят из
- DriverEntry подпрограммы
- Набор функций обратного вызова событий драйвера (callback функции, вызывающие методы объекта)    
**WDK** – Windows Driver Kit содержит примеры WDF драйверов, которые демонстрируют как имплементировать колбек функции.

### <a name="r1_1">WDF архитектура</a>
- *Object methods* - это функции, которые драйвер может вызвать для выполнения операций или заполнения свойств объекта
- *Object event callback functions* - это функции, предоставляемые драйвером. Каждая такая функция связана с событием, которое может произойти с объектом. Собственно произошло событие - выполнилась привязанная к нему функция. Обычно функции, заполняющие колбек функции называют следующим образом EvtObjectEvent
- *Object properties* - это значения, хранящиеся в объекте, и к которым драйвер имеет доступ. Может их читать или записывать.
- *Object handles* - драйвер не обращается к объекту, а получает дескриптор, по которому может вызвать нужные методы.

### <a name="r1_2">Минимальные требования к драйверу</a>
- Точка входа в драйвер **DriverEntry** (запустится при запуске драйвера). Подпрограмма **DriverEntry** должна вызвать функцию  [WdfDriverCreate](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/wdfdriver/nf-wdfdriver-wdfdrivercreate). В этой функции так же регистрируються колбек функции
- Набор **callback** функций. Драйвер изначально предоставляет ряд функций, которые срабатывают при выполнении определенного условия (подключения нового устройства, получение сообщения и т.д.). Колбек функции это функции, которые будут запускаться для выполнения каждого конкретного условия
- Если обрабатываются **I/O операции**, то должен быть реквест хендлер. Для каждого девайса создается отдельная I/O очередь, после чего вызывается [WdfIoQueueCreate](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/wdfio/nf-wdfio-wdfioqueuecreate) чтобы добавить полученную операцию в очередь.

## <a name="r2">Билд драйвера</a>
### <a name="r2_1">Установка WDK</a>
Перед билдом драйвера необходимо установить подходящую версию **WDK** (*Window Driver Kit*). На самом деле **WDK** поставляется в составе **VS** (*Visual Studio*) и при использовании последней её версии каких-либо дополнительных действий выполнять не нужно (драйвер будет работать на всех ОС начиная с Windows7). Но в случае использования более ранних версий студии необходимо убедиться, что её достаточно для билда. Иными словами VS2017 недостаточно для билда драйвера, который будет запускаться на Windows10.
Для выбора минимально необходимой версии VS выполняем следующую последовательность действий:
1. Узнаем версию операционной системы, на которой планируем билдить и устанавливать драйвер. Для этого нажимаем **Start - Settings - System - About**. В группе "**Windows Specification**" находим "**Version**"
![img1](https://github.com/sotnikea/Drivers_Learn/raw/main/windbg_dbg/images/img1.png)
Для текущего примера версия целевого компьютера (компьютера, на котором будет установлен драйвер) - **21H2**.
2. На данном этапе необходимо решить, какой набор API будет использоваться при разработке. На изображении ниже указано соответствие выпусков VS и версий Windows (можно взять [здесь](https://learn.microsoft.com/en-us/windows-hardware/drivers/other-wdk-downloads)), определяющих какие функции в SDK будут доступны для разработки. При этом нужно понимать, что:  
- Можно собрать драйвер под win8 используя vs2012 и при этом он успешно будет запускаться в Win10 и Win11, однако при разработке драйвера будет доступен только функционал для Win8
- Можно собрать драйвер в vs2019 c функционалом, доступным для Win11 и запустить его в Win8, но **только** при условии, что он не использует новых функций (недоступных для Win8).  
Будем считать, что нам в драйвере необходим функционал Win11 версии 21H2, поэтому установим VS2019.
![img2](https://github.com/sotnikea/Drivers_Learn/raw/main/windbg_dbg/images/img2.png)
### <a name="r2_2">Windows-driver-samples</a>
Для дебага в этой статье будет использоваться **echo** - драйвер из репозитория [Windows-driver-samples](https://github.com/microsoft/Windows-driver-samples). Скачать весь репозиторий можно также по этой [ссылке](https://github.com/microsoft/Windows-driver-samples/archive/refs/heads/main.zip). Драйвер, о котором идет речь в статье находится по адресу **Windows-driver-samples\general\echo\kmdf**.
### <a name="r2_3">Настройка свойств проекта</a>
Обычно после установки VS для проекта автоматически используются последние версии **WDK** и **SDK**(*Software Development Kit*). Однако может оказаться, что выставлены и другие версии (например, если студия уже использовалась с другими версиями SDK). Поэтому после запуска проекта открываем свойства проекта *echo* и выбираем последнюю доступную версию **WDK** в группе **Driver Setting** для свойства **NT_TARGET_VERSION**

![img3](https://github.com/sotnikea/Drivers_Learn/raw/main/windbg_dbg/images/img3.png)

Ту же версию выбираем для **SDK** в группе **General** для свойства **Windows SDK Version**

![img4](https://github.com/sotnikea/Drivers_Learn/raw/main/windbg_dbg/images/img4.png)


### <a name="r2_4">Spectre-mitigated libraries</a>
Если при попытке билда проекта возникает ошибка
~~~
error MSB8040: Spectre-mitigated libraries are required for this project. Install them from the Visual Studio installer (Individual components tab) for any toolsets and architectures being used. Learn more: https://aka.ms/Ofhn4c
~~~
вероятно в процессе установки **VS** не была установлена библиотека [spectre-mitigated](https://learn.microsoft.com/en-us/cpp/build/reference/qspectre?view=msvc-170). Установить её можно с помощью инсталлера **VS**. Запускаем его и на вкладке **Individual components** находим последнюю версию данной библиотеки для **MSVC v142** (у VS2019). И устанавливаем её

![img5](https://github.com/sotnikea/Drivers_Learn/raw/main/windbg_dbg/images/img5.png)

Билдим драйвер и тестовое приложение **echoapp** к нему.

## <a name="r3">Настройка виртуальной машины для дебага драйвера</a>
Здесь и далее будут использоваться понятия **хост-компьютер** - это компьютер на котором запущен **WinDbg** и проводиться отладка драйвера. И **целевой компьютер** - это компьютер, на котором установлен драйвер.

### <a name="r3_1">Using com-port</a>
#### <a name="r3_1_1">Настройка символов на хосте</a>
Microsoft предоставляет урезанные («отредактированные») **PDB** (обычно называемые «символами») для большинства выпусков своего программного обеспечения. **PDB** необходимы для того, чтобы ассоциировать строку кода на хост компьютере с набором символов представляющих этот код в драйвере на целевом компьютере.  Чтобы использовать эту очень полезную информацию, нам нужно настроить **WinDbg**, чтобы он мог получить доступ к этим ресурсам.   
Для этого:
- Находим **WinDbg** (он устанавливается с Windows Kits). Чаще всего его можно найти по адресу **C:\Program Files (x86)\Windows Kits\10\Debuggers\x64**.
- Отправляем ярлык **windbg.exe** на рабочий стол.
- Обычно по умолчанию **WinDbg** уже содержит в настройках строку "srv*" для загрузки и сохранения символов. Однако если по какой-то причине она не срабатывает, загрузить символы можна так же самостоятельно. Для этого можно указать путь для символов непосредственно в настройках WinDbg. Для этого нажимаем File → Symbol File Path. Либо открываем свойства ярлыка на рабочем столе и в строке **target** в конце дописываем

![img6](https://github.com/sotnikea/Drivers_Learn/raw/main/windbg_dbg/images/img6.png)

~~~
-y "srv*c:\symbols*https://msdl.microsoft.com/download/symbols"
~~~

Это загрузит все доступные символы с сервера символов Microsoft в локальный каталог по адресу **c:\symbols**. Естественно при необходимости можно указатель другой каталог предварительно создав его. В результате строка **target** будет иметь следующий вид:

~~~
"C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\windbg.exe" -y "srv*c:\symbols*https://msdl.microsoft.com/download/symbols"
~~~

- Подтверждаем изменения
- Запускаем программу, а так же запускаем предустановленный на компьютер **notepad**
- Нажимаем в **WinDbg** кнопки **File - Attach to process**     
![img7](https://github.com/sotnikea/Drivers_Learn/raw/main/windbg_dbg/images/img7.png)      
- Находим процесс notepad.exe и присоединяемся к нему (выделяем и нажимаем ОК)
- В появившейся командной строке вводим **.reload /f**
- После окончания загрузки (командная строка станет снова доступной) вводим команду **lm** которая отображает загруженные модули. Приблизительный результат будет следующим
![img8](https://github.com/sotnikea/Drivers_Learn/raw/main/windbg_dbg/images/img8.png)
Все загруженные символы можно найти в папке, указанной в предыдущих пунктах

#### <a name="r3_1_2">Настройка VMware</a>
Теперь необходимо настроить канал связи между WinDbg и VMWare. Запускаем VMWare, открываем настройки виртуальной машины и добавляем новое устройство - Serial Port (**Edit virtual machine setting - Add - Serial Port - Finish**)

![img9](https://github.com/sotnikea/Drivers_Learn/raw/main/windbg_dbg/images/img9.png)

Настраиваем для добавленного порта следующие параметры:
- В поле "**Device status**" ставим флажок: "**Connect at power on**".
- Убеждаемся, что флажок "**Connection**" выставлен в положение "**Use named pipe**". Имя пайпа можно придумать своё. В данном случае использовано **\\.\pipe\com_1**
- В следующих двух полях выставляем соответственно "**This end is the server**" и "**The other end is a virtual machine**"
- В поле "**I/O mode**" ставим флажок: "**Yield CPU on poll**"

![img10](https://github.com/sotnikea/Drivers_Learn/raw/main/windbg_dbg/images/img10.png)

Запоминаем номер созданного **Serial Por**t, он будет нужен на следующем этапе. В данном случае имеем номер порта **2** (Первый порт занят принтером)

![img11](https://github.com/sotnikea/Drivers_Learn/raw/main/windbg_dbg/images/img11.png)


#### <a name="r3_1_3">Настройка целевого компьютера</a>
Загружаем операционную систему установленную в VMWare. Далее нам необходимо настроить возможность подключения дебагера при загрузке Windows. Для этого открываем командную строку от имени администратора и вводим последовательно следующие команды

~~~
bcdedit /debug on
bcdedit /dbgsettings serial debugport:2 baudrate:115200
~~~

Обратите внимание, что после debugport: мы указываем номер Serial Port из прошлого действия    
Чтобы проверить, что все успешно настроено, вводим последовательно

~~~
bcdedit /dbgsettings
bcdedit
~~~
Результатом выполнения должны получить следующее:

![img12](https://github.com/sotnikea/Drivers_Learn/raw/main/windbg_dbg/images/img12.png)

Завершаем работу ОС
#### <a name="r3_1_4">Настройка WinDbg на хост компьютере</a>
Запускаем ярлык **WinDbg** настроенный на предыдущих этапах. Нажимаем **File - Kernel Debug**, переходим на вкладку **COM** и указываем выбранное название пайпа, а также сопутствующие настройки

![img13](https://github.com/sotnikea/Drivers_Learn/raw/main/windbg_dbg/images/img13.png)

Загружаем ОС в VMWare. **WinDbg** должен автоматически установить соединение с VMware при загрузке Windows.

![img14](https://github.com/sotnikea/Drivers_Learn/raw/main/windbg_dbg/images/img14.png)

Чтобы войти в отладчик, нажимаем **Ctrl+Brea**k или «**Debug**», а затем «**Break**» в строке меню. В этот момент виртуальная машина будет находиться в приостановленном состоянии (Windows перестанет отзываться). Подгружаем символы командой **.reload /f** и на этой настройку дебага можно считать завершенной

#### <a name="r3_1_5">Настройка немедленного запуска отладки</a>
Данный пункт не является обязательным, но он ускоряет и упрощает процесс запуска отладки.    
Открываем свойства ярлыка созданного для WinDbg и на вкладке Shortcut в строке Target в конце дописываем:
~~~
-k com:pipe,port=\\.\pipe\com_1,resets=0,reconnect
~~~
Вместо com_1 указываем название того com порта, который был создан в разделе Настройка VMWare. Конечный вариант поля target с учетом предыдущих настроек должно иметь приблизительно следующий вид:
~~~
"C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\windbg.exe" -y "srv*c:\symbols*https://msdl.microsoft.com/download/symbols" -k com:pipe,port=\\.\pipe\com_1,resets=0,reconnect
~~~
### <a name="r3_2">Using network</a>
#### <a name="r3_2_1">Setting Up KDNET Keys</a>
1. Для подключения хоста к целевому компьютеру для дебага, необходимо указать ip и порт, с которым мы будем работать. Начнем с ip. На хост компьютере открываем командную строку и вводим команду **ipconfig**. Записываем **IPv4 Address**

![img15](https://github.com/sotnikea/Drivers_Learn/raw/main/windbg_dbg/images/img15.png)

2. Чтобы убедиться, что **network debugging** поддерживается на целевом компьютере, нам понадобиться утилита **kdnet.exe**. С большой вероятностью на хост машине она уже должна быть (идет в комплекте с SDK). Проверить это можно по адресу:
~~~
C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\
~~~
Если же она там отсутствует, придется её скачивать. К сожалению отдельно сделать это не получится. Поэтому для начала придется скачать и запустить установку [Windows SDK](https://developer.microsoft.com/en-us/windows/downloads/windows-10-sdk/).    
После запуска установки, снять флажки со всех элементов кроме **Debugging tools for Windows** запустить и дождаться окончания установки

![img16](https://github.com/sotnikea/Drivers_Learn/raw/main/windbg_dbg/images/img16.png)

3. Нам понадобится только 2 файла:
- **kdnet.exe** C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\
- **VerifiedNICList.xml** C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\    
На целевом компьютере создаем папку (например **kdnet**) и кладем туда указанные два файла.

4. Чтобы убедиться, что **network debugging** поддерживается на целевом компьютере, запускаем командную строку, переходим в папку, которую создали на предыдущем пункте и выполняем команду **kdnet**. В случае успеха мы должны получить следующее сообщение

![img17](https://github.com/sotnikea/Drivers_Learn/raw/main/windbg_dbg/images/img17.png)

"Network debugging is supported on the following NICs" - говорит о том, что нужный нам дебаг поддерживается    
5. Для той же папки в командной строке выполняем команду
~~~
kdnet <HostComputerIPAddress> <YourDebugPort> 
~~~
Здесь требуются некоторые пояснения:
- **HostComputerIPAddress**- это **IPv4 Address** полученный нами на 1 этапе
- **YourDebugPort**- это порт, на котором будет работать дебаг. Рекомендуемый диапазон для него 50000-50039    
В результате будет получена строка с ключем, необходимым для подключения WinDbg с хост компьютера к целевому компьютеру. Всю эту строку можно сохранить, чтобы не придумывать свой ключ и не переписывать вручную номер порта (хотя можно придумать к короткий свой ключ, например 1.1.1.1)

![img18](https://github.com/sotnikea/Drivers_Learn/raw/main/windbg_dbg/images/img18.png)


#### <a name="r3_2_2">Configure kernel–mode debugging</a>
Переходим к конфигурации дебага
1. Включим режим отладки ядра в целевой системе. Для этого сначала проверим наличие связи хоста с целевым компьютером. На целевом компьютере открываем командную строку и выполняем команду
~~~
ping <HostComputerIPAddress>
~~~
где **HostComputerIPAddress**- это **IPv4 Address** полученный в предыдущем разделе.    
В результате должен получиться успешный обмен пакетами

2. Делаем активным kernel mode дебаг. Для этого на целевом компьютере открываем командную строку с правами администратора и выполняем команду
~~~
bcdedit /set {default} DEBUG YES
~~~
3. Делаем активной тестовую подпись командой
~~~
bcdedit /set TESTSIGNING ON 
~~~
4. Вводим команду для указания ip адреса хоста:
~~~
bcdedit /dbgsettings net hostip:<HostComputerIPAddress> port:<YourDebugPort> key:<SavedWithKDNETKey>
~~~
Здесь требуются некоторые пояснения:
- **HostComputerIPAddress**- это **IPv4 Addres**s полученный нами в предыдущем разделе
- **YourDebugPort** - это порт, на котором будет работать дебаг, так же заданный в предыдущем разделе
- **SavedWithKDNETKey** - ключ, полученный после выполнения команды kdnet и сохраненный в предыдущем разделе. Стоит так же обратить внимание, что сгенерированный в kdnet ключ не есть обязательный. Его можно придумать самостоятельно. Важно лишь, чтобы он совпадал здесь и в настройках WinDbg.
5. Чтобы убедиться, что все настроено верно, вводим команду
~~~
bcdedit /dbgsettings
~~~
Ожидаемый результат выполнения этой команды:

![img19](https://github.com/sotnikea/Drivers_Learn/raw/main/windbg_dbg/images/img19.png)

В случае возникновения предупреждения от файервола, необходимо предоставить полный доступ

![img20](https://github.com/sotnikea/Drivers_Learn/raw/main/windbg_dbg/images/img20.png)


#### <a name="r3_2_3">Запуск WinDbg Network debugging</a>
На хост компьютере находим **WinDbg**, обычно он находится по адресу 
~~~
C:\Program Files(x86)\Windows Kits\10\Debuggers\x64
~~~
и размещаем его ярлык в удобном месте. Запускаем его идем в меню **File → Kernel Debug**. Указываем в появившемся окне использованный в предыдущих этапах номер порта и ключ. Чтобы не выполнять эти действия после каждого запуска **WinDbg** можно так же задать это прямо в свойствах ярлыка программы. Для этого открываем ПКМ по ярлыку **WinDbg → Properties** и на вкладке **shortcut** в строке **Target** после находящегося там адреса вставляем всю строку полученную после выполнения команды **kdnet**

![img21](https://github.com/sotnikea/Drivers_Learn/raw/main/windbg_dbg/images/img21.png)

Приблизительный вид будет следующим
~~~
"C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\windbg.exe" -k net:port=50010,key=3hd86zx3atgzn.1p5p75ssdd053.3ifv7a32xvvcm.3lsskuh1vc7e0
~~~
Запускаем **WinDbg**, перезапускаем целевой компьютер, после чего получаем в **WinDbg** информацию об успешном подключении

![img22](https://github.com/sotnikea/Drivers_Learn/raw/main/windbg_dbg/images/img22.png)


## <a name="r4">Установка тестового сертификата и драйвера</a>
Включаем возможность запуска драйверов с тестовой подписью. Для этого на целевом компьютере:
- Открываем **Start - Settings - Update & Security - Recovery - Restart Now**
- После перезагрузку выбираем **Startup options - Troubleshoot - Advanced options - Startup Settings - Restart**
- Выбираем "**Disable driver signature enforcement**" нажатием кнопки **F7**
- Перезагружаем компьютер

Создаем на целевом компьютере папку, например "**EchoDriver**" и копируем в нее:
- **devcon.exe** из C:\Program Files (x86)\Windows Kits\10\Tools\x64\devcon.exe
- **echo.cer** из Windows-driver-samples\general\echo\kmdf\driver\AutoSync\x64\Debug
- **echo.inf** из Windows-driver-samples\general\echo\kmdf\driver\AutoSync\x64\Debug
- **echo.sys** из Windows-driver-samples\general\echo\kmdf\driver\AutoSync\x64\Debug
- **echoapp.exe** из Windows-driver-samples\general\echo\kmdf\exe\x64\Debug

Устанавливаем сертификат. Для этого в созданной папке нажимаем по **echo.ce**r ПКМ и выбираем "**Install Certificate**"

![img23](https://github.com/sotnikea/Drivers_Learn/raw/main/windbg_dbg/images/img23.png)

Для установки драйвера на целевом компьютере:
- Открываем командную строку от имени администратора
- Переходим в созданную ранее папку с файлами драйвера
- Выполняем в ней команду
~~~
devcon install echo.inf root\ECHO
~~~
После выполнения команды появится предупреждение, что Windows не может проверить разработчика драйвера. Указываем, что все равно хотим установить драйвер

![img24](https://github.com/sotnikea/Drivers_Learn/raw/main/windbg_dbg/images/img24.png)

## <a name="r5">Дебаг драйвера</a>
### <a name="r5_1">Установка брейкпоинта</a>
Запускаем **WinDbg** на хосте и запускаем целевой компьютер. В **WinDbg** выполняем **Break** и вводим команду
~~~
.sympath+ [Полный путь к папке с pdb файлом] Windows-driver-samples\general\echo\kmdf\driver\AutoSync\x64\Debug\
~~~
Устанавливаем брейкпоинт на функцию DeviceEntry. Для этого можно загрузить cpp файл и на нужной строке нажать F9 или ввести команду
~~~
bm ECHO!DriverEntry
~~~
Проверить, что он установился, можно с помощью команды bl
~~~
1: kd> bl
1 e Disable Clear  fffff806`52d67000     0001 (0001) ECHO!DriverEntry
~~~

### <a name="r5_2">Дебаг добавления драйвер</a>
Отпускаем дебаг командой **g**

На целевом компьютере открываем **DeviceManager** и разворачиваем **Sample Device**. ПКМ по вложенному ECHO драйверу и **Disable device**. После чего снова ПКМ и **Enable device**

![img25](https://github.com/sotnikea/Drivers_Learn/raw/main/windbg_dbg/images/img25.png)

Если все сделано правильно, в этот момент дальнейшее выполнение на целевом компьютере будет приостановлено и WinDbg предложит начать дебаг

![img26](https://github.com/sotnikea/Drivers_Learn/raw/main/windbg_dbg/images/img26.png)
