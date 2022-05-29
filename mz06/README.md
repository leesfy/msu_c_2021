
# Динамическая загрузка модулей и динамические (разделяемые) объекты

В данном документе рассматривается механизм динамической загрузки модулей, реализованный в Linux.
Linux поддерживает стандарт POSIX для интерфейса библиотеки динамической загрузки и использует формат
исполняемых файлов ELF. Поэтому рассматриваемый код, опции компиляции и имена файлов
будут переносимы на другие POSIX-системы с форматом исполняемых файлов ELF.

Хотя MacOS поддерживает стандарт POSIX, формат исполняемых файлов отличается от ELF. Динамические
библиотеки в MacOS имеют суффикс dylib. Возможно, потребуются немного другие аргументы командной строки
для компилятора и загрузчика.

## Компиляция динамически подгружаемой библиотеки

Чтобы скомпилированный файл можно было бы подгружать в память во время работы программы,
его нужно скомпилировать специальным образом.

Предположим, что в исходном файле myfunc.c определена функция myfunc.

```c
int myfunc(int x)
{
    return x + 1;
}
```

Для компиляции файла в динамически подгружаемую библиотеку нужно выполнить следующие команды:

```
gcc -O2 -Wall -Werror -std=gnu11 -fPIC -DPIC -c myfunc.c
```

отличие от обычной команды компиляции в опциях `-fPIC` `-DPIC`. Опция `-fPIC` включает режим
генерации позиционно-независимого кода. Опция `-DPIC` определяет макрос PIC значением 1.
В исходном коде может использоваться условная компиляция для случая генерации динамически
подгружаемых библиотек, тогда ненулевое значение макроса PIC будет означать генерацию кода для
динамической библиотеки.

Результатом первой команды будет объектный файл `myfunc.o`, который нужно скомпоновать
для получения окончательного результата с помощью команды:

```
gcc -std=gnu11 -fPIC -shared myfunc.o -olibmyfunc.so
```

Опция `-shared` передается в компоновщик (линкер) ld и задает формат выходного файла
shared object (динамически подгружаемая библиотека). Результат компоновки сохраняется 
в файл `libmyfunc.so`.

## Использование динамически подгружаемой библиотеки при компоновке

Предположим, что мы хотим использовать функцию `myfunc` из главной программы.

```
int myfunc(int x);  // определяем прототип функции

int main()
{
    int x;
    scanf("%d", &x);
    printf("%d\n", myfunc(x));
}
```

Объектный файл получается из исходной программы обычной командой компиляции.

```
gcc -O2 -Wall -Werror -std=gnu11 -c main.c
```

Чтобы скомпоновать исполняемую программу с использованием динамически подгружаемой библиотеки
потребуется следующая команда:

```
gcc main.o -L. -Wl,-rpath,. -omain -lmyfunc
```

Опция `-llib` указывает, что поиск всех функций, используемых в главной программе, в том числе функции
`myfunc` должен производиться и в библиотеке `myfunc`, которая находится в файле либо `libmyfunc.so`,
либо `libmyfunc.a` (это статическая библиотека). Динамически подгружаемая библиотека имеет приоритет над
статической библиотекой, кроме случая, когда требуется статическая компоновка (опция `-static`).

Опция `-L.` добавляет текущий каталог (`.`) к списку каталогов, в которых будет искаться файл `libmyfunc.so`.
По умолчанию текущий каталог отсутствует в списке каталогов библиотек, поэтому файл `libmyfunc.so` не будет
найден, несмотря на то, что он находится в текущем каталоге.

Опция `-Wl,` передает опции, записанные непосредственно после запятой, компоновщику, при этом символ запятая
будет разделителем опций. То есть опция `-Wl,-rpath,.` компилятора gcc приведет к тому, что компоновщику
(программе ld) будут переданы помимо всех прочих два аргумента командной строки: `-rpath` и `.`.
Эти аргументы добавляют текущий каталог (.)  в список каталогов, которые будут просматриваться
динамическим загрузчиком (ld-linux.so) для поиска необходимых библиотек при запуске программы.
Как и в случае с каталогами библиотек, текущий каталог отсутствует в списке каталогов,
просматриваемых динамическим загрузчиком по умолчанию, поэтому программа просто не запустится на выполнение.

Другой способ указания текущего каталога для загрузки динамически подгружаемых библиотек - задать
переменную окружения `LD_LIBRARY_PATH` при запуске программы на выполнение:
```
LD_LIBRARY_PATH=. ./main
```
тогда не нужно будет задавать опцию `-Wl,-rpath,.` в командной строке, но потребуется установить переменную
окружения. Вариант с заданием пути поиска динамически подгружаемых библиотек при компиляции программы
предпочтительнее.

