# Case-study оптимизации

## Актуальная проблема
В нашем проекте возникла серьёзная проблема.

Необходимо было обработать файл с данными, чуть больше ста мегабайт.

У нас уже была программа на `ruby`, которая умела делать нужную обработку.

Она успешно работала на файлах размером пару мегабайт, но для большого файла она работала слишком долго, и не было понятно, закончит ли она вообще работу за какое-то разумное время.

Я решил исправить эту проблему, оптимизировав эту программу.

## Формирование метрики
Для того, чтобы понимать, дают ли мои изменения положительный эффект на быстродействие программы я придумал использовать такую метрику: Время выполнения программы + Расход памяти

## Гарантия корректности работы оптимизированной программы
Программа поставлялась с тестом. Выполнение этого теста позволяет не допустить изменения логики программы при оптимизации.

## Feedback-Loop
Для того, чтобы иметь возможность быстро проверять гипотезы я выстроил эффективный `feedback-loop`, который позволил мне получать обратную связь по эффективности сделанных изменений за 145.5 c

Вот как я построил `feedback_loop`: Для определения первых точек роста, я использовал файл со 10_000 строками данных. По мере улучшения метрики, увеличивал кол-во строк до максимума.

## Вникаем в детали системы, чтобы найти 20% точек роста
Для того, чтобы найти "точки роста" для оптимизации я воспользовался :
 - Выводом объём памяти RAM, выделенной процессу в настоящее время
 - memory_profiler
 - Отчётами исполюзуя gc_tracer
 - Stackprof ObjectAllocations and Flamegraph
 - Massif Visualizer исполюзуя valgrind
 - QCachegrind memory profiling (данный инструмент внёс 90% КПД в процесс оптимизации)

Вот какие проблемы удалось найти и решить

### Ваша находка №1
file_lines.each do |line|
    cols = line.split(',')
    users << parse_user(cols) if cols[0] == 'user'
    sessions << parse_session(cols) if cols[0] == 'session'
end

Используем "<<" и передаём переменную cols
Расход памяти уменьшился в 4 раза.

### Ваша находка №2
Время выполнения программы 10_000 ~ 10c, при 20_000 ~ 40c
Убираем перебор огромного массива sessions для создания User.new

file_lines.each do |line|
  ***
  if cols[0] == 'session'
    sessions_browsers << cols[3]
    sessions_by_id[cols[1]] ||= {}
    sessions_by_id[cols[1]]['sessions'] ||= []
    sessions_by_id[cols[1]]['sessions'] << parse_session(cols)
  end
  ***
end

users.each do |user|
  ***
  user_sessions = sessions_by_id[user['id']]['sessions']
  ***
end

+ модифицируем код под данную логику

Время выполнения программы 20_000 ~ 0,5c
Расход памяти уменьшился в 100 раз.

### Ваша находка №3
Время выполнения программы при 500_000 ~ 34.57с

Метод collect_stats_from_users вызывается 7 раз и внутри себя прогоняет список Users каждый раз.
+ Некоторые методы формирования хеша съедали память
Меняем на:

users_objects.each do |user|
  array_time = user.sessions.map { |s| s['time'].to_i }
  array_browser = user.sessions.map { |s| s['browser'] }
  user_key = "#{user.attributes['first_name']}" + ' ' + "#{user.attributes['last_name']}"
  report['usersStats'][user_key] ||= {}
  report['usersStats'][user_key]['sessionsCount'] = user.sessions.count
  report['usersStats'][user_key]['totalTime'] = array_time.sum.to_s + ' min.'
  report['usersStats'][user_key]['longestSession'] = array_time.max.to_s + ' min.'
  report['usersStats'][user_key]['browsers'] = array_browser.map { |b| b.upcase}.sort.join(', ')
  report['usersStats'][user_key]['usedIE'] = array_browser.map { |b| b.upcase =~ /INTERNET EXPLORER/ }.include? 0
  report['usersStats'][user_key]['alwaysUsedChrome'] = user.sessions.map{|s| s['browser']}.all? { |b| b.upcase =~ /CHROME/ }
  report['usersStats'][user_key]['dates'] = user.sessions.map{|s| s['date'] }.sort.reverse
end

Время выполнения программы при 500_000 ~ 17с
Расход памяти уменьшился в 3 раза.

## Результаты
В результате проделанной оптимизации наконец удалось обработать файл с данными.
Удалось улучшить метрику системы с не работающей программы, до программы, которая выполняется за 145с и расходует ~ 2,5 гб памяти

## Защита от регресса производительности
Для защиты от потери достигнутого прогресса при дальнейших изменениях программы все методы перебора разбил на маленькие части используя "each_slice"