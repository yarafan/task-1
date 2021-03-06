# Case-study оптимизации

## Актуальная проблема
В нашем проекте возникла серьёзная проблема.

Необходимо было обработать файл с данными, чуть больше ста мегабайт.

У нас уже была программа на `ruby`, которая умела делать нужную обработку.

Она успешно работала на файлах размером пару мегабайт, но для большого файла она работала слишком долго, и не было понятно, закончит ли она вообще работу за какое-то разумное время.

Я решил исправить эту проблему, оптимизировав эту программу.

## Формирование метрики
Для того, чтобы понимать, дают ли мои изменения положительный эффект на быстродействие программы я придумал использовать такую метрику: объем затрачиваемой памяти и время выполнения

## Гарантия корректности работы оптимизированной программы
Программа поставлялась с тестом. Выполнение этого теста позволяет не допустить изменения логики программы при оптимизации.

## Feedback-Loop
Для того, чтобы иметь возможность быстро проверять гипотезы я выстроил эффективный `feedback-loop`, который позволил мне получать обратную связь по эффективности сделанных изменений в среднем за 10 секунд

Вот как я построил `feedback_loop`:
1. Изначально выбрал маленький размер исходных данных (около 1Мб) позволяющий скрипту успешно отработать без оптимизаций
2. Поиск базовой метрики (время и память)
3. Дописал тест на регрессию по времени и памяти
4. Профилирование и поиск "точек роста"
5. Внесение изменений в код
6. Повторное тестирование и сбор новых метрик
7. Увеличение объема данных

## Вникаем в детали системы, чтобы найти 20% точек роста
Для того, чтобы найти "точки роста" для оптимизации я воспользовался

Гемы:
* ruby-prof
* memory_profiler
* get_process_mem

Stdlib:
* benchmark

Вот какие проблемы удалось найти и решить

### Находка №1
Считывание всего файла в строку с дальнейшим разбиением на массив строк  крайне неэффективно и было одним из основным блокеров для работы с большими файлами

Использование построчной обработки существенно ускорило выполнение программы примерно в 2,3 раза

### Находка №2
Memory_profiler показал, что аллоцируется огромное количество строк, хоть большинство из них собираются GC, однако все равно увеличивают объем потребляемой памяти и время выполнения (как минимум за счет работы GC)

Для исправления этой проблемы был использован ```# frozen_string_literal: true``` и в качестве ключей стали использоваться Symbol

### Находка №3
Избыточное и неоптимальное использование итераторов

По возможности, обработка данных переписана так, чтобы использовать минимальное число итераций и использоание "in-place" модификаций чтобы снизить расходы по памяти

### Находка №4
Генерация "сущностей без надобности"

Был убран класс User, а так же вместо отдельных массивов users и sessions был использован хеш ключем, которого стал user_id, что упростило связывание пользователей и сессий

### Находка №5
Неоптимальная аггрегация данных

Вместо того, чтобы агрегировать статистику по завершении обработки файла, код был переписан так, чтобы статистика аггрегировалась "по ходу" обработки файла, также для снижения потребляемой памяти, посчитанные данные удалялись

### Находка №6
Использование "медленных" методов без особой надобности

Ruby-prof показал, что метод Date#parse занимает порядка 8% от всего времени выполнения, поэтому он был заменен на Date#strptime

### Находка №7
Использование регулярных выражений без особой надобности

Поиск по регулярным выражениям был заменен на поиск по вхождению подстроки, поскольку он более производительный

## Результаты
В результате проделанной оптимизации наконец удалось обработать файл с данными.
В среднем файл обрабатывается за 29 - 30 секунд при средних затратах по памяти 850 - 860Мб

## Защита от регресса производительности
Для защиты от потери достигнутого прогресса при дальнейших изменениях программы добавлены дополнительные тесты для защиты от регрессий по памяти и времени выполнения
