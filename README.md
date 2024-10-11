# Задача 3

В данной задаче предлагается реализовать функции приёма и передачи сообщения, с использованием техники bit stuffing
(что-то издали похожее на [PPP in HDLC-like Framing](https://www.rfc-editor.org/rfc/rfc1662))

Пусть имеется некий "канал" передачи данных, который может передавать данные побитово.
Чтобы передать последовательность байтов через такой "канал", сообщение переводится в последовательность битов следующим образом:

```
Исходное сообщение
0x77     0x72     0x69     0x74     0x65     0x5F     0x74     0x65     0x73     0x74
0      7 8      F 10    17 18    1F 20    27 28    2F 30    37 38    3F 40    47 48    4F // Номера битов в последовательности
01110111 01110010 01101001 01110100 01100101 01011111 01110100 01100101 01110011 01110100

Последовательность битов, которую хотим передать
01110111011100100110100101110100011001010101111101110100011001010111001101110100
```

Чтобы разграничивать передаваемые сообщения, перед полезной нагрузкой и после неё добавляется последовательность битов `01111110` (`0x7E`).
Будем называть их стартовым и конечным маркером.
Чтобы избежать коллизии полезной нагрузки и маркера границы, в полезной нагрузке после 5 идущих подряд битов `1` вставляется бит `0`.
Между границами двух сообщений могут встречаться только биты `1` (для дополнения сообщения до целого числа байтов используются биты `1`).
Таким образом, для передачи данного сообщения, в "канал" должна быть записана такая последовательность битов:

```
01111110011101110111001001101001011101000110010101011111001110100011001010111001101110100011111101111111
```

Так как в языке *C* нельзя оперировать отдельными битами, последовательность битов необходимо записывать и читать побайтово.
Заполнять байты битовой последовательностью можно двумя способами: от старшего бита к младшему и от младшего бита к старшему.
Реализовать заполнение байтов можно на выбор одним из этих способов.

- 1й вариант (Байты заполняются от старшего бита к младшему)
```
Последовательность байтов, которую передаём в "канал"
0      7 8      F 10    17 18    1F 20    27 28    2F 30    37 38    3F 40    47 48    4F 50    57 58    5F 60    67 // Номера битов в последовательности
01111110 01110111 01110010 01101001 01110100 01100101 01011111 00111010 00110010 10111001 10111010 00111111 01111111
    0x7E     0x77     0x72     0x69     0x74     0x65     0x5F     0x3A     0x32     0xB9     0xBA     0x3F     0x7F
```
- 2й вариант (Байты заполняются от младшего бита к старшему)
```
Последовательность байтов, которую передаём в "канал"
7      0 F      8 17    10 1F    18 27    20 2F    28 37    30 3F    38 47    40 4F    48 57    50 5F    58 67    60 // Номера битов в последовательности
01111110 11101110 01001110 10010110 00101110 10100110 11111010 01011100 01001100 10011101 01011101 11111100 11111110
    0x7E     0xEE     0x4E     0x96     0x2E     0xA6     0xFA     0x5C     0x4C     0x9D     0x5D     0xFC     0xFE
```

**Важно!!!** В реализации записи и чтения заполнение байтов последовательностью битов должно совпадать.
Тестирующая система 1 раз определяет выбранный способ и далее опирается на него.

Сигнатуры функций, которые надо реализовать, и максимальный размер полезной нагрузки в сообщении заданы в файле [include/protocol.h](include/protocol.h):

```c
int read_message(FILE *stream, void *buf);
```
- `stream` — "канал", из которого читается закодированное сообщение
- `buf` — адрес буфера, в который надо записать декодированное сообщение
- Функция возвращает число считанных в буфер байтов в случае успешной передачи.
  В случае возникновения ошибки (закодированное сообщение в канале содержит некорректную последовательность
  битов, полезная нагрузка содержит нецелое число байтов, ошибка чтения из канала), функция должна вывести
  в стандартный поток ошибок сообщение о возникшей ситуации и вернуть `EOF`.

```c
int write_message(FILE* stream, const void *buf, size_t nbyte);
```
- `stream` — "канал", в который записывается закодированное сообщение
- `buf` — адрес буфера, содержащего передаваемое сообщение
- `nbyte` — число байтов буфера, которые надо передать через "канал"
- Функция возвращает число переданных байтов буфера в случае успешной передачи.
  В случае возникновения ошибки при передаче функция должна вывести
  в стандартный поток ошибок сообщение о возникшей ситуации и вернуть `EOF`.

Примеры использования и ожидаемое поведение функций смотрите в тестах.

### Дополнительные соглашения:
- В случае использования глобальных переменных в какой-либо функции их нужно явно инициализировать в ней, 
  иначе возможна непредвиденная работа тестов. Но лучше их просто не использовать (и вообще неясно, 
  зачем они в этой задаче).
- Запись сообщения в "канал связи" производить только с помощью функции *putc(...)*.
- При записи сообщения биты `1` нужно дописывать только после маркера конца сообщения до целого байта:
  * добавленные до маркера начала приведут к ошибке тестирования;
  * добавленные после маркера конца сообщения в следующий байт приведут к ошибке.
- Чтение сообщения из "канала связи" производить только с помощью функции *getc(...)*.
- Другие функции работы с `FILE *` кроме `getc` и `putc` **не использовать**.
- При чтении из канала сообщение может начинаться с битов `1` (идущих до маркера начала), они должны быть проигнорированы.
- Вывод сообщения об ошибке в стандартный поток ошибок осуществлять функцией *fprintf(stderr, ...)*.

### Работа с GitHub:
- Работа с GitHub аналогична задачам 1 и 2
- [Ссылка на тестирующую систему](https://github.com/spbu-coding-2024/3-grading-system)