В результате компоновки будет получена программа `main`, для выполнения которой потребуется динамически
подгружаемая библиотека `libmyfunc.so`, находящаяся в текущем рабочем каталоге.

## Явная динамическая подгрузка библиотек

В предыдущем примере действия по подгрузке требуемой динамической библиотеки в память процесса,
нахождению в ней имени `myfunc` и связыванию функции из динамической библиотеки с точкой вызова
в основной программе выполняются динамическим загрузчиком (ld-linux.so). Это удобно,
когда имена библиотек и функций известны на этапе компиляции и не меняются динамически.
Как правило, динамическое связывание основной программы и стандартных библиотек используется
по умолчанию.

Однако если имя динамически подгружаемой библиотеки или имя функции в ней неизвестно на этапе
компиляции и определяется на этапе работы программы (берется из конфигурационных файлов
или из ввода пользователя), динамическая компоновка средствами компилятора не применима.
Тогда нужно использовать библиотечные средства для загрузки библиотек и поиска имен в них.

В стандарте POSIX библиотечные функции для динамической загрузки определены в библиотеке dl.
Для их использования необходимо подключить заголовочный файл `dlfcn.h`.

```
#include <dlfcn.h>
```

Функции работы с динамическим загрузчиком не находятся в составе стандартной библиотеки libc, а вынесены
в отдельную библиотеку libdl. Поэтому при компоновке программы необходимо добавить опцию `-ldl`
для подключения этой библиотеки. Командная строка компоновки может выглядеть
следующим образом:

```
gcc main.o -omain -ldl
```

Концептуально при использовании библиотеки libdl каждая подгружаемый динамически модуль рассматривается
как некоторый ресурс, работа с которым ведется по модели "открыть-использовать-закрыть" (аналогично файлам,
каталогам и т. п.).

Динамически подгружаемый модуль загружается в память (т. е. "открывается") с помощью функции 'dlopen'.

```
void *dlopen(const char *filename, int flags);
```

Первый параметр `filename` - это имя файла, содержащего подгружаемый модуль. Если имя файла содержит '/',
оно интерпретируется как абсолютный или относительный путь к загружаемому модулю. Иначе для поиска модуля
используются значение переменной окружения `LD_LIBRARY_PATH`, список путей
вкомпилированный в исполняемый файл с помощью опции `-rpath` и системный список каталогов. Абсолютное имя
файла должно точно указывать на загружаемый модуль. Суффикс имени `.so` (на ELF-системах) можно не указывать, но
можно и указывать.

Параметр `flags` позволяет указать различные флаги. Должен быть указан либо флаг `RTLD_LAZY`, либо флаг `RTLD_NOW`.
`RTLD_LAZY` включает режим отложенного связывания, в котором поиск функции в библиотеке выполняется в момент
первого обращения к ней, а флаг `RTLD_NOW` включает режим немедленного связывания, в котором все действия
по связыванию использования имен с определениями выполняются на этапе загрузки библиотеки. Как правило,
всегда достаточно указать режим `RTLD_LAZY`. Кроме режима открытия в параметре `flags` могут передаваться
дополнительные флаги, объединяемые с помощью операции побитового или (`|`). Описание дополнительных флагов
можно найти в документации на функцию.

В случае ошибки функция `dlopen` возвращает `NULL`,
при этом текстовое сообщение об ошибке можно получить с помощью функции `dlerror`.

В случае успешной загрузки модуля в память процесса функция `dlopen` возвращает "handle" загруженной библиотеки,
некоторый указатель, который нужно передавать в последующие вызовы для работы с данным модулем.
Если модуль уже был загружен в память ранее, будет возвращен ранее созданный "handle", модуль не будет загружаться
второй раз, но счетчик его загрузок будет увеличен на 1. Это означает, в частности, что при повторной загрузке
модуля в память глобальные переменные этого модуля сохранят свои текущие значение, а не будут сброшены в 0.

Информацию об ошибке можно получить с помощью функции `dlerror`.

```
char *dlerror(void);
```

Функция возвращет NULL, если ошибки не было и текстовую строку описания ошибки в противном случае.
При этом флаг ошибки сбрасывается. Таким образом, второй подряд вызов `dlerror` всегда вернет NULL.
По аналогии с `errno` флаг ошибки не сбрасывается при успешном выполнении функций `dlopen`, `dlsym`
и т. п., поэтому перед первым вызовом `dlopen` имеет смысл вызвать `dlerror` чтобы принудительно
сбросить флаг ошибки.

Модуль "закрывается" с помощью функции `dlclose`.

```
int dlclose(void *handle);
```

Если счетчик загрузок модуля становится равным 0, он выгружается из памяти.

Найти в загруженном модуле функцию или переменную можно с помощью функции `dlsym`.

```
void *dlsym(void *handle, const char *symbol);
```

