# EV3_CLEVER_LIBRARY
Library for CLEVER for Lego EV3 brick with simple motions like go forward, backward, turn left/right with one/two motors

## Problem
CLEVER and competition participants usually don't share libraries because it will help opponents. Though, a lot of teams use SPIKE, still some teams use EV3 block. With this library, it will be much easier to solve some problems.




### Dulat_library

's1=0

s2=0

kp_line = 0

kd_line =0

@ki=0

@integral=0

kp_align = 0

kd_align = 0

@OldE = 0

kB = 0

kC = 0

curr_speed = 0

@DestEnc = 0

@prevTime = 0


@prevEnc = 0

@avgs=0

s1exists = "true"

s2exists = "true"

kp_align1 = 0

kd_align1 = 0


@kp_sync = 2.5

@TurnRatio = 0.6

@StopSignal = 0

@AsyncPort = "A"

@AsyncSpeed = 40

@AsyncDeg = 150
' -------------------------------------------
'''' Задать для управления 2 больших мотора. В - слева, С - справа
Function LargeBase()
  @kB = 1
  @kC = 1
EndFunction

' -------------------------------------------
'''' Задать для управления 2 средних мотора. В - слева, С - справа
Function MiddleBase()
  @kB = -1
  @kC = 1
EndFunction

' -------------------------------------------
'''' Задать для управления 2 средних мотора с шестерёнками. В - слева, С - справа
Function MiddleBase2()
  @kB = 1
  @kC = -1
EndFunction

' -------------------------------------------
'''' проверка, что все переменные заданы
Function Check()
  error = ""
  If @kB = 0 Or @kC = 0 Then
    error = "kb,kc=0"
  EndIf
  If error <> "" Then
    ' вывод ошибки
    Speaker.Play(100, "Error alarm")
    EV3.SetLEDColor("RED", "FLASH")

    ' ждём нажатия любой кнопки
    While Buttons.Current <> ""
    EndWhile
    While Buttons.Current = ""
    EndWhile
    Program.End()
  EndIf
EndFunction

' -------------------------------------------
'''' Движение по линии до Т-перекрёстка, 1-слева, 2-справа, В-слева, С-справа
Function LineT()
  Check()
  @OldE = 0
  @integral = 0  ' Сброс интегральной составляющей
  exit = 0
  
  ' Предварительное чтение датчиков
  If @s1exists = "true" Then
    @s1 = Sensor.ReadPercent(1)
  EndIf
  If @s2exists = "true" Then
    @s2 = Sensor.ReadPercent(2)
  EndIf
  
  While exit = 0
    ' Движение по линии с основным регулятором
    reg()
    
    ' Остановка на Т-перекрёстке: один из датчиков видит очень низкое значение яркости
    If @s1 < 11 Or @s2 < 11 Then
      ' Добавляем небольшую задержку и повторное чтение для подтверждения
    
      
      ' Перепроверяем показания датчиков
      If @s1exists = "true" Then
        @s1 = Sensor.ReadPercent(1)
      EndIf
      If @s2exists = "true" Then
        @s2 = Sensor.ReadPercent(2)
      EndIf
      
      ' Если показания все еще низкие, значит мы на перекрестке
      If @s1 < 15 Or @s2 < 15 Then
        exit = 1
      EndIf
    EndIf
    
      ' Небольшая задержка для предотвращения перегрузки процессора
  EndWhile
  
  Stop()
EndFunction

' -------------------------------------------
'''' Движение по линии до Х-перекрёстка, 1-слева, 2-справа, В-слева, С-справа
Function LineX()
  Check()
  @OldE = 0
  exit = 0
  While exit = 0
    reg()
    ' остановка на Х-перекрёстке: оба датчика видят маленькое значения яркости
    If @s1 < 20 And @s2 < 20 Then
      exit = 1
    EndIf
  EndWhile
EndFunction

' -------------------------------------------
'''' ПИД регулятор для 1 и 2 порта(колор сенсор)

Function reg()
  If @s1exists = "true" Then
    @s1 = Sensor.ReadPercent(1)
  Else
    @s1 = @avgS
  EndIf
  If @s2exists = "true" Then
    @s2 = Sensor.ReadPercent(2)
  Else
    @s2 = @avgS
  EndIf
  E = @s1 - @s2
  P = E * @kp_line
  I = @integral + (E * @ki_line)
  D = (E - @OldE) * @kd_line
  A = P + I + D
  @OldE = E
  integral = I
  SmoothSpeed()
 
  MotorB.StartPower((@curr_speed + A) * @kB)
  MotorC.StartPower((@curr_speed - A) * @kC)
EndFunction

' -------------------------------------------
'''' Ожидание нажатия кнопки
Function WaitButton(in string B)
  ' ждём, пока пользователь руку с этой кнопки уберёт
  While Buttons.Current = B
  EndWhile
  ' ожидание нажатия
  While Buttons.Current <> B
  EndWhile
EndFunction

' -------------------------------------------
'''' Выравнивание на чёрной линии
Function Align()
  Check()
  @OldE = 0
  exit = 0
  While exit = 0
    @s1 = Sensor.ReadPercent(1)
    @s2 = Sensor.ReadPercent(2)
    E = @s1 - @s2
    P = E * @kp_align1
    D = (E - @OldE) * @kd_align1
    A = P + D
    @OldE = E
    MotorB.StartPower((0 + A) * @kB)
    MotorC.StartPower((0 - A) * @kC)
    ' Если кажется, что слишком долго выравнивается, увеличиваем порог
    If Math.abs(E) < 1.6 Then
      exit = 1
      StopPower()
    EndIf
  EndWhile
EndFunction

' -------------------------------------------


' -------------------------------------------

Function Stop()
  Motor.Stop("BC", "true")
  @curr_speed = 0
  @prevEnc = 0
  @prevTime = 0
EndFunction

Function StopPower()
  MotorB.StartPower(0)
  MotorC.StartPower(0)
  @curr_speed = 0
  @prevEnc = 0
  @prevTime = 0
EndFunction
' -------------------------------------------
'''' Движение по линии на заданное расстояние DestEnc
Function LineEncoder(in number DestEnc)
  Check()
  ' Вариант 1 - сбрасывать значения энкодеров
  Motor.ResetCount("BC")
  @OldE = 0
  exit = 0
  While exit = 0
    reg()
    encB = Math.abs(MotorB.GetTacho())
    encC = Math.abs(MotorC.GetTacho())
    avg = (encB + encC)/2
    If avg > DestEnc Then
      exit = 1
    EndIf
  EndWhile
