# ESP32-HUB75-MatrixPanel-I2S-DMA-icn2053
Полностью переделанная библиотека на ESP32-HUB75-MatrixPanel-I2S-DMA (взят принцип) под ICN2053 (FM6353), и в отличии от ICN2053_ESP32_LedWall (взято число тактов OE для переключения строк) не потребляет все ресурсы ядра на прямой вывод в порты GPIO. Все параметры регистров конфигурации ICN2053 выдраны из программы LEDVISION для HUB75 контроллера.

Пока поддерживаются только драйверы ICN2053 и аналоги: FM6353.  

Принцип работы библиотеки:
  ICN2053 требует по 16 бит на каждый пиксел. С учетом что на каждые 2 бита пикселя требуется 1 слово (2 байта) DMA буфера - памяти в ESP32, если использовать полностью метод из ESP32-HUB75-MatrixPanel-I2S-DMA, то одна панель съест почти всю DMA память. Поэтому для изображения создается отдельный простой видео-буфер с нужной битностью (без необходимости поддержки DMA), а для DMA вывода выделяется буфер всего на несколько строк (1, 2 или 4). Готовность DMA канала для подготовки новых строк производится по DMA прерыванию.
Так как драйвер ICN2053 уже сам по себе содержит двойной буфер строк для двух кадров, а заполнение драйвера ведется в теневой буфер, то задержки с выводом влияют только на частоту смены кадров.
  Переключение буферов в ICN2053 буферов произовится DMA посылкой буфера префикса, содержащего команды и значений управляющих регистров для ICN2053 (регистры последовательно чередуются с каждой новый отправкой префикса).  
Каждая отправка DMA строк или префикса заканчевается зацикленной отправкой буфера суффикса, содержащего простой перебор адресов строк матрицы и наборов импульсов OE для переключения памяти строк в ICN2053. Начало перебора строк также содержится в буферах строк и префикса, поэтому их переход в суффикс происходит по смещению в последнем, чтобы перебор строк был непрерывным.
  Примечание: префикс и суффикс - потому-что подобным образом шли управляющие пакеты данных (в начале кадра и в конце ) у контроллера HUB75, но по факту в библиотеке prefix будет идти обычно после строк.
  Отправка буферов через DMA производится через наборы DMA дескрипторов. Для префикса и строк используется по одному DMA набору на буфер, буфер суффикса имеет два набора DMA дескрипторов, так как с окончания одного набора будет производится переход на вывод строк\префикса, а другой закольцовывается как конечный(и так попеременно). 
  
  Для уменьшения задержек вывода кадров (каждое заполнение DMA буфера строк из видео буфера занимает несколько милисекунд) опционально предусмотрены возможности: 
  1. двойной видеобуфер - работает стандартно, рисуем в одном буфере, выводим другой
  2. двойной буфер DMA строк (следующая готовится не дожидаясь вывода предыдущей) - немного сокращает задержки на выводах суффикса, зависит от скорости отрисовки нового кадра, размера буфера DMA строк (2,4 строки), частоты шины DMA, и в каком месте суффикса заканчивается вывод строк. 
  
Другие особенности:
  - так как драйвера ICN2053 имеют встроенную память, вывод изображения на матрицу всегдла производится как при двойной видео-буферизации с вызовом функции отправки\смены кадра.
  - Рабочая скорость DMA не более 13 МГц. Болше начинабтся проблемы с DMA или его прерываниями (по логическому анализатору исчезает часть данных в пакетах на HUB-75).
  - Так скорость перебора строк зависит только от числа строк в одной панели и частоты DMA, то уже на 5Мгц DMA скорость обновления изображения около 1кГц
  - Предельная скорость смены кадров для четырех панелей 64x32 около 30Гц, плазма демо с двойной буферизацией дает 20Гц
  - Библиотека теоретически поддерживает разную битность видеобуфера, но оставлена реализация только 16 бит, как оптимальная для adafruit_gfx, другие битности можно реализовать через свои буфера, функции отрисовки прямоугольника и передачи цвета пикселов в подготовку DMA строки. 
Примечание: так как подготовка DMA строки производится в обработке прерывания, то есть огранчения на ее длительность (увеличение размера буфера DMA строк увеличивает длительность обработка прерывания)
  - Поддержка драйверов без встроеной памяти лишь предусмотрена, но не реализована (простой копипакст из ESP32-HUB75-MatrixPanel-I2S-DMA не пройдет)  
  - Виртуальная матрица ESP32-VirtualMatrixPanel-I2S-DMA переделана как объект на базе MatrixPanel_DMA, вместо Adafruit_GFX\GFX, что позволило убрать дублирование методов. (в теории можно вообще виртуальную матрицу в MatrixPanel_DMA убрать) 
  - параметры матрицы либо задаются константами в ESP32-HUB75-MatrixPanel-config.h, либо через передачу своей стркутуры hub75_i2s_cfg_t
  - параметры драйверов можно просмотреть и некоторые изменить в ESP32-HUB75-MatrixPanel-leddrivers_v2.h
  
P.S.
С наворотами C++ не дружу, код по возможности в Паскаль-стиле, оптимизации компиляторов под микроконтроллеры не доверяю - поэтому многие базовые вещи ESP32-HUB75-MatrixPanel-I2S-DMA реализованы по другому.

