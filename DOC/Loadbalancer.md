# Балансировщик нагрузки
Балансировщик работает на nginx. Кстати, спасибо его разработчику Игорю Сысоеву за отличный продукт.

## Dockerfile
```
FROM alpine:latest
RUN apk add --update-cache --no-progress --no-cache nginx
#We well ensure that errors and access go to standard error/ouput
RUN ln -sf /dev/stdout /var/log/nginx/access.log
RUN ln -sf /dev/stderr /var/log/nginx/error.log
ENV APPLICATION_PORT 3000
COPY nginx.conf /etc/nginx/nginx.conf
COPY entrypoint.sh /entrypoint.sh
EXPOSE 80
ENTRYPOINT ["/entrypoint.sh"]
CMD nginx -g 'pid /run/nginx.pid; daemon off;'
```

Рассмотрим процесс создания образа контейнера.
1. В первой строке объявляется исходный обрахдля сборки. За основу взят легковесный Alpine Linux. Только в учебных целях сборка ведётся издалека. В Prodcution за основу можно взять готовый образ Nginx с DockerHub. 
```
FROM alpine:latest
```
2. В следующей строке используем менеджер пакетов ```apk``` для установки nginx. 
- Опция ```--update-cache``` обновляет кэш репозитория. Нам нужны самые свежие версии пакетов.
- Опция ```--no-progress``` отключает отображение полосы прогресса. В контейнере излишества не нужны.
- Опция ```--no-cache``` отключает создание кэша при установке пакетов. В контейнере он не нужен.
```
RUN apk add --update-cache --no-progress --no-cache nginx
```
3. В следующих двух строках выводим журнал доступа к серверу и журнал ошибок напрямую в стандартные поток вывода и поток ошибок.
```
RUN ln -sf /dev/stdout /var/log/nginx/access.log
RUN ln -sf /dev/stderr /var/log/nginx/error.log
```
4. В следующей строке объявляем переменную среды, через которую во время сборки балансировщуку будет передан номер порта, на котором приложение слушает запросы.
```
ENV APPLICATION_PORT 3000
```
5. В следующих двух командах мы копируем в контейнер файл с настройками ```nginx.conf``` и скрипт для запуска ```entrypoint.sh```
```
COPY nginx.conf /etc/nginx/nginx.conf
COPY entrypoint.sh /entrypoint.sh
```
6. Следующая команда публикует внутренний порт 80, на котором балансировщик ожидает запросы. При таком синтаксисе внутренний порт будет связан со случайным внешним портом из диапазона, который начинается примерно с 32780. (Точное число на нашёл в документации.) Позже, при сборке через файл Docker-Compose.yaml, этот порт будет соединен с внешним портом 8080.
```
EXPOSE 80
```
7. Предпоследняя команда запускает настройку балансировщика. Инструкции хранятся в файле entrypoint.sh.
```
ENTRYPOINT ["/entrypoint.sh"]
```
8. Завершающая команда фактически запускает сам балансировщик нагрузки.
- Опция ```-g``` передаёт команды глобальной настройки.
- Команда ```pid /run/nginx.pid``` устанавливает идентификатор главного процесса nginx.
- Команда ```daemon off``` переводит Nginx на передний план (foreground). Это типично при запуске в контейнере.
```
CMD nginx -g 'pid /run/nginx.pid; daemon off;'
```

## Варианты оптимизации образа
1. Можно объединить все команды RUN в одну строку. Тогда получится
```
RUN apk add --update-cache --no-progress --no-cache nginx && \
ln -sf /dev/stdout /var/log/nginx/access.log && \
ln -sf /dev/stderr /var/log/nginx/error.log
```
2. Начиная с версии Docker 1.13 можно использовать опцию --squash, которая сама объединит все слои в один. Тогда собирать образ нужно так
```
docker build --squash .
```
