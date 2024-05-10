#pg_repl


В документе описаны 4 роли:
1) pg_init_master - Настройка Master
2) pg_init_standby - Настройка StandBy
3) pg_promote_standby - Завершение работы в режиме StandBy и начало работы в режиме чтения/записи.
4) pg_change_role_master - Настройка Master в режим StandBy

Для тестирования ролей необходимо иметь 2 сервера на базе CentOS 7.
Сервера, должны быть запущены.
В данном случае авторизация происходит по паре логин/пароль

Структура инвентори `hosts.yml`:
```yaml
db_servers:
  hosts:
    server01:
      ansible_host: <ip>
      ansible_user: "{{ server01_user}}"
      ansible_password: "{{ server01_pass}}"
    server02:
      ansible_host: <ip>
      ansible_user: "{{ server02_user}}"
      ansible_password: "{{ server02_pass}}"

master:
  hosts:
    server01:


standby:
  hosts:
    server02:
```

Креды лежат в зашифрованном файле ansible-vault `hosts_info.yml`
Содержимое которого:
```yml
---
server01_user: root
server01_pass: password

server02_user: root
server02_pass: password
```

Параметры который необходимо задать для работы серверов и скриптов описаны в файле **`group_vars/all`**

В корне папки проекта также лежат 3 плейбука, которые выполняют поставленные задачи:
+ `side.yml` - Настраивает Master(роль pg_init_master) , затем настраивает StandBy(роль pg_init_standby). Запуск роли pg_init_standby выполняется после pg_init_master
+ `side_promote.yml` - запускает роль pg_promote_standby для хоста `server02`
+ `side_change_role_master.yml` - запускает роль pg_change_role_master для хоста `server01`

Пример запуска
```bash
ansible-playbook -i hosts.yml side.yml --ask-vault-pass
ansible-playbook -i hosts.yml side_promote.yml --ask-vault-pass
ansible-playbook -i hosts.yml side_change_role_master.yml --ask-vault-pass
```


После настройки параметров и инвентори можно запустить плейбуки:

Выполняем конфигурацию:
```bash
ansible-playbook -i hosts.yml side.yml --ask-vault-pass
```
логинимся на мастер
```bash
su - postgres
psql -c "CREATE TABLE test_table (id INT, name TEXT);"
psql -c "INSERT INTO test_table (id, name) VALUES (1, 'test');"
psql -c "SELECT * FROM test_table;"
```
логинимся на стендбай
```bash
su - postgres
psql -c "SELECT * FROM test_table;"
```
Результат совпадает.
Также можно проверить использование слота репликации:
```bash
# Выполняем на Master
psql -c "\x" -c "SELECT * FROM pg_stat_replication;"
```

```bash
# Выполняем на StandBy
psql -c "\x" -c "SELECT * FROM pg_stat_wal_receiver;"
```

Выключаем StandBy. 

```bash
# Выполняем на Master
psql -c "INSERT INTO test_table (id, name) VALUES (2, 'test2');
```

Ждем 5 минут. Включаем StandBy. Логинимся. Проверяем
```bash
# Выполняем на StandBy
su - postgres
psql -c "SELECT * FROM test_table;"
```

Выключаем Master. Ждем 5 минут. Включаем. 
```bash
# Выполняем на Master
su - postgres
psql -c "INSERT INTO test_table (id, name) VALUES (3, 'test3');
```

```bash
# Выполняем на StandBy
su - postgres
psql -c "SELECT * FROM test_table;"
```


Выполняем с управляющего хоста
```bash
# Переводим StandBy в Master
ansible-playbook -i hosts.yml side_promote.yml --ask-vault-pass
```

```bash
# Выполняем на StandBy
psql -c "INSERT INTO test_table (id, name) VALUES (4, 'test4');"
psql -c "SELECT * FROM test_table;"
```

Выполняем с управляющего хоста
```bash
ansible-playbook -i hosts.yml side_change_role_master.yml --ask-vault-pass
```

```bash
# Выполняем на StandBy
psql -c "INSERT INTO test_table (id, name) VALUES (5, 'test5');"
psql -c "SELECT * FROM test_table;"
```

```bash
# Выполняем на Master
su - postgres
psql -c "SELECT * FROM test_table;"
```

Выключаем Master
```bash
# Выполняем на StandBy
psql -c "INSERT INTO test_table (id, name) VALUES (6, 'test6');"
psql -c "SELECT * FROM test_table;"
```

Ждем 5 минут. Включаем Master.

```bash
# Выполняем на Master
su - postgres
psql -c "SELECT * FROM test_table;"
```

Выключаем StandBy. Ждем 5 минут. Включаем
```bash
# Выполняем на StandBy
su - postgres
psql -c "INSERT INTO test_table (id, name) VALUES (7, 'test7');"
psql -c "SELECT * FROM test_table;"
```

```bash
# Выполняем на Master
su - postgres
psql -c "SELECT * FROM test_table;"
```