EndFunction

' -------------------------------------------
'''' Движение по линии на заданное расстояние DestEnc (относительный энкодер)
Function LineEncoder2(in number DestEnc)
  Check()
  ' Вариант 2 - запоминать начальное значение энкодеров
  ' Относительный энкодер
  StartB = MotorB.GetTacho()
  StartC = MotorC.GetTacho()
  @OldE = 0
  exit = 0
  While exit = 0
    reg()
    encB = Math.abs(MotorB.GetTacho() - StartB)
    encC = Math.abs(MotorC.GetTacho() - StartC)
    avg = (encB + encC)/2
    If avg > DestEnc Then
      exit = 1
    EndIf
  EndWhile
EndFunction

' -------------------------------------------
'''' Движение прямо на заданное расстояние
Function vpered(in number DestEnc)
    Check() ' Проверка конфигурации базы

    ' --- Инициализация для этого конкретного движения ---
    StartB = MotorB.GetTacho()          ' Начальное положение B
    StartC = MotorC.GetTacho()          ' Начальное положение C
    @OldE = 0                           ' Сброс предыдущей ОШИБКИ ВЫРАВНИВАНИЯ

    ' Используем While "True" и Break для цикла
    While "True"
        ' --- Расчет пройденного пути каждым мотором ---
        encB = MotorB.GetTacho() - StartB
        encC = MotorC.GetTacho() - StartC

        ' --- ПД-регулятор ВЫРАВНИВАНИЯ ---
        ' Расчет ошибки выравнивания
        E = encC * @kC - encB * @kB
        ' Расчет P и D компонент (ИСПОЛЬЗУЕМ _align коэффициенты!)
        P = E * @kp_align
        D = (E - @OldE) * @kd_align
        ' Расчет корректирующего воздействия
        A = P + D
        ' Обновление предыдущей ошибки для следующей итерации
        @OldE = E

        ' --- Плавный разгон ---
        SmoothSpeed() ' Обновляет глобальную @curr_speed до @speed

        ' --- Применение мощности к моторам ---
        ' Базовая скорость (@curr_speed) +/- коррекция (A)
        powerB = @curr_speed + A
        powerC = @curr_speed - A
        ' Ограничение мощности [-100, 100]
        powerB = Math.Min(100, Math.Max(-100, powerB))
        powerC = Math.Min(100, Math.Max(-100, powerC))
        ' Установка мощности
        MotorB.StartPower(powerB * @kB)
        MotorC.StartPower(powerC * @kC)

        ' --- Расчет среднего пройденного расстояния ---
        avg = (Math.Abs(encB) + Math.Abs(encC))/2

        ' --- Проверка условия выхода ---
        If avg >= DestEnc Then ' Используем >= для надежности
          Break ' Выход из цикла While
        EndIf

        ' --- Thread.Yield() НЕ ДОБАВЛЕН (по вашей инструкции) ---

    EndWhile ' Конец цикла While

    ' --- Stop() НЕ ДОБАВЛЕН (по вашей инструкции) ---
    ' !!! ПОМНИТЕ: Вызвать Stop() в основном коде после вызова vpered !!!

EndFunction

' -------------------------------------------
'''' Движение назад на заданное расстояние
Function nazad(in number power, in number DestEnc)
    Check() ' Проверка конфигурации базы

    ' --- Инициализация для этого конкретного движения ---
    StartB = MotorB.GetTacho()          ' Начальное положение B
    StartC = MotorC.GetTacho()          ' Начальное положение C
    @OldE = 0                           ' Сброс предыдущей ОШИБКИ ВЫРАВНИВАНИЯ
    ' Базовая мощность для движения назад (делаем отрицательной)
    baseBackwardPower = Math.Abs(power) * -1

    ' Используем While "True" и Break для цикла
    While "True"
        ' --- Расчет пройденного пути каждым мотором ---
        encB = MotorB.GetTacho() - StartB
        encC = MotorC.GetTacho() - StartC

        ' --- ПД-регулятор ВЫРАВНИВАНИЯ ---
        ' Расчет ошибки выравнивания (так же, как для vpered)
        E = encC * @kC - encB * @kB
        ' Расчет P и D компонент (ИСПОЛЬЗУЕМ _align коэффициенты!)
        P = E * @kp_align
        D = (E - @OldE) * @kd_align
        ' Расчет корректирующего воздействия
        A = P + D
        ' Обновление предыдущей ошибки для следующей итерации
        @OldE = E

        ' --- Применение мощности к моторам для движения НАЗАД ---
        ' Базовая ОТРИЦАТЕЛЬНАЯ мощность +/- коррекция (A)
        powerB = baseBackwardPower + A
        powerC = baseBackwardPower - A
        ' Ограничение мощности [-100, 100]
        powerB = Math.Min(100, Math.Max(-100, powerB))
        powerC = Math.Min(100, Math.Max(-100, powerC))
        ' Установка мощности
        MotorB.StartPower(powerB * @kB)
        MotorC.StartPower(powerC * @kC)

        ' --- Расчет среднего пройденного расстояния (по модулю) ---
        avg = (Math.Abs(encB) + Math.Abs(encC))/2

        ' --- Проверка условия выхода ---
        If avg >= DestEnc Then ' Используем >= для надежности остановки
          Break ' Выход из цикла While
        EndIf

        ' --- Thread.Yield() НЕ ДОБАВЛЕН (по вашей инструкции) ---

    EndWhile ' Конец цикла While

    ' --- Stop() НЕ ДОБАВЛЕН (по вашей инструкции) ---
    ' !!! ПОМНИТЕ: Моторы продолжат вращаться с последней заданной мощностью !!!
    ' !!! Вы должны вызвать Stop() в основном коде СРАЗУ после вызова nazad !!!

EndFunction

' -------------------------------------------
'''' Движение по линии на время в милисекундах
Function LineTime(in number Milis)
  ' измеряем относительное время (1-й отвечает за потоки, его не берём)
  startTime = Time.Get2()
  ' Time.Get2() - startTime = delta_t
  While Time.Get2() - startTime < Milis
    reg()
    @count ++
  EndWhile