В параметре `handle` передается модуль, в котором нужно искать имя, а в параметре `symbol` - само имя.
Функция возвращает NULL, если имя не найдено, и указатель на соответствующий объект, если имя найдено.
Возвращенный указатель потребуется явно или неявно привести к типу указателя на функцию,
чтобы его можно быть использовать для вызова.

Таким образом, простейший вариант функции `main`, который использует явную
загрузку динамического модуля libmyfunc может быть таким.

```
int main()
{
    int x;
    scanf("%d", &x);
    // подгружаем модуль
    void *handle = dlopen("./libmyfunc.so", RTLD_LAZY);
    // ищем в нем myfunc
    void *sym = dlsym(handle, "myfunc");
    // вызываем
    int res = ((int (*)(int)) sym)(x);
    printf("%d\n", res);
    dlclose(handle);
}
```

В этом примере игнорируются возможные ошибки при выполнении функций `dlopen`, `dlsym`. В реальных программах так делать нельзя!

## Работа с функциями в самой программе

Основная программа принципиально ничем не отличается от подгружаемых модулей, то есть нам может потребоваться искать адреса
функций или переменных по их именам не только в подгружаемых модулях, но и в основной программе. Работа с именами
в основной программе тоже ведется по схеме "открыть-использовать-закрыть". Чтобы "открыть" основную программу
нужно в параметре filename передать NULL, например:

```
    mainhandle = dlopen(NULL, RTLD_LAZY);
```

После чего можно искать имена в основной программе:

```
    mainfunc = dlsym(mainhandle, "main");
```

Однако по умолчанию основная программа компилируется без таблицы имен, доступной для функции `dlsym`,
поэтому функция вернет `NULL`. Чтобы включить генерацию таблицы имен в основную программу
нужно компоновать ее с опцией `-rdynamic`.

```
gcc -rdynamic main.o -omain -ldl
```

# Файлы, отображаемые в память

Основным способом управления адресным пространством процесса в современных
операционных системах является отображение файлов в адресное пространство процесса.

После отображения файла в адресное пространство содержимое отображенного файла становится
доступным при обращении в память. Файл таким образом становится как бы массивом
в памяти. Все модификации массива могут быть сохранены в файл ядром операционной
системы без специальных системных вызовов со стороны выполняющегося процесса.

Механизм отображения файла в память реализуется с помощью механизма
страничной виртуальной памяти. Одна страница - это минимальная единица
работы ядра с виртуальным адресным пространством. То есть, если отобразить
область файла, меньшую одной страницы, то все равно будет занята одна
целая страница виртуального адресного пространства. 

## Получение размера страницы

На многих современных процессорных архитектурах,
например, на наиболее распространенных в настоящее
время x86 и ARM, размер страницы
виртуальной памяти равен 4 KiB. Однако переносимая программа
должна запросить у ядра размер страницы виртуальной памяти.
Это можно сделать следующим образом:
```c
#include <unistd.h>

size_t sz = sysconf(_SC_PAGESIZE);
```

Размер страницы всегда является степенью числа 2.

## Системный вызов `mmap`

Основной системный вызов для отображения файла в память - `mmap`.

```c
#include <sys/mman.h>
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
```

Режим отображения файла в память задается с помощью параметра `flags`.
Должен быть выбран либо режим `MAP_PRIVATE`, либо режим `MAP_SHARED`.

В режиме `MAP_SHARED` поддерживается синхронизация содержимого памяти
и файла в обе стороны, то есть, если файл изменился на диске, это
изменение будет отображено в память, и наоборот, если файл
изменился в памяти, измененное содержимое будет сохранено на диск.
Если некоторая страница виртуального адресного пространства,
отображенная на файл, изменилась одновременно с изменением
соответствующей области на диске, будет неопределенный результат,
может сохранится любое из изменений, либо оба (ошибка типа race condition).

Если несколько процессов откроют один и тот же файл в режиме `MAP_SHARED`,
они будут видеть изменения друг друга, то есть эти процессы смогут использовать
разделяемую память. В реализации это означает, что виртуальные страницы памяти
нескольких процессов будут отображены на одни и те же физические страницы памяти.

В режиме `MAP_PRIVATE` файл отображается для "приватного" использования.
Пока отображенная из файла страница не будет изменена процессом
содержимое в памяти будет синхронизировано с содержимым файла, то есть
если файл изменится, то изменится и содержимое отображенной страницы в памяти.

Однако как только процесс модифицирует какую либо виртуальную страницу,
связь этой страницы с содержимым файла разрывается, и в дальнейшем эта страница будет
"жить своей жизнью", то есть если файл изменится, это изменение не отразится
на содержимом страницы, и наоборот, изменение содержимого страницы не будет
сохранено в файл. Такая стратегия называется "copy-on-write", то есть при
первой операции модификации страницы в памяти создается ее копия,
и далее процесс работает с копией страницы, а не с оригинальной страницей.

