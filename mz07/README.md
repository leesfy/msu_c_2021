# Работа с процессами в POSIX-системах, часть 1

## Содержание

* [Создание процесса](#ProcessCreation)
* [Получение идентификации процессов](#ProcessIdentification)
* [Завершение работы процесса](#ProcessTermination)
* [Ожидание завершения процесса](#ProcessWaiting)
  * [Системный вызов `wait`](#ProcessWaitingWait)
  * [Системный вызов `waitpid`](#ProcessWaitingWaitpid)
  * [Системный вызов `wait4`](#ProcessWaitingWait4)
  * [Возможные коды ошибок системных вызовов семейства `wait`](#ProcessWaitingErrors)
* [Использование системных вызовов](#ProcessUsage)
  * [Что такое "успешное завершение" процесса](#ProcessUsageSuccess)
  * [Как дождаться всех запущенных процессов](#ProcessUsageWaitAll)
  * [Почему после `fork` в сыне предпочтительнее `_exit`](#ProcessUsageExit)

Дополнительные темы

* [Автоматическое уничтожение "зомби"-процессов](#ExtraNoZombie)
* [Процесс init](#ExtraInit)
* [Группы процессов](#ExtraProcessGroup)
* [Процессы как файловые дескрипторы (pidfd)](#ExtraPidfd)

Понятие «процесс» наряду с понятием «файл» относится к основным понятиям операционной системы.
Под процессом можно понимать программу в стадии выполнения.
С процессом в системе связано много атрибутов, например, страницы памяти, занимаемые процессом, или идентификатор пользователя.
Процесс использует ресурсы системы, поэтому одна из основных задач ядра операционной системы — распределение ресурсов между процессами.
Каждый процесс имеет своё адресное пространство. Если в системе в текущий момент есть несколько готовых к выполнению процессов, они работают параллельно.
На системе с одним процессором (процессорным ядром) эти процессы разделяют время одного процессора, а на системе
с несколькими процессорами (ядрами) каждый процесс может выполняться на своём процессоре, и
они будут работать действительно параллельно.
Параллелизм выполнения вносит принципиально новые моменты по сравнению с обычным последовательным программированием и
делает параллельные программы значительно более трудоёмкими в разработке и отладке.

<a name="ProcessCreation" />

## Создание процесса

Новый процесс создаётся с помощью системного вызова `fork`.

```c
#include <unistd.h>
pid_t fork(void);
```

Вновь созданный (сыновний) процесс отличается только своим идентификатором процесса pid
и идентификатором родительского процесса ppid.
Кроме того, у нового процесса сбрасываются счётчики использования ресурсов, блокировки файлов и ожидающие сигналы.

Сыновний процесс продолжает выполнять тот же код, что и родительский процесс с точки программы, непосредственно
после возврата из функции `fork`.
Отличить новый процесс от родительского можно только по возвращаемому системным вызовом `fork` значению.
В сыновний процесс возвращается 0, а в родительский процесс возвращается идентификатор сыновнего процесса.
Кроме того, функция fork возвращает число -1,
когда новый процесс не может быть создан из-за нехватки ресурсов, либо из-за превышения
максимального разрешённого числа процессов для пользователя или всей системы. Обычно
функция fork используется следующим образом:

```c
    if ((pid = fork())) < 0) {
        /* сигнализировать об ошибке */
    } else if (!pid) {
        /* этот фрагмент кода выполняется в сыновнем процессе */
        /* ... */
        _exit(0);
    } else {
        /* а этот фрагмент выполняется в родительском процессе */
    }
```

<a name="ProcessIdentification" />

## Получение идентификации процессов

Системные вызовы `getpid`, `getppid` позволяют получить идентификатор текущего или родительского процесса.

```c
#include <unistd.h>
pid_t getpid(void);
pid_t getppid(void);
```

Функция `getpid` возвращает идентификатор текущего процесса, а функция `getppid` возвращает идентификатор родительского процесса.
Если родительский процесс завершил своё выполнение раньше сыновнего, «родительские права» передаются процессу с номером 1 —
процессу init (но это не точно).

Идентификатор процесса - это положительное число в интервале от 1 до некоторого максимального значения.
При создании нового процесса система назначает ему новый идентификатор.
Когда идентификатор процесса достигнет максимального допустимого значения, счёт начнётся снова с 1
и операционная система назначит идентификатор процесса, не занятый в данный момент каким-либо процессом.
Когда операционная система работает достаточно долго, один и тот же идентификатор процесса будет использоваться несколько раз.

<a name="ProcessTermination" />

## Завершение работы процесса

```c
#include <stdlib.h>
void exit(int status);
void _exit(int status);
void _Exit(int status);
void abort(void);
```

Процесс может завершить свою работу одним из следующих способов:
* Вызовом библиотечной функции `exit(status);`, где `status` — код завершения процесса.
* Возвратом из функции `main`. Этот способ эквивалентен предыдущему. В качестве кода
завершения процесса используется значение, возвращаемое из `main`.
* Вызовом системного вызова `_exit(status);` или, что то же самое `_Exit(status);`, где `status` — код завершения процесса.
* Получением необрабатываемого, неигнорируемого и неблокируемого сигнала, который
вызывает по умолчанию нормальное или аварийное завершение процесса.

У значения `status` значимым является только младший байт, то есть код завершения процесса - это
целое число в интервале от 0 до 255. Значение кода завершения процесса доступно процессу-родителю
с помощью системных вызовов семества `wait`.

Библиотечная функция `exit` отличается от системного вызова `_exit` (или `_Exit`) тем, что функция
`exit` выполнит корректное закрытие всех открытых дескрипторов потока (`FILE *` )и вызовет обработчики завершения работы процесса,
зарегистрированные с помощью функции `atexit`. В программе на C++ при вызове `exit` будут вызваны деструкторы глобальных объектов,
но не деструкторы локальных объектов. Системный вызов `_exit` вызывает немедленное завершение процесса,
при этом содержимое незаписанных буферов дескрипторов потоков (`FILE *`) теряется.

Отличие функции `_exit` от `_Exit` в том, что первая функция стандартизирована в POSIX (то есть в Unix-ориентированных
операционных системах), а вторая функция стандартизирована в ISO C99. Содержательных отличий между ними нет.

Библиотечная функция `abort` эквивалентна вызову `raise(SIGABRT)`, то есть отправке сигнала `SIGABRT` процессом
самому себе. Если сигнал `SIGABRT` обрабатывается по умолчанию, процесс завершится и будет сгенерирован core-файл.

В любом случае все ресурсы, связанные с процессом освобождаются, но запись в таблице процессов не удаляется для того,
чтобы процесс-родитель смог прочитать статус завершения процесса.
Процесс в таком состоянии, когда все ресурсы уже освобождены, и осталась только запись в таблице процессов, называется «зомби».
Если процесс-родитель «не интересуется» судьбой сыновних процессов, они останутся зомби до тех пор, пока процесс родитель не завершится.
Тогда их родителем станет процесс 1 (init). Как только родительский процесс прочитает статус завершения сыновнего процесса-зомби, запись в таблице
процессов уничтожается, и процесс окончательно прекращает своё существование, а идентификатор этого процесса станет
доступным для повторного использования.

<a name="ProcessWaiting" />

## Ожидание завершения процесса

Для ожидания завершения процесса и получения информации о завершившимся процессе используются системные вызовы семейства `wait`.
Это - `wait` (самый простой), `waitpid`, `wait3`, `wait4`.

<a name="ProcessWaitingWait" />

### Системный вызов `wait`

Простейшая функция, с помощью которой можно узнать состояние процесса, — системный вызов `wait`.

```c
#include <sys/types.h>
#include <sys/wait.h>
pid_t wait(int *pstatus);
```

Функция `wait` приостанавливает выполнение текущего процесса до тех пор, пока какой-либо сыновний процесс не завершит своё выполнение,
либо пока в текущий процесс не поступит сигнал, который вызовет обработчик сигнала или завершит выполнение процесса.
Если какой-либо сыновний процесс уже завершил выполнение («зомби»), функция возвращается немедленно, и зомби-процесс окончательно уничтожается.

Если указатель pstatus не равен NULL, функция `wait` записывает информацию о статусе завершения по адресу `pstatus`.
Для получения информации о статусе завершения процесса могут использоваться следующие макросы.
Они принимают в качестве параметра сам статус завершения процесса (типа `int`), а не указатель на него.

* `WIFEXITED(status)` — принимает ненулевое значение, если процесс завершил своё выполнение самостоятельно с помощью системного вызова
семейства `exit`.

* `WEXITSTATUS(status)` — этот макрос может быть использован, только если вызов макроса `WIFEXITED` дал ненулевое значение.
Макрос выдает код завершения процесса (число от 0 до 255), который был задан как аргумент при самостоятельном завершении процесса с
помощью функции семейства `exit`.

* `WIFSIGNALED(status)` — принимает ненулевое значение, если процесс завершил своё выполнение в результате получения необрабатываемого сигнала,
который по умолчанию вызывает завершение работы процесса.

* `WTERMSIG(status)` — этот макрос может быть использован, только если вызов макроса `WIFSIGNALED` дал ненулевое значение.
Макрос выдаёт номер сигнала, который вызвал завершение работы процесса. В Linux номер сигнала может принимать значения от 1 до 64.

* `WCOREDUMP(status)` — возвращает истинное значение, если при завершении процесса должет был быть сгенерирован `core`-файл.
Был ли он сгенерирован в действительности и каково его имя - зависит от настроек ядра операционной системы.

Функция возвращает идентификатор завершившегося процесса, либо -1 в случае ошибки.
В этом случае переменная `errno` устанавливается в код ошибки. Возможные коды ошибки приведены в таблице.

<a name="ProcessWaitingWaitpid" />

### Системный вызов `waitpid`

Больше возможностей предоставляет системный вызов `waitpid`, определенный следующим образом.

```c
#include <sys/types.h>
#include <sys/wait.h>
pid_t waitpid(pid_t pid, int *pstatus, int options);
```

Параметр `pstatus` имеет такой же смысл, как и в системном вызове `wait`, то есть задает адрес переменной,
в которую записывается статус завершения процесса. Параметр может быть равен `NULL`, тогда статус
завершения процесса не будет записан никуда.

Параметр `pid` позволяет уточнить, какой завершение какого процесса нужно ждать.
* Если `pid > 0`, системный вызов ожидает завершения процесса с заданным `pid`. Этот процесс должен быть процессом-сыном
данного процесса, для которого еще не выполнялся системный вызов семейства `wait`.
* Если `pid == -1`, системный вызов ожидает завершения любого процесса-сына данного процесса.
* Если `pid == 0`, системный вызов ожидает завершения любого процесса-сына в той же самой группе процессов, к которой
принадлежит процесс-отец.
* Если `pid < 0`, системный вызов ожидает завершения любого процесса-сына в группе процессов с идентификатором `-pid`.

В любом случае, действие системных вызовов семейства `wait` распространяется только на непосредственных сыновей
данного процесса, и не распространяется на внуков, правнуков, и т. д.

Параметр `options` позволяет указать дополнительные опции системного вызова. Если указать опцию `WNOHANG`,
системный вызов не будет переводить процесс в режим ожидания в случае, если ни один из
процессов, подходящих под параметр `pid`, не завершился, а вместо этого `waitpid` вернет значение 0 немедленно.

Вызов

```c
res = waitpid(-1, &status, 0);
```

эквивалентен вызову

```c
res = wait(&status);
```

<a name="ProcessWaitingWait4" />

### Системный вызов `wait4`

Еще больше возможностей предоставляет системный вызов `wait4`, определённый следующим образом:

```c
#include <sys/types.h>
#include <sys/resource.h>
#include <sys/wait.h>
pid_t wait4(pid_t pid, int *pstatus, int options, struct rusage *prusage)
```

Дополнительно к возможностям системного вызова `waitpid` системный вызов `wait4` позволяет
получить информацию о потреблении ресурсов завершившегося процесса.

Если параметр `prusage` не равен `NULL`, информация об использовании процессом ресурсов системы записывается по адресу,
указанному в аргументе `prusage`.
Структура `struct rusage` определена в операционной системе Linux следующим образом:

```c
struct rusage
{
    struct timeval ru_utime; /* время в режиме пользователя */
    struct timeval ru_stime; /* время в режиме ядра */
    long ru_maxrss;          /* максимальный размер в памяти (RSS) */
    long ru_ixrss;           /* размер всех разделяемых страниц - не заполняется */
    long ru_idrss;           /* размер всех неразделяемых страниц - не заполняется */
    long ru_isrss;           /* размер страниц стека - не заполняется */
    long ru_minflt;          /* подгрузок страниц (page fault) без операций ввода-вывода */
    long ru_majflt;          /* подгрузок страниц (page fault) с операциями ввода-вывода */
    long ru_nswap;           /* количество полных откачек - не заполняется */
    long ru_inblock;         /* блочных операций ввода */
    long ru_oublock;         /* блочных операций вывода */
    long ru_msgsnd;          /* послано сообщений - не заполняется */
    long ru_msgrcv;          /* получено сообщений - не заполняется */
    long ru_nsignals;        /* получено сигналов - не заполняется */
    long ru_nvcsw;           /* "добровольных" переключений контекста - при запросах на ввод-вывод */
    long ru_nivcsw;          /* принудительных переключений на другой процесс */
};
```

Часть полей (отмеченных "не заполняется") в Linux всегда равны 0. Эти поля добавлены для совместимости с другими системами.
Возможно, поддержка заполнения этих полей будет добавлена когда-нибудь.

<a name="ProcessWaitingErrors" />

### Возможные коды ошибок системных вызовов семейства `wait`

| Код ошибки | Описание |
|------------|----------|
|`ECHILD` | У процесса в данный момент времени нет детей (в т. ч. "зомби"). Если процесс создавал другие процессы, то все они завершились и для всех был выполнен `wait`. Или (для `waitpid`/`wait4`) указан отсутствующий идентификатор процесса/группы или указанный идентификатор процесса/группы не относится к сыновьям текущего процесса.
|`EINTR`  | Во время ожидания завершения процессов пришел сигнал, который был обработан с помощью пользовательского обработчика, и управление было возвращено в основную программу.
|`EINVAL` | Неверный параметр системного вызова - неверные флаги `options` или неверные указатели `pstatus` или `prusage`.

<a name="ProcessUsage" />

## Использование системных вызовов

<a name="ProcessUsageSuccess" />

### Что такое "успешное завершение" процесса

Процесс считается успешно завершившимся, если он завершился с помощью системного вызова `_exit` или функций семейства `exit` с кодом завершения 0.
Если процесс завершился `exit` с ненулевым младшим байтом кода возврата, либо процесс завершился из-за получения сигнала, который не обрабатывается
процессом и вызывает его завершение, такой процесс не считается успешно завершившимся.

На уровне интерпретатора командной строки успешно завершившиеся процессы расцениваются как вернувшие булевское значение true, а все прочие - как false.
То есть в фрагменте скрипта
```shell
prog && echo 'success'
```

Строка `success` будет выведена только при успешном завершении программы `prog`, то есть когда код завершения `prog` равен 0.
По сути, в логике shell значение true - это 0, а значение false - это любое ненулевое значение.

Чтобы программно проверить процесс на успешное завершение нужно убедиться, что процесс завершился по `exit` и код его возврата 0, примерно следующим образом.

```c
    int status;
    wait(&status);  // дожидаемся завершения процесса
    if (WIFEXITED(status) && !WEXITSTATUS(status)) {
        // успешное завершение процесса
    }
```

<a name="ProcessUsageWaitAll" />

### Как дождаться всех запущенных процессов

Если необходимо просто дождаться всех запущенных процессов, можно использовать цикл while.

```c
    while (wait(NULL) > 0) {}
```

Пока есть процессы-сыновья системный вызов `wait` будет дожидаться их завершения.
Как только все процессы-сыновья завершены и их завершение подтверждено родителем с помощью `wait`,
очередной вызов `wait` вернет значение -1 и установит код ошибки в `ECHILD`. На этом завершиться цикл `while`
и процесс-родитель продолжит выполнение программы после цикла.

<a name="ProcessUsageExit" />

### Почему после `fork` в сыне предпочтительнее `_exit`

Системный вызов `fork` создает новый процесс, адресное пространство которого является точной копией исходного процесса.
В том числе, будут скопированы все структуры данных, находящиеся в адресном пространстве исходного процесса,
например, буфера ввода-вывода высокоуровневых дескрипторов потока (`FILE *` или `std::ostream`).

Например, в следующем фрагменте кода

```c
    printf("Hello");
    if (!fork()) {
        exit(0);
    }
```

строка `Hello` будет выведена дважды как `HelloHello`. Дело в том, что при создании процесса будет скопирован буфер
стандартного потока вывода `stdout` в адресном пространстве процесса-родителя, который содержит строку `Hello`.
Этот буфер не был записан в файловый дескриптор 1, так как по умолчанию включена либо построчная, либо полная
буферизация. Вызов `exit(0)` в процессе-сыне перед завершением процесса выполнит операцию `fflush(stdout)`,
которая выполнит системный вызов `write` и запишет содержимое буфера на стандартный поток вывода еще раз.

Использование `_exit` вместо `exit` снимает эту проблему. `_exit` - это "чистый" системный вызов, который
просто завершает процесс. При этом не выполняются вызовы `fflush` для дескрипторов потоков, и т. п.
Ядро освободит все ресурсы, выделенные процессу при `fork`, поэтому утечек ресурсов не произойдет.
Это особенно важно в библиотечных функциях, которые создают новые процессы, так как библиотечная функция
может вызываться в разных программах в разных состояниях программы.

# Дополнительные темы

<a name="ExtraNoZombie" />

## Автоматическое уничтожение "зомби"-процессов

По умолчанию завершившийся процесс останется в состоянии "зомби", пока статус его завершения не будет
запрошен и получен его родителем. Если родитель не выполняет `wait*` и запускает новые процессы,
процессы-зомби будут накапливаться, что в итоге может привести к исчерпанию ресурсов.

Однако, если установить обработчик сигнала `SIGCHLD` в игнорирование:

```c
    signal(SIGCHLD, SIG_IGN);
```

то процессы в состоянии "зомби" будут уничтожены автоматически. То есть, если на момент завершения
процесса его родитель выполняет системный вызов `wait` на данный процесс, то родитель
получит статус завершения сына. Однако, если родитель в момент завершения процесса выполняет
другие действия, то информация о статусе завершения этого процесса потеряется.
Процесс-родитель не сможет стандартными средствами получить информацию о том, когда и как
завершился процесс.

<a name="ExtraInit" />

## Процесс init

Традиционно если родитель некоторого процесса завершался раньше самого процесса, то "родительские права"
на процесс передавались процессу с pid 1, который традиционно называется init. Процесс init
отвечает за запуск и остановку процессов-демонов операционной системы, инициализацию терминалов для входа,
и т. д.

Например, в следующем фрагменте программы:

```c
    if (!fork()) {
        printf("%d\n", getppid());
    }
    exit(0);
```

На стандартный поток вывода может быть выведет pid текущего процесса, либо 1, в зависимости от того,
успел ли процесс-родитель завершиться раньше момента, когда процесс-сын выполнит системный вызов `getppid`.

Современный Linux, однако, позволяет менять процесс, который получить "родительские права"
в случае, если процесс-отец завершится раньше процесса-сына. Для этого используется специфичный
для Linux системный вызов `prctl`.

```c
    prctl(PR_SET_CHILD_SUBREAPER, 1, 0, 0, 0);
```

Этот системный вызов устанавливает флаг передачи "родительских прав" у текущего процесса. После этого,
если какой-либо процесс ниже по дереву создания процессов завершится раньше своих сыновей,
"родительские права" на сыновей будут переданы данному процессу. Точнее, если у некоторого процесса
отец завершается раньше самого процесса, то вверх по дереву создания процессов ищется ближайший
процесс с флагом `CHILD_SUBREAPER`, и такой процесс станет родителем данного. Процесс 1 (init) является
корнем дерева создания процесса и у него установлен флаг `CHILD_SUBREAPER`, поэтому такая система
обратно совместима с традицией в Unix.

<a name="ExtraProcessGroup" />

## Группы процессов

Процессы в Unix объединяются в группы процессов (process group). У группы процессов выделяется главный процесс - "лидер"
группы процессов, с которого начинается создание группы. Группа процессов идентифицируется идентификатором процесса-лидера группы,
то есть используются те же значения pid, что и для процессов. Процесс-лидер может завершиться раньше других процессов данной
группы процессов, но группа процессов сохранится. Идентификатор процесса-лидера не будет повторно использоваться,
пока не завершатся все процессы этой группы.

Идентификатор группы процессов может использоваться для ожидания любого процесса в группе (см. выше), а также
для отправки сигналов одновременно всем процессам в группе. Группы процессов используются ядром операционной системы
для рассылки сигналов SIGINT, SIGQUIT, SIGTSTP.

По умолчанию группа процессов наследуется при `fork` и `exec*`, то есть процессы-сыновья остаются в группе процессов родителя.
Родитель совместно с сыновьями может манипулировать принадлежностью создаваемых групп процессов, чтобы организовать
для созданных процессов отдельную группу. Так поступает shell при запуске процессов из командной строки.
Процессы, запущенные из командной строки, помещаются в новую группу процессов, отличную от группы процессов,
в которой работает shell. Процессы, образующие конвейер, например, `a | b | c` все помещаются в одну
группу процессов.

Получить группу процессов, к которой принадлежит указанный процесс, можно с помощью системного вызова `getpgid`.

```c
pid_t getpgid(pid_t pid);
```

Для того чтобы получить идентификатор группы текущего процесса можно передавать значение 0 в качестве аргумента,
то есть:

```c
    pid_t mypgid = getpgid(0);
```

Необходимость получить идентификатор группы процессов у другого процесса возникает редко.

Чтобы изменить принадлежность процесса группе процессов используется системный вызов `setpgid`.

```c
int setpgid(pid_t pid, pid_t pgid);
```

Первый аргумент `pid` указывает идентификатор процесса, к которому должна быть применена операция,
а второй аргумент `pgid` указывает идентификатор группы процессов, в которую должен быть помещен
данный процесс. 0 в качестве идентификатора процесса `pid` обозначает текущий процесс.

Переходы процесса из одной группы процессов в другую возможны только между группами процессов,
относящихся к одной сессии процессов.

### Создание первого процесса в новой группе процессов (лидера группы)

Системный вызов `setpgid` должен быть выполнен и в отце, и в сыне после `fork` примерно
следующим образом.

```c
    pid_t leader = fork();
    if (leader < 0) {
        // ошибка
    } else if (!leader) {
        // находимся в сыне
        setpgid(0, 0);  // текущий процесс переносится в группу процессов с идентификатором текущего процесса
        // поскольку изначально этот процесс состоял в группе процессов своего отца,
        // то создается новая группа процессов и этот сын становится ее лидером
        // так как идентификатор группы процессов совпадает с идентификатором самого процесса
        // возможная ошибка выполнения системного вызова игнорируется намеренно
    } else {
        // находимся в отце
        setpgid(leader, 0); // процесс-сын переносится в группу процессов с идентификатором процесса-сына
        // возможная ошибка выполнения системного вызова игнорируется намеренно
    }
```

Операция переноса процесса-сына в новую группу процессов выполняется и отцом, и сыном, поскольку
неопределено, в каком порядке будут параллельно выполняться операции в отце и сыне.
Если отец не выполнит операцию переноса сына в новую группу процессов и продолжит исполняться дальше,
то возможна ситуация, когда второй процесс-сын пытается перенести себя в группу процессов `leader`,
а она еще не создана, так как первый сын еще не успел выполнить `setpgid`.
Если сын не выполнит операцию создания новой группы процессов, возможна ситуация, что сын
завершится раньше, чем отец успеет выполнить `setpgid`, и поэтому группа процессов не будет создана.

### Создание последующих процессов в группе процессов

При создании последующих процессов и отец, и сын должны поместить каждый новый процесс в уже
существующую группу процессов примерно следующим образом.

```c
    pid_t pid = fork();
    if (pid < 0) {
        // ошибка
    } else if (!pid) {
        // находимся в сыне
        setpgid(0, leader); // переносим себя в группу процессов leader
        // возможная ошибка выполнения системного вызова игнорируется намеренно
    } else {
        // находимся в отце
        setpgid(pid, leader); // переносим сына в группу процессов leader
        // возможная ошибка выполнения системного вызова игнорируется намеренно
    }
```

<a name="ExtraPidfd" />

## Процессы как файловые дескрипторы (pidfd)

У традиционного способа работы с процессами в Unix, рассмотренного выше, есть несколько неудачных особенностей.

* Во-первых, повторное использование идентификаторов процесса. Если процесс завершится и будет "похоронен" с помощью
`wait`, его идентификатор процесса освобождается для повторного использования. Возможна ситуация (например, в случае
какого-либо race condition), что процесс работает с некоторым идентификатором другого процесса, считая что это
еще исходный процесс, хотя на самом деле на его месте уже находится другой процесс, что может приводить, например,
к проблемам с безопасностью.
* Во-вторых, для ожидания завершения процесса используется системный вызов семества `wait*`. Он может выполняться
и в неблокирующем варианте (флаг `WNOHANG`), но тогда придется периодически выполнять проверку.
Для файловых дескрипторов в Unix существует несколько системных вызовов (`select`, `poll`, `epoll`), которые
позволяют ждать наступления события сразу на множестве файловых дескрипторов, но вот схема с ожиданием
на `wait` сюда не вписывается.

Одно из направлений развития Linux - унификация разнородных интефейсов традиционного Unix в классический интерфейс
файловых дескрипторов. Так в Linux реализована работа с сигналами с помощью файловых дескрипторов (signalfd),
работа с таймерами с помощью файловых дескрипторов (timerfd), работа с семафорами с помощью файловых дескрипторов
(eventfd). Поэтому поддержка работы с процессами с помощью файловых дескрипторов выглядит логично.
Однако реализация оказалась намного труднее, чем предполагалось, поэтому поддержка управления процессами
с помощью файловых дескрипторов появилась относительно недавно и еще развивается. Мы кратко рассмотрим этот механизм.

### Получения файлового дескриптора процесса

Получить файловый дескриптор для процесса можно с помощью `pidfd_open`.

```c
#include <sys/syscall.h>

#ifndef __NR_pidfd_open
#define __NR_pidfd_open 434   /* System call # on most architectures */
#endif

static int
pidfd_open(pid_t pid, unsigned int flags)
{
    return syscall(__NR_pidfd_open, pid, flags);
}
```

Для `pidfd_open` еще не реализована поддержка в glibc, поэтому приходится использовать низкоуровневую функцию `syscall`
для системных вызовов напрямую.

Параметр `pid` - это идентификатор процесса, для которого нужно получить файловый дескриптор,
параметр `flags` должен быть равен 0.

Системный вызов возвращает файловый дескриптор, с помощью которого можно управлять процессом.
Для этого файлового дескриптора уже установлен флаг закрытия при exec (`O_CLOEXEC`).
При ошибке возвращается -1, и переменная `errno` устанавливается в код ошибки.

Использовать этот системный вызов можно следующим образом:

```c
    int pidfd = -1;
    pid_t pid = fork();
    if (pid > 0) {
        // в родителе получаем файловый дескриптор для процесса-сына
        pidfd = pidfd_open(pid, 0);
    }
```

Системный вызов `pidfd_open` вернет файловый дескриптор даже в случае, если процесс-сын успел завершиться
и находится в состоянии зомби. Однако если он уже был похоронен и его pid был освобожден для повторного использования,
остается возможность получить либо ошибку, либо доступ к совершенно другому процессу, то есть все равно сохраняется
окно для race condition. Следует заметить, тем не менее, что это возможно только в специальных случаях,
описанных выше в разделе про автоматическое освобождение pid, либо если был выполнен системный вызов семейства
`wait` в другой нити. При использовании в однопоточной программе без переустановки обработчика `SIGCHLD`
этот фрагмент совершенно корректен.

Если нужно гарантированно во всех случаях избежать race condition, следует использовать низкоуровневый
системный вызов `clone` с флагом `CLONE_PIDFD`.

### Отправка сигнала процессу по файловому дескриптору

Для отправки сигнала можно использовать системный вызов `pidfd_send_signal`. В данный момент для него не
реализована поддержка в glibc, поэтому нужно использовать низкоуровневый вызов `syscall` по аналогии с
`pidfd_open`.

```c
#include <signal.h>

int pidfd_send_signal(int pidfd, int sig, siginfo_t *info, unsigned int flags);
```

Параметр `pidfd` - это файловый дескриптор процесса, `sig` - отправляемый сигнал,
`info` - дополнительная информация, передаваемая с этим сигналом, как в вызове `rt_sigqueueinfo`.
Значение параметра `info` может быть `NULL`, тогда все поля структуры `siginfo_t` будут
заполнены автоматически как при простом системном вызове `kill`. Параметр `flags` должен быть равен 0.

### Ожидание завершения процесса

Аналогом системных вызовов `wait*` для файлового дескриптора процесса является системный вызов `waitid`.

```c
#include <sys/types.h>
#include <sys/wait.h>

int waitid(idtype_t idtype, id_t id, siginfo_t *infop, int options);
```

Для использования pidfd параметр `idtype` должен быть равен `P_PIDFD`, в параметре `id` передается
файловый дескриптор процесса, параметр `infop` может указывать на структуру, которая будет
заполнена в результате системного вызова. В параметре `options` передаются опции:

* `WEXITED` - ждать детей, которые завершились (системным вызовов или сигналом).
* `WSTOPPED` - срабатывать при приостановке выполнения процесса по сигналу `SIGSTOP`.
* `WCONTINUED` - срабатывать при продолжении выполнения процесса по сигналу `SIGCONT`.
* `WNOHANG` - не приостанавливать выполнение процесса если нет изменения состояния указанного процесса.
* `WNOWAIT` - не "хоронить" завершившийся процесс, он останется в состоянии "зомби".

Флаги объединяются с помощью операции "побитового или" `|`.

После завершения системного вызова `waitid` в структуре, на которую указывает `infop` будут заполнены следующие поля:
* `si_pid` - pid завершившегося процесса
* `si_uid` - реальный идентификатор пользователя завершившегося процесса
* `si_signo` - всегда равен `SIGCHLD`
* `si_status` - статус завершения, как в `wait`
* `si_code` - тип завершения:
** `CLD_EXITED` - по вызову `_exit`
** `CLD_KILLED` - из-за получения сигнала
** `CLD_DUMPED` - из-за завершения сигнала, дополнительно сгенерирован core-файл
** `CLD_STOPPED` - процесс остановлен из-за `SIGSTOP`
** `CLD_TRAPPED` - трассируемый процесс приостановлен
** `CLD_CONTINUED` - процесс продолжен с помощью `SIGCONT`

### Ожидания события на нескольких файловых дескрипторах

Файловый дескриптор процесса может добавляться во множество файловых дескрипторов, передаваемое
в системные вызовы `select`, `poll` или `epoll`. Когда состояние процесса изменится,
этот файловый дескриптор будет отмечен как готовый на чтение.

Однако выполнять системный вызов `read` для такого файлового дескриптора нельзя, для
получения нового статуса процесса следует использовать `waitid`.

Source: https://github.com/cmc-prak/cmc-os/tree/master/2021-2022