EndFunction


' -------------------------------------------
'''' Поворот направо на определённый угол
Function Right(in number DestEnc)
  Check()
  StartB = MotorB.GetTacho()
  StartC = MotorC.GetTacho()
  @OldE = 0
  exit = 0
  While exit = 0
    SmoothSpeed()
    MotorB.StartPower(@curr_speed * @kB)
    MotorC.StartPower(-@curr_speed * @kC)
    encB = MotorB.GetTacho() - StartB
    encC = MotorC.GetTacho() - StartC
    ' суммируем с ABS - потому мы вращаемся воруг своей оси
    avg = (Math.Abs(encB) + Math.Abs(encC))/2 /2
    If avg > DestEnc Then
      exit = 1
    EndIf
  EndWhile
EndFunction

' -------------------------------------------
'''' Поворот налево на определённый угол
Function Left(in number DestEnc)
  check()
  StartB = MotorB.GetTacho()
  StartC = MotorC.GetTacho()
  @OldE = 0
  exit = 0
  While exit = 0
    SmoothSpeed()
    MotorB.StartPower(-@curr_speed * @kB)
    MotorC.StartPower(@curr_speed * @kC)
    ' всю работу с энкодерами можно взять из Straight
    encB = MotorB.GetTacho() - StartB
    encC = MotorC.GetTacho() - StartC
    ' суммируем с ABS - потому мы вращаемся воруг своей оси
    avg = (Math.Abs(encB) + Math.Abs(encC))/2 /2
    If avg > DestEnc Then
      exit = 1
    EndIf
  EndWhile
EndFunction

Function SmoothSpeed()
  max_speed = @speed
  If @curr_speed < max_speed Then
    If @prevTime = 0 Then
      @prevTime = Time.Get2()
    EndIf
    If @prevEnc = 0 Then
      encB = MotorB.GetTacho()
      encC = MotorC.GetTacho()
      @prevEnc = (Math.Abs(encB) + Math.Abs(encC))/2
    EndIf
    delta_time = Time.Get2() - @prevTime
    If delta_time > 10 Then
      encB = MotorB.GetTacho()
      encC = MotorC.GetTacho()
      enc = (Math.Abs(encB) + Math.Abs(encC))/2
      delta_enc = enc - @prevEnc
      If delta_enc > @DestEnc Then
        delta_enc = @DestEnc
      EndIf
      delta_speed = delta_enc * max_speed / @DestEnc
      If delta_speed < 1 Then
        delta_speed = 1 ' избавляемся от замкнутого круга приращений нуля
      EndIf
      @curr_speed += delta_speed
      ' Заполняем предыдущие значения
      @prevTime = Time.Get2()
      @prevEnc = enc
    EndIf
  Else
    @curr_speed = max_speed
  EndIf
EndFunction

Function SoloL(in number spd,in number DestEnc)
  Motor.Stop("BC", "true")
  StartB = MotorB.GetTacho()
  exit=0
   While exit=0
    MotorB.StartPower((spd) * @kB)
    encB = MotorB.GetTacho() - StartB
    avg = Math.Abs(encB) 
    If avg >= DestEnc Then
      exit = 1
    EndIf
    
  EndWhile
  
EndFunction

  

Function SoloR(in number spd,in number DestEnc)
    Motor.Stop("BC", "true")

   StartC = MotorC.GetTacho()
  exit=0
   While exit=0
        MotorC.StartPower((spd) * @kC)

    encC = MotorC.GetTacho() - StartC
    avg = Math.Abs(encC)
    If avg >= DestEnc Then
      exit = 1
    EndIf
    
  EndWhile
  

EndFunction

Function SmoothStop()
  now_speed = ( @speed + (MotorB.GetSpeed() + MotorC.GetSpeed()) / 2  )/2
  For i = 2 To 1 Step -1
    drop = now_speed * i / 3.0
    MotorB.StartPower(@kB * drop)
    MotorC.StartPower(@kC * drop)
    t = Time.Get2()
    exit = 0
    While exit = 0
      If (MotorB.GetSpeed() + MotorC.GetSpeed()) / 2 <= drop Then
        exit = 1
      EndIf
      If Time.Get2() - t > 10 Then
        exit = 1
      EndIf
    EndWhile
  EndFor
  MotorB.StartPower(0)
  MotorC.StartPower(0)
  @curr_speed = 0
  @prevTime = 0
  @prevEnc = 0
EndFunction