Если файл отображается в память только для чтения, то все равно,
в каком режиме его отображать: `MAP_SHARED` или `MAP_PRIVATE`.
Режим `MAP_PRIVATE` нужен, чтобы задать начальное значение для
страниц виртуальной памяти, которые, возможно, далее будут
модифицироваться процессов. Именно в режиме `MAP_PRIVATE` отображаются
в память исполняемые файлы и динамические библиотеки.

Другие полезные флаги, допустимые в параметре `flags`, перечислены ниже.

Флаг `MAP_ANONYMOUS` позволяет отображать "анонимные" файлы.
По сути, анонимный файл - это просто оперативная память.
В режиме `MAP_ANONYMOUS | MAP_PRIVATE` выделяется виртуальная
память для использования только текущим процессом, то есть это
просто выделение страниц виртуальной памяти процессу, например,
для использования в качестве буферов или в качестве динамической памяти.
В режиме `MAP_ANONYMOUS | MAP_SHARED` память будет
разделяться между текущим процессом и процессами,
которые унаследуют виртуальную память после `fork`.

Флаг `MAP_FIXED` вынуждает ядро отобразить виртуальную память
точно по адресу `addr`. Параметр `addr` должен быть в этом случае
корректно выровнен. Если это сделать невозможно, `mmap` завершается
с ошибкой.

Флаг `MAP_FIXED_NOREPLACE` запрещает новому отображению перекрывать
уже существующие отображения в память, то есть все страницы виртуальной
памяти должны быть свободны.

Без этого флага системный вызов `mmap` позволяет перекрывать уже
существующие отображения. Если некоторая страница виртуальной
памяти была уже занята, то она освобождается и на ее место
помещается страница нового отображения. Это позволяет создавать
различные конфигурации виртуальных страниц, например,
вставлять в середину одного отображения другое отображения и т. п.

Параметр `prot` устанавливает права доступа на отображенные виртуальные страницы.
Флаг `PROT_READ` разрешает чтение, `PROT_WRITE` разрешает запись,
а `PROT_EXEC` - исполнение. Если разрешены запись или исполнение,
разрешение на чтение подразумевается, то есть на уровне виртуальной
памяти нельзя создать страницы памяти "только для записи"
или "только для исполнения".

Параметр `addr` задает желаемый адрес для отображения (hint).
Его можно передавать равным `NULL`, тогда ядро само отобразит
файл в свободную область.

Параметр `length` задает длину отображения. Она округляется вверх до полных страниц
виртуальной памяти. Если параметр `length` равен 0, результат работы
системного вызова неопределен.

Параметр `fd` - это открытый файловый дескриптор регулярного файла,
который необходимо отобразить в память. Однако, режим отображения
`MAP_ANONYMOUS` не требует файлового дескриптора, и в этом случае
должно передаваться значение `-1`.

Параметр `offset` - это смещение от начала файла отображаемой
части файла. Оно должно быть кратно странице.

В случае успеха возвращается виртуальный адрес начала отображенного файла.
Обратите внимание, что он может быть равен 0, суперюзер
имеет право отображать файлы на нулевой адрес.

В случае неудачи системный вызов возвращает значение `MAP_FAILED`.

## Системный вызов `munmap`

```c
#include <sys/mman.h>
int munmap(void *addr, size_t length);
```

Отключает виртуальные страницы существующего отображения.
Параметр `addr` - адрес начала (должен указывать на начало страницы),
а параметр `length` - длина (округляется вверх до размера кратного странице).
Отключаться может любая часть (даже середина) существующего отображения.
Если страница и так никуда не отображается, это не вызывает ошибку.

## Особенности использования

* Файл должен быть открыт либо в режиме `O_RDONLY`, либо в режиме `O_RDWR`.
Файл, открытый в режиме `O_WRONLY`, отобразить нельзя.

* Попытка отображения в память пустого файла - undefined behavior.

* Файл, открытый в режиме `MAP_PRIVATE` можно отображать
с правами `PROT_READ | PROT_WRITE` независимо от режима открытия файла.

* После `mmap` файловый дескриптор можно закрыть, отображение
файла в память сохранится.

* Если размер файла не кратен размеру страницы, последняя страница
отображения будет заполнена содержимым файла частично, хвост будет
обнулен. Обращаться в память можно в границах страницы, но модификации
после конца файла в файл сохранены не будут.

* Если уже после отображения размер файла уменшился, программа может
получить `SIGBUS` при доступе за концом файла.

* Если при работе с отображенным файлом возникла ошибка ввода-вывода,
программа получит сигнал `SIGBUS`.

Source: https://github.com/cmc-prak/cmc-os/tree/master/2021-2022