# Лабораторная работа 1
## Уровни изоляции транзакций

1. Создаем новый проект в gcloud, создаем instance. Подключаемся через CLI

2. gcloud compute ssh postgresql20221118
сгенерировался ssh key, добавился в метаданные gcloud
подтвердила и инициировалось соединение по putty


3. сгенерирую еще один ключ с паролем
cmd --> ssh-keygen -t rsa

> Your public key has been saved in C:\Users\annak/.ssh/id_rsa.pub.The key fingerprint is:SHA256:nlCOczAUq5ov/n/sAra8h+VpHlrH1icaV+XJ+89l+eA annak@CheaterThe key's randomart image is:


4. Добавляем ключ в метаданные инстанса qcloud
Metadata -> SSh KEYS -> EGUIVALENT REST -> add rsa key from C:\Users\annak\.ssh id_rsa PUB


5. Подключаемся по ssh 
cmd -> ssh annak@34.27.91.234 --> вводим пароль

6. Устанавливаем кластер postgresql 12
sudo apt-get -y install postgresql

7. Кластер стартовал

annak@postgresql20221118:~$ pg_lsclusters
>Ver Cluster Port Status Owner    Data directory              Log file
12  main    5432 online postgres /var/lib/postgresql/12/main /var/log/postgresql/postgresql-12-main.log


8. Заходими в psql пользователем postgres
>sudo -u postgres psql

9. Создадим бд edu_pg

>CREATE DATABASE edu_pg;

10. Подключаемся к бд edu_pg
 
 >\с edu_pg

11. Создаем таблицу

>create table persons(id serial, first_name text, second_name text);

12. Инсертим данные в таблицу persons

>insert into persons(first_name, second_name) values('ivan', 'ivanov');\
insert into persons(first_name, second_name) values('petr', 'petrov');\
commit;

>INSERT 0 1\
INSERT 0 1\
COMMIT


13. Смотрим текущий уровень изоляции транзакций:

>show transaction isolation level;\
transaction_isolation\
>----------------------\
read committed\
(1 row)


14. Подключаемся к бд по ssh второй сессией. Уровень изоляции транзакций не меняем. 
Первой сессией инсертим данные в таблицу persons

>insert into persons(first_name, second_name) values('sergey', 'sergeev');\
INSERT 0 1

15. Во второй сессии делаем выборку

>select * from persons
и видим результат\
\
> id | first_name | second_name\
----+------------+-------------\
  1 | ivan       | ivanov\
  2 | petr       | petrov\
  3 | sergey     | sergeev\
(3 rows)

Потому что: 
* каждая комманда для pgsql оборачивается в  транзакцию, стоит AUTOCOMMIT (по умолчанию) и
мы и видим (во второй сессии) зафиксированные изменения (первой сессией)

>\echo :AUTOCOMMIT\
on


* во-вторых мы видими анамалию read committed что соответствует нашему уровню изоляции транзакции.


16. Сделав commit; в двух сессиях мы увидим все тоже самое. Потому что AUTOCMMIT ON

17. Поставим в двух сессиях 

 > \set AUTOCOMMIT OFF

18. В первой сессии делаем:
>BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;\
insert into persons(first_name, second_name) values('sveta', 'svetova');

19. Во второй сессии делаем 
>BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;\
select * from persons;\
\
 >id | first_name | second_name\
----+------------+-------------\
  1 | ivan       | ivanov\
  2 | petr       | petrov\
  3 | sergey     | sergeev\
(3 rows)

Видим 3 строки, потому что первая транзакция не зафиксирована, а pg не делает грязные чтения и уровень изоляции транзакции у нас REPEATABLE READ.

20. в первой сессии делаем commit;

Во второй сессии делаем опять выборку 
>select * from persons;\
 \
id | first_name | second_name\
----+------------+-------------\
  1 | ivan       | ivanov\
  2 | petr       | petrov\
  3 | sergey     | sergeev\
(3 rows)

Не видим четвертую строку потому что в рамках данной транзакции (REPEATABLE READ) 
мы и не должны ее видеть, так работает субд ;) 

21. Во второй сессии делаем commit; и снова выборку 

>select * from persons;\
\
 id | first_name | second_name\
----+------------+-------------\
  1 | ivan       | ivanov\
  2 | petr       | petrov\
  3 | sergey     | sergeev\
  4 | sveta      | svetova\
(4 rows)

Теперь видим строку 4, т.к выборка запущена в рамках новой транзакции.