''''--------специфические функции-----------------

Function reg2()
  
  @s1 = Sensor.ReadPercent(3)
  @s1 = @avgS
  @s2 = Sensor.ReadPercent(4)
  @s2 = @avgS
  E = @s1 - @s2
  P = E * @kp
  I = @integral + (E * @ki)
  D = (E - @OldE) * @kd
  A = P + I + D
  OldE = E
  integral = I
  SmoothSpeed()
  
  MotorB.StartPower((@curr_speed + A) * @kB)
  MotorC.StartPower((@curr_speed - A) * @kC)
EndFunction

Function 2CompasEncoder(in number DestEnc)
  Check()
  ' Вариант 2 - запоминать начальное значение энкодеров
  ' Относительный энкодер
  StartB = MotorB.GetTacho()
  StartC = MotorC.GetTacho()
  @OldE = 0
  exit = 0
  While exit = 0
    reg2()
    encB = Math.abs(MotorB.GetTacho() - StartB)
    encC = Math.abs(MotorC.GetTacho() - StartC)
    avg = (encB + encC)/2
    If avg > DestEnc Then
      exit = 1
    EndIf
  EndWhile
EndFunction

Function 1CompasEncoder(in number DestEnc, in number angle)
  Check()
  ' Вариант 2 - запоминать начальное значение энкодеров
  ' Относительный энкодер
  kp=1
  rel = 0
  StartB = MotorB.GetTacho()
  StartC = MotorC.GetTacho()
  @OldE = 0
  exit = 0
  While exit = 0
    @s1 = rel
    
    E = angle + rel
    P = E * kp
    I = @integral + (E * @ki)
    D = (E - @OldE) * @kd
    A = P + I + D
    OldE = E
    integral = I
    SmoothSpeed()
    
    MotorB.StartPower((@curr_speed + A) * @kB)
    MotorC.StartPower((@curr_speed - A) * @kC)
    encB = Math.abs(MotorB.GetTacho() - StartB)
    encC = Math.abs(MotorC.GetTacho() - StartC)
    
    avg = (encB + encC)/2
    If avg > DestEnc Then
      exit = 1
    EndIf
  EndWhile
  
  
  
EndFunction

Function timevp(in number Milis)
  Time.Reset2()
  startTime = Time.Get2()
  Check()
  ' Проблема - накопление энкодера
  StartB = MotorB.GetTacho()
  StartC = MotorC.GetTacho()
  @OldE = 0
  exit = 0
  While Time.Get2() < Milis
    encB = MotorB.GetTacho() - StartB
    encC = MotorC.GetTacho() - StartC
    E = encC * @kC - encB * @kB
    P = E * @kp_align
    D = (E - @OldE) * @kd_align
    A = P + D
    @OldE = E
    SmoothSpeed()
    MotorB.StartPower((@curr_speed + A) * @kB)
    MotorC.StartPower((@curr_speed - A) * @kC)

    
    
  EndWhile
EndFunction

Function timevpnazad(in number Milis)
  Time.Reset2()
  startTime = Time.Get2()
  Check()
  ' Проблема - накопление энкодера
  StartB = MotorB.GetTacho()
  StartC = MotorC.GetTacho()
  @OldE = 0
  exit = 0
  While Time.Get2() < Milis
    encB = MotorB.GetTacho() - StartB
    encC = MotorC.GetTacho() - StartC
    E = encC * @kC - encB * @kB
    P = E * @kp_align
    D = (E - @OldE) * @kd_align
    A = P + D
    @OldE = E
    SmoothSpeed()
    MotorB.StartPower((@curr_speed - A) * -@kB)
    MotorC.StartPower((@curr_speed + A) * -@kC)

    
    
  EndWhile
EndFunction

'''------параллельный чек бриков для ВРО 

'чекер будет настраиваться персонально под каждую задачу
'первоночально нам нужно замерить расстояние между бриками,
'после чего нам нужно идеально ровно приезжать на одну и ту же точку возле бриков

'работа чекера будет основываться на записи определенных цветовых значений в определенных значениях энкодеров моторов

'1)rast - расстояние между бриками
'2)firstrast - расстояние с начальной точки до бриков
'color sensor port 4
'brick1b = 0 
'brick2b = 0 
'brick3b = 0
'brick4b = 0
'Function checker(in number Rast, in number firstRast, out brick1b, out brick2b, out brick3b, out, brick4b )
  'Check()
  
  'DestEnc= Rast * 4 + firstRast+10
  'StartB = MotorB.GetTacho()
  'StartC = MotorC.GetTacho()
  '@OldE = 0
  'exit = 0
  'While exit = 0
    'reg()
    'encB = Math.abs(MotorB.GetTacho() - StartB)
    'encC = Math.abs(MotorC.GetTacho() - StartC)
    'avg = (encB + encC)/2
    
    'If avg>= firstRast-5 And avg<= firstRast+5 Then
      'Color.getRGBW(4, red, green , blue, white )
      'brick1r= red
      'brick1g = green
      'brick1b = blue
      'brick1w = white
      
    'ElseIf avg>= firstRast+rast -5 And avg<= firstRast+rast +5 Then
      'Color.getRGBW(4, red, green , blue, white )
      'brick2r= red
      'brick2g = green
      'brick2b = blue
      'brick2w = white
      
    'ElseIf avg>= firstRast+2*rast -5 And avg<= firstRast+2*rast +5 Then
      'Color.getRGBW(4, red, green , blue, white )
      'brick3r= red
      'brick3g = green
      'brick3b = blue
      'brick3w = white
      
    'ElseIf avg>= firstRast+3*rast -5 And avg<= firstRast+3*rast +5 Then
      'Color.getRGBW(4, red, green , blue, white )
      'brick4r= red
      'brick4g = green
      'brick4b = blue
      'brick4w = white
      
    'EndIf
    
    'If avg > DestEnc Then
      'exit = 1
    'EndIf
  'EndWhile
  
'EndFunction







' ====================================================================
'''' Асинхронно запускает ЧЕТЫРЕ мотора (A, B, C, D) на заданные углы
'''' с индивидуальной мощностью и ОЖИДАЕТ ЗАВЕРШЕНИЯ движения ВСЕХ моторов.
'
'   Принцип: Каждый мотор получает свою задачу через Motor.SchedulePower,
'            которая запускается асинхронно. Затем функция ожидает, пока
'            все четыре мотора не завершат свои задачи, используя Motor.Wait.
'
'   Параметры:
'     angleA, angleB, angleC, angleD: Целевые углы поворота для моторов A, B, C, D
'                                     в градусах. Функция возьмет модуль угла,
'                                     а направление будет определяться знаком мощности.
'     powerA, powerB, powerC, powerD: Мощность для моторов A, B, C, D (от -100 до 100).
'                                     Знак определяет направление вращения.
'
'   Использует стандартные функции EV3 Basic:
'     Motor.SchedulePower, Motor.Wait, Math.Abs, Math.Min, Math.Max
' ====================================================================
Function AsyncFourMotors(in number angleA, in number powerA, in number angleB, in number powerB, in number angleC, in number powerC, in number angleD, in number powerD)

    ' --- Подготовка параметров для SchedulePower ---
    ' Углы (зоны) для SchedulePower должны быть положительными
    absAngleA = Math.Abs(angleA)
    absAngleB = Math.Abs(angleB)
    absAngleC = Math.Abs(angleC)
    absAngleD = Math.Abs(angleD)

    ' Ограничиваем мощность диапазоном от -100 до 100 для надежности
    pA = Math.Min(100, Math.Max(-100, powerA))
    pB = Math.Min(100, Math.Max(-100, powerB))
    pC = Math.Min(100, Math.Max(-100, powerC))
    pD = Math.Min(100, Math.Max(-100, powerD))

    ' Режим торможения в конце для точности
    brakeMode = "True"

    ' --- Шаг 1: Асинхронный запуск всех четырех моторов ---
    ' Используем SchedulePower с нулевыми зонами разгона/торможения (zone1=0, zone3=0)
    ' и основной зоной движения (zone2), равной целевому углу.
    ' Каждый вызов команды возвращает управление немедленно.
    Motor.SchedulePower("A", pA, 0, absAngleA, 0, brakeMode)
    Motor.SchedulePower("B", pB, 0, absAngleB, 0, brakeMode)
    Motor.SchedulePower("C", pC, 0, absAngleC, 0, brakeMode)
    Motor.SchedulePower("D", pD, 0, absAngleD, 0, brakeMode)

    ' --- Шаг 2: Ожидание завершения работы ВСЕХ запущенных моторов ---
    ' Команда Motor.Wait блокирует дальнейшее выполнение программы до тех пор,
    ' пока ВСЕ моторы, указанные в строке ("ABCD"), не завершат свои
    ' текущие задачи (запущенные через Schedule или Move).
    Motor.Wait("ABCD")

    ' --- Все четыре мотора завершили свое движение ---
    ' Функция выполнила свою задачу и может завершиться.

