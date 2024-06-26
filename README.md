# Домашнее задание №30 - C-REPL (6 баллов)

Вам предстоит реализовать REPL среду программирования для небольшого подмножества языка Си.

## REPL

_REPL (read-eval-print loop)_ - интерактивная среда программирования, похожая на командную строку,
с сохраняемым состоянием переменных между командами. В такой среде пользователь вводит выражения
(read), которые тут же начинают исполняться (eval), и их результат тут же выводится на экран
(print).

REPL среды популярны для многих языков программирования: Python, LISP, Scala, JavaScript и так
далее. Вам предстоит добавить язык Си в это множество.

## Подмножество языка

Мы ограничимся директивами препроцессора и определениями функций, определением переменных
(обязательно с инициализирующим выражением) и операцией присваивания в переменную выражения. Типы
переменных не содержат пробелов. Директивы, определения и выражения записываются в одну строку.
Вокруг оператора присваивания всегда будет по одному символу пробела. Некорректных выражений в
тестах не будет.

Таким образом в тестах будут только однострочные корректные выражения четырёх видов:
* `#...` - директивы препроцессора
* `T ident(...` - определение функции
* `T ident = ...` - определение переменной
* `ident = ...` - присваивание в переменную

Такие ограничения упростят анализ входного потока и позволят вам сконцентрироваться на существенной
части задачи - механизме интерпретации программы на языке Си.

## Интерпретация

REPL среда является интерпретатором языка программирования. Стандартный подход к интерпретации
состоит в полном синтаксическом и семантическом анализе входной программы, построении простейшего
внутреннего представления и исполнении этого представления шаг за шагом. Таким подходом
действительно можно реализовать эту задачу, но вам предлагается другой, гораздо более простой
способ.

Выражения, поданные во входящий поток, можно собрать в текст полноценной программы:
* директивы и определения функций друг за другом
* определения переменных и операции присваивания друг за другом внутри одной основной 
  _функции интерпретации_ 

В конце функции интерпретации можно дописать команду, печатающую значение последней определённой
или присвоенной переменной.

Входящий поток, записанный в таком виде в файл, может быть скомпилирован внешним компилятором в
разделяемый объект, затем этот объект может быть загружен вашей программой в собственное адресное
пространство, и функция интерпретации может быть позвана, используя ваши стандартные потоки
ввода-вывода.

## Пример по шагам

### Первая комманда

Допустим, на вход поступает команда:

```
int foo(int x) { return x * 2; }
```

Благодаря ограничениям на язык, можно легко понять, что это определение функции, после чего REPL
просто запоминает его, и переходит к чтению следующей команды.

### Вторая комманда

На вход поступает команда:

```
int a = foo(10);
```

Также легко понять, что это определение переменной, которую нужно теперь вычислить и напечатать
пользователю её значение. Для этого REPL составляет следующий текст программы:

```
#include <stdio.h>

int foo(int x) { return x * 2; }

void interpret_function() {
  int a = foo(10);

  printf("a: %d\n", a);
}
```

Обратите внимание на то, какие компоненты этого текста получены из входящих команд, а какие
созданы самим REPL. Этот текст записывается в файл (например, `source.c`) и затем компилируется
внешним компилятором в разделяемый объект. Например, компилятором GCC это можно сделать, вызвав
следующие команды (через fork-exec механизм):
```
gcc <arch-arg> -c -fpic source.c
gcc <arch-arg> -shared -o lib.so source.o
```

`<arch-arg>` равен либо `-m32`, либо `-m64`, в зависимости от битности вашего REPL приложения.
Битность приложения и разделяемого объекта `lib.so` обязательно должны совпадать.

Следующим шагом REPL программа загружает `lib.so` объект в собственное адресное пространство
посредством функции `dlopen`, находит в нём адрес функции `interpret_function` посредством функции
`dlsym` и затем вызывает её. Функция интерпретации напечатает:

```
a: 20
```

После этого REPL программе остаётся удалить объект `lib.so` из адресного пространства посредством
функции `dlclose` и перейти к чтению следующей команды из входного потока.

## Условие

REPL программа должна читать команды из входящего потока до тех пор, пока не встретится команда
`exit`. Если команда является определением переменной `X` или присваиванием в переменную `X`, REPL
программа должна проинтерпретировать накопленный набор команд и выдать значение переменной `X` в
формате `X: V`, где `V` - значение переменной `X`.

## Пример

### Входные данные

```
int x = 6;
int fib(int n) { return (n == 1 || n == 2) ? 1 : fib(n - 1) + fib(n - 2); }
x = fib(x);
int y = fib(x);
x = fib(y);
exit
```

### Выходные данные

```
x: 6
x: 8
y: 21
x: 10946

```