EndFunction

Function AllMotorsMove(in number B_dist, in number C_dist, in number D_dist, in number A_dist, in number speed)
  Check()
  
  ' Сброс энкодеров и инициализация
  Motor.ResetCount("BCDA")
  StartB = MotorB.GetTacho()
  StartC = MotorC.GetTacho()
  StartD = MotorD.GetTacho()
  StartA = MotorA.GetTacho()
  
  ' Флаги завершения для каждого мотора
  B_done = 0
  C_done = 0
  D_done = 0
  A_done = 0
  
  ' Настройка направления для каждого мотора
  B_dir = 1
  C_dir = 1
  D_dir = 1
  A_dir = 1
  
  If B_dist < 0 Then
    B_dir = -1
    B_dist = Math.Abs(B_dist)
  EndIf
  
  If C_dist < 0 Then
    C_dir = -1
    C_dist = Math.Abs(C_dist)
  EndIf
  
  If D_dist < 0 Then
    D_dir = -1
    D_dist = Math.Abs(D_dist)
  EndIf
  
  If A_dist < 0 Then
    A_dir = -1
    A_dist = Math.Abs(A_dist)
  EndIf
  
  ' Основной цикл движения
  While B_done + C_done + D_done + A_done < 4
    ' Мотор B
    If B_done = 0 Then
      encB = Math.Abs(MotorB.GetTacho() - StartB)
      If encB >= B_dist Then
        B_done = 1
        Motor.Stop("B","True")
      Else
        MotorB.StartPower(speed * B_dir * @kB)
      EndIf
    EndIf
    
    ' Мотор C
    If C_done = 0 Then
      encC = Math.Abs(MotorC.GetTacho() - StartC)
      If encC >= C_dist Then
        C_done = 1
        Motor.Stop("C","True")
      Else
        MotorC.StartPower(speed * C_dir * @kC)
      EndIf
    EndIf
    
    ' Мотор D
    If D_done = 0 Then
      encD = Math.Abs(MotorD.GetTacho() - StartD)
      If encD >= D_dist Then
        D_done = 1
        Motor.Stop("D","true")
      Else
        MotorD.StartPower(speed * D_dir)
      EndIf
    EndIf
    
    ' Мотор A
    If A_done = 0 Then
      encA = Math.Abs(MotorA.GetTacho() - StartA)
      If encA >= A_dist Then
        A_done = 1
        Motor.Stop("A","True")
      Else
        MotorA.StartPower(speed * A_dir)
      EndIf
    EndIf
    
    Program.Delay(5)
  EndWhile
EndFunction

Function checkerEncoder(in number drivePower, in number kp_align, in number kd_align, in number Rast, in number firstRast)
    Check() ' Проверка конфигурации базы

    ' --- Инициализация ---
    ' Дистанция, чтобы проехать мимо центра 4-го брика (индекс 3) + запас
    DestEnc = firstRast + Rast * 3 + 50
    StartB = MotorB.GetTacho()
    StartC = MotorC.GetTacho()
    @OldE = 0   ' Сброс состояния ПД выравнивания
    exit = 0    ' Флаг выхода из цикла

    ' Локальные переменные для цикла
    local encB, encC, avg, currentB, currentC, distB, distC
    local E, P, D, A, powerB, powerC
    local red, green, blue, white
    local window = 5 ' Окно сканирования (+/- 5 градусов)


    ' --- Основной цикл ---
    While exit = 0
        ' --- Шаг 1: Расчет и применение ПД-выравнивания ---
        currentB = MotorB.GetTacho()
        currentC = MotorC.GetTacho()
        ' Относительный поворот для расчета ошибки
        encB_relative_err = (currentB - startB) * @kB
        encC_relative_err = (currentC - startC) * @kC
        E = encC_relative_err - encB_relative_err ' Ошибка выравнивания
        P = E * kp_align         ' P-член (используем параметры kp/kd)
        D = (E - @OldE) * kd_align ' D-член (используем параметры kp/kd)
        A = P + D                ' Коррекция
        @OldE = E                ' Обновляем предыдущую ошибку

        powerB = drivePower + A  ' Мощность с учетом коррекции
        powerC = drivePower - A
        powerB = Math.Min(100, Math.Max(-100, powerB)) ' Ограничение
        powerC = Math.Min(100, Math.Max(-100, powerC))
        MotorB.StartPower(powerB * @kB) ' Применяем мощность
        MotorC.StartPower(powerC * @kC)

        ' --- Шаг 2: Расчет пройденной дистанции ---
        ' Для дистанции используем абсолютные значения
        encB = currentB - StartB
        encC = currentC - StartC
        avg = (Math.Abs(encB) + Math.Abs(encC))/2 ' Среднее АБСОЛЮТНОЕ расстояние

        ' --- Шаг 3: Проверка цвета в окнах ---
      

        ' --- Шаг 4: Проверка выхода по общей дистанции ---
        If avg >= DestEnc Then ' Используем >=
           exit = 1
        EndIf

        ' --- Thread.Yield() НЕ ДОБАВЛЕН (по инструкции) ---

    EndWhile ' Конец основного цикла While

    ' --- Stop() НЕ ДОБАВЛЕН (по инструкции) ---
    ' !!! Робот продолжит движение! Вызвать Stop() в MAIN !!!


EndFunction
Function checkerEncoder_6slots(in number drivePower, in number kp_align, in number kd_align, in number Rast, in number firstRast)
    Check() ' Проверка конфигурации базы

    ' --- Инициализация ---
    ' !!! ИЗМЕНЕНО: Дистанция для проезда мимо 6-го слота (индекс 5) + запас 50 градусов !!!
    DestEnc = firstRast + Rast * 5 + 50
    StartB = MotorB.GetTacho()
    StartC = MotorC.GetTacho()
    @OldE = 0   ' Сброс состояния ПД выравнивания
    exit = 0    ' Флаг выхода из цикла

    'local encB, encC, avg, currentB, currentC, distB, distC ' Локальные для цикла
    'local E, P, D, A, powerB, powerC            ' Локальные для ПД
    'local red, green, blue, white             ' Локальные для цвета
    'local window = 5                            ' Окно сканирования (+/- 5) - МОЖЕТ БЫТЬ СЛИШКОМ МАЛЕНЬКОЕ!

    ' --- Основной цикл ---
    While exit = 0
        ' --- Шаг 1: Расчет и применение ПД-выравнивания ---
        currentB = MotorB.GetTacho()
        currentC = MotorC.GetTacho()
        encB_relative = (currentB - startB) * @kB
        encC_relative = (currentC - startC) * @kC
        E = encC_relative - encB_relative       ' Ошибка выравнивания
        P = E * kp_align                        ' P-член
        D = (E - @OldE) * kd_align              ' D-член
        A = P + D                               ' Коррекция
        @OldE = E                               ' Обновляем предыдущую ошибку

        powerB = drivePower + A                 ' Мощность с учетом коррекции
        powerC = drivePower - A
        powerB = Math.Min(100, Math.Max(-100, powerB)) ' Ограничение
        powerC = Math.Min(100, Math.Max(-100, powerC))
        MotorB.StartPower(powerB * @kB)         ' Применяем мощность
        MotorC.StartPower(powerC * @kC)

        ' --- Шаг 2: Расчет пройденной дистанции ---
        encB = currentB - StartB
        encC = currentC - StartC
        avg = (Math.Abs(encB) + Math.Abs(encC))/2 ' Среднее АБСОЛЮТНОЕ расстояние

        ' --- Шаг 3: Проверка цвета в окнах для 6 слотов ---
        

        ' --- Шаг 4: Проверка выхода по общей дистанции ---
        If avg >= DestEnc Then ' Используем >=
           exit = 1
        EndIf

        ' --- Thread.Yield() НЕ ДОБАВЛЕН (по инструкции) ---

    EndWhile ' Конец основного цикла While

    ' --- Stop() НЕ ДОБАВЛЕН (по инструкции) ---
    ' !!! Робот продолжит движение! Вызвать Stop() в MAIN !!!

EndFunction



' --- Необходимые импорты (в начале основного файла) ---
' import "Mods\AdvMtCtrls"
' import "Mods\Tool" ' Опционально, для constrain

' --- Глобальные переменные ---
' @kB, @kC (должны быть установлены Base... функцией)

' ====================================================================
'''' Функция "ИДЕАЛЬНОГО" движения ПРЯМО на заданное расстояние.
'    Использует контроллер ускорения/замедления AccTwoEnc из AdvMtCtrls
'    и ПД-регулятор выравнивания по энкодерам.
'    Останавливается точно на заданной дистанции с плавным замедлением.
'    ВАЖНО: НЕ содержит Thread.Yield и НЕ останавливает моторы в конце!
'
'   Параметры:
'     totalDistance:  Общее расстояние, которое нужно проехать (градусы энкодера).
'     maxPower:       Максимальная мощность (0-100) на фазе постоянной скорости.
'     minPower:       Минимальная мощность (>0) для старта и конца замедления.
'     accelDecelDist: Расстояние (градусы энкодера) для фазы ускорения И замедления.
'                     (Должно быть меньше totalDistance / 2).
'     kp_align:       Kp для ПД-выравнивания (ПОДОБРАТЬ!).
'     kd_align:       Kd для ПД-выравнивания (ПОДОБРАТЬ!).
'
'   Использует глобальные: @kB, @kC.
'   Использует функции: Check(), Stop() (из вашей библиотеки).
'                     AdvMtCtrls.AccTwoEnc_Config, AdvMtCtrls.AccTwoEnc
'                     Tool.constrain (опционально)
'   Использует стандартные: MotorB/C.GetTacho/StartPower, Math.*
' ====================================================================

Function accel_move(in number totalDistance, in number accelDistance, in number decelDistance, in number minSpeed, in number maxSpeed)
  

  Check()
  ' Сброс и инициализация
  Motor.ResetCount("BC")
  StartB = MotorB.GetTacho()
  StartC = MotorC.GetTacho()
  @OldE = 0
  exit = 0

  While exit = 0
    ' Текущее пройденное расстояние
    encB_abs = Math.Abs(MotorB.GetTacho() - StartB)
    encC_abs = Math.Abs(MotorC.GetTacho() - StartC)
    avgDist = (encB_abs + encC_abs) / 2
    remainingDist = totalDistance - avgDist

    ' Расчет целевой скорости для текущего шага с плавным переходом
    current_target_speed = 0
    If accelDistance > 0 And avgDist < accelDistance Then
      ' Косинусоидальное сглаживание ускорения: S-образная кривая
      ratio = avgDist / accelDistance
      current_target_speed = minSpeed + (maxSpeed - minSpeed) * (1 - math.Cos(ratio * Math.Pi)) / 2
    ElseIf decelDistance > 0 And remainingDist < decelDistance Then
      ' Косинусоидальное сглаживание замедления
      ratio = remainingDist / decelDistance
      current_target_speed = minSpeed + (maxSpeed - minSpeed) * (1 - math.Cos(ratio * math.pi)) / 2
    Else
      ' Фаза постоянной скорости
      current_target_speed = maxSpeed
    EndIf

    ' Ограничение рассчитанной скорости
    If current_target_speed < minSpeed Then
      current_target_speed = minSpeed
    EndIf
    If current_target_speed > maxSpeed Then
      current_target_speed = maxSpeed
    EndIf

    ' Если почти доехали, гарантируем минимальную скорость (не останавливаясь полностью)
    If remainingDist <= 2 Then
       current_target_speed = minSpeed
      If current_target_speed < 1 Then 
        current_target_speed = 1 
      EndIf
    EndIf

    ' Выравнивание по энкодерам (ПД-регулятор)
    encB = MotorB.GetTacho() - StartB
    encC = MotorC.GetTacho() - StartC
    E = encC * @kC - encB * @kB
    P = E * @kp_accel
    D = (E - @OldE) * @kd_accel
    A = P + D
    @OldE = E

    ' Управление моторами
    MotorB.StartPower((current_target_speed + A) * @kB)
    MotorC.StartPower((current_target_speed - A) * @kC)

    ' Условие выхода
    If avgDist >= totalDistance Then
      exit = 1
    EndIf

    Program.Delay(10)
  EndWhile

  StopPower()
EndFunction

Function uni(in number B_M, in number C_M, in number DestEnc)
  StartB = MotorB.GetTacho()
  StartC = MotorC.GetTacho()
  exit = 0
  While exit = 0
    MotorB.StartPower((B_M) * @kB)
    MotorC.StartPower((C_M) * @kC)
    encB = MotorB.GetTacho() - StartB
    encC = MotorC.GetTacho() - StartC
    avg = (Math.Abs(encB) + Math.Abs(encC)) / 2
    If avg >= DestEnc Then
      exit = 1
      StopPower()
    EndIf
  EndWhile
EndFunction



Function accel_move_backward(in number totalDistance, in number accelDistance, in number decelDistance, in number minSpeed, in number maxSpeed)
  Check()
  ' Сброс и инициализация
  Motor.ResetCount("BC")
  StartB = MotorB.GetTacho()
  StartC = MotorC.GetTacho()
  @OldE = 0
  exit = 0
  
  While exit = 0
    ' Текущее пройденное расстояние (отрицательные значения для заднего хода)
    encB_abs = Math.Abs(MotorB.GetTacho() - StartB)
    encC_abs = Math.Abs(MotorC.GetTacho() - StartC)
    avgDist = (encB_abs + encC_abs) / 2
    remainingDist = totalDistance - avgDist
    
    ' Расчет целевой скорости для текущего шага с плавным переходом
    current_target_speed = 0
    If accelDistance > 0 And avgDist < accelDistance Then
      ' Косинусоидальное сглаживание ускорения: S-образная кривая
      ratio = avgDist / accelDistance
      current_target_speed = minSpeed + (maxSpeed - minSpeed) * (1 - Math.Cos(ratio * Math.Pi)) / 2
    ElseIf decelDistance > 0 And remainingDist < decelDistance Then
      ' Косинусоидальное сглаживание замедления
      ratio = remainingDist / decelDistance
      current_target_speed = minSpeed + (maxSpeed - minSpeed) * (1 - Math.Cos(ratio * Math.Pi)) / 2
    Else
      ' Фаза постоянной скорости
      current_target_speed = maxSpeed
    EndIf
    
    ' Ограничение рассчитанной скорости
    If current_target_speed < minSpeed Then
      current_target_speed = minSpeed
    EndIf
    If current_target_speed > maxSpeed Then
      current_target_speed = maxSpeed
    EndIf
    
    ' Если почти доехали, гарантируем минимальную скорость
    If remainingDist <= 2 Then
       current_target_speed = minSpeed
      If current_target_speed < 1 Then 
        current_target_speed = 1 
      EndIf
    EndIf
    
    ' Выравнивание по энкодерам (ПД-регулятор)
    encB = MotorB.GetTacho() - StartB
    encC = MotorC.GetTacho() - StartC
    E = encC * @kC - encB * @kB
    P = E * @kp
    D = (E - @OldE) * @kd
    A = P + D
    @OldE = E
    
    ' Управление моторами в обратном направлении (негативные значения для движения назад)
    MotorB.StartPower((-current_target_speed + A) * @kB)
    MotorC.StartPower((-current_target_speed - A) * @kC)
    
    ' Условие выхода
    If avgDist >= totalDistance Then
      exit = 1
    EndIf
    
    Program.Delay(10)
  EndWhile
  
  StopPower()
EndFunction


'LOL
'
'

' ========= Г Л О Б А Л Ь Н Ы Е  П Е Р Е М Е Н Н Ы Е ==========

' PID‐коэффициенты для простого движения вперёд
@Kp_fwd      = 0.6     ' пропорциональный
@Ki_fwd      = 0.002   ' интегральный
@Kd_fwd      = 0.1     ' дифференциальный

@dt_ms       = 20      ' шаг цикла, мс (50 Гц)
@eps_enc     = 3       ' допустимая погрешность по энкодерам, градусы

' Направление моторов B и C (1 или –1)
' Установить в LargeBase()/MiddleBase() или вручную

' Вспомогательные PID-переменные
OldErr_fwd  = 0
IntErr_fwd  = 0
PrevEnc_fwd = 0
PrevTime_fwd= 0

' =========================================


' === Простая функция: едем вперед на DestDeg градусов колёс ===
Function DriveForwardBasic(in number DestDeg)
  Check()
  ' Сбрасываем PID-переменные
  @OldErr_fwd   = 0
  @IntErr_fwd   = 0
  @PrevEnc_fwd  = 0
  @PrevTime_fwd = 0

  ' Сбрасываем энкодеры и запоминаем старт
  Motor.ResetCount("BC")
  startB = MotorB.GetTacho()
  startC = MotorC.GetTacho()

  ' Целевая позиция (усреднённая)
  @PrevTime_fwd = Time.Get2()
  i =1 
  While i=1
    currentTime = Time.Get2()
    dt = currentTime - @PrevTime_fwd
    If dt < @dt_ms Then
      Program.Delay(@dt_ms - dt)
      currentTime = Time.Get2()
      dt = currentTime - @PrevTime_fwd
    EndIf

    ' Текущие энкодеры
    currB = MotorB.GetTacho() - startB
    currC = MotorC.GetTacho() - startC
    enc_avg = (currB * @kB + currC * @kC) / 2

    ' Ошибка по позиции
    ErrP = DestDeg - enc_avg

    ' Интеграл ошибки
    @IntErr_fwd = @IntErr_fwd + ErrP * dt

    ' Дифференциал ошибки
    dErr = 0
    If dt > 0 Then
      dErr = (ErrP - @OldErr_fwd) / dt
    EndIf

    ' PID-коррекция
    u = @Kp_fwd * ErrP + @Ki_fwd * @IntErr_fwd + @Kd_fwd * dErr

    ' Ограничиваем мощность
    If u > 100 Then
      u = 100
    ElseIf u < -100 Then
      u = -100
    EndIf

    ' Посылаем моторы
    MotorB.StartPower(u * @kB)
    MotorC.StartPower(u * @kC)

    ' Сохраняем для следующей итерации
    @OldErr_fwd   = ErrP
    @PrevEnc_fwd  = enc_avg
    @PrevTime_fwd = Time.Get2()

    ' Завершение, когда близко к цели
    If Math.Abs(ErrP) < @eps_enc Then
      @i = 0
    EndIf
  EndWhile

EndFunction



Function LineEncoderEnhanced(in number DestEnc)
  Check()
  @OldE = 0
  @integral = 0
  exit = 0
  
  ' Remember initial encoder values
  StartB = MotorB.GetTacho()
  StartC = MotorC.GetTacho()
  
  ' Advanced motion profiling
  accel_dist = Math.Min(DestEnc * 0.25, 100)
  decel_dist = Math.Min(DestEnc * 0.35, 120)
  
  ' Dynamic PID parameters
  kp_straight = @kp * 1.1  ' Higher P for straights
  kp_curve = @kp * 1.3    ' Even higher P for curves
  kd_straight = @kd * 1.2  ' Higher D for straights
  
  ' Performance tracking
  last_time = Time.Get2()
  last_distance = 0
  
  While exit = 0
    ' Current distance calculation
    encB = Math.abs(MotorB.GetTacho() - StartB)
    encC = Math.abs(MotorC.GetTacho() - StartC)
    current_distance = (encB + encC) / 2
    remaining = DestEnc - current_distance
    
    ' Speed profiling with enhanced curves
    If current_distance < accel_dist Then
      ' Acceleration phase - quadratic profile for smooth start
      progress = current_distance / accel_dist
      target_speed = 20 + (@speed - 20) * (progress * progress)
    ElseIf remaining < decel_dist Then
      ' Deceleration phase - quadratic profile for smooth stop
      progress = remaining / decel_dist
      target_speed = 20 + (@speed - 20) * (progress * progress)
    Else
      ' Constant speed phase
      target_speed = @speed
    EndIf
    
    ' Smooth speed adjustment (limit acceleration/deceleration)
    If @curr_speed < target_speed Then
      @curr_speed = Math.Min(@curr_speed + 2, target_speed)
    ElseIf @curr_speed > target_speed Then
      @curr_speed = Math.Max(@curr_speed - 2, target_speed)
    EndIf
    
    ' Advanced line following with sensor reading
    If @s1exists = "true" Then
      @s1 = Sensor.ReadPercent(1)
    EndIf
    If @s2exists = "true" Then
      @s2 = Sensor.ReadPercent(2)
    EndIf
    
    ' Error calculation
    E = @s1 - @s2
    
    ' Dynamic PID based on error magnitude (curve detection)
    current_kp = kp_straight
    current_kd = kd_straight
    
    If Math.Abs(E) > 10 Then
      ' We're on a curve - use curve parameters
      current_kp = kp_curve
      
      ' Adjust speed on curves for better tracking
      curve_factor = 1 - Math.Min(0.3, (Math.Abs(E) - 10) / 30)
      @curr_speed = @curr_speed * curve_factor
    EndIf
    
    ' PID calculation
    P = E * current_kp
    
    ' Integral with anti-windup and region limiting
    If Math.Abs(E) < 10 Then
      @integral = @integral + (E * @ki)
      If @integral > 25 Then
        @integral = 25
      ElseIf @integral < -25 Then
        @integral = -25
      EndIf
    EndIf
    I = @integral
    
    ' Derivative with filtering
    D = (E - @OldE) * current_kd
    @OldE = E
    
    ' Final control value
    A = P + I + D
    
    ' Apply control
    If remaining > 5 Then
      ' Normal driving
      MotorB.StartPower((@curr_speed + A) * @kB)
      MotorC.StartPower((@curr_speed - A) * @kC)
    Else
      ' Precision approach near target
      precision_factor = remaining / 5
      precision_speed = Math.Max(10, @curr_speed * precision_factor)
      
      MotorB.StartPower((precision_speed + A) * @kB)
      MotorC.StartPower((precision_speed - A) * @kC)
    EndIf
    
    ' Check for completion
    If current_distance >= DestEnc Then
      ' Precision stop
      Stop()
      exit = 1
    EndIf
    
    ' Real-time performance optimization
    current_time = Time.Get2()
    time_delta = current_time - last_time
    
    If time_delta > 50 Then
      ' Calculate actual travel speed
      distance_delta = current_distance - last_distance
      travel_rate = distance_delta / time_delta
      
      ' Use this data to optimize parameters if needed
      ' (Advanced optimization could be implemented here)
      
      ' Update tracking variables
      last_time = current_time
      last_distance = current_distance
    EndIf
    
    Program.Delay(3)  ' Shorter delay for faster response
  EndWhile
EndFunction



Function LineEncoderASAN(in number DestEnc)
  Check()
  ' Remember initial encoder values
  StartB = MotorB.GetTacho()
  StartC = MotorC.GetTacho()
  @OldE = 0
  @integral = 0  ' Reset integral component
  exit = 0
  
  While exit = 0
    reg()
    
    ' Calculate relative distance traveled
    encB = Math.abs(MotorB.GetTacho() - StartB)
    encC = Math.abs(MotorC.GetTacho() - StartC)
    avg = (encB + encC)/2
    
    ' Stop at specified distance
    If avg > DestEnc Then
      exit = 1
      
    EndIf
  EndWhile
EndFunction


Function CheckVoltage()
  
  if EV3.BatteryVoltage < 8.09 Then
    Speaker.Tone(100, 3000, 100000000)
    stop()
    Program.Delay(1000000000)
    stop()
    
  ElseIf EV3.BatteryVoltage > 8.29 Then
    Motor.MoveSync("BC", 100, 100, 100000000, "TRUE")
    stop()
  EndIf
EndFunction
