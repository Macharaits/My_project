# ETL задача
## Краткое описание
Создание DAG в Airflow, который наполняет таблицу данными за вчерашний день. Данные выгружаются из Clickhouse по двум таблицам: данные по ленте новостей и данные по сообщениям. В таблице feed_actions для каждого ползователя посчитаем число просмотров и лайков контента. В таблице message_actions для каждого юзера считаем, сколько он получает и отсылает сообщений, скольким людям он пишет, сколько людей пишут ему. В результате должны получить новую таблицу со всем подсчитанными значениями и загрузить ее обратно в Clickhouse.

Структура итоговой таблицы:

Дата - event_date
Название среза - dimension
Значение среза - dimension_value
Число просмотров - views
Число лайков - likes
Число полученных сообщений - messages_received
Число отправленных сообщений - messages_sent
От скольких пользователей получили сообщения - users_received
Скольким пользователям отправили сообщение - users_sent
Срез - это os, gender и age

Стэк:
* Стэк:
* JupiterHub
* Clickhouse
* Python
* SQL
* Airflow

# Основной код
## Настройка связи
Настроили подключение к Clickhouse и задали параметры для выполнения дага. После чего с помощью SQL-запроса получим нужные данные, которые будут ежедневно выгружаться в таблицу.

```python 
# Дефолтные параметры, которые прокидываются в таски
default_args = {
 'owner': 'm.rajts',
 'depends_on_past': False,
 'retries': 2,
 'retry_delay': timedelta(minutes=5),
 'start_date': datetime(2023, 8, 9), #yesterday()
}
# Интервал запуска DAG
schedule_interval = '0 23 * * *'
```
## Создание тасков

Всего создали 8 задач. Двое из них выгружают данные по действиям новостной ленты и по данным с сообщениями.
```python 
@task()
 def extract_feed_actions():
 #В feed_actions для каждого юзера посчитаем число просмотров и лайков контента.
 query = """SELECT user_id, 
                   toDate(time) event_date,
                   countIf(action = 'like') likes,
                   countIf(action = 'view') as views,
                   os,
                   gender, 
                   age
            FROM simulator_20230620.feed_actions
            WHERE toDate(time) = yesterday()
            GROUP BY toDate(time), user_id, gender, age, os
            format TSVWithNames"""
 df_cube_feed = ch_get_df(query=query)
 return df_cube_feed

@task()
  def extract_message_actions():
       #В message_actions для каждого юзера считаем, сколько он получает и отсылает сообщений, скольким людям он пишет, сколько людей пишут ему.
      query =  """SELECT user_id, 
                         event_date,
                         messages_received, 
                         messages_sent, 
                         users_received, 
                         users_sent,
                         os,
                         gender,
                         age
                  FROM (SELECT user_id, 
                               toDate(time) as event_date,
                               COUNT() as messages_sent,
                               COUNT(DISTINCT reciever_id) as users_sent,
                               os,
                               gender,
                               age
                        FROM simulator_20230620.message_actions
                        WHERE toDate(time) = yesterday()
                        GROUP BY toDate(time), user_id, gender, age, os) t1
                  FULL JOIN 
                           (SELECT reciever_id as user_id, 
                                   toDate(time) as event_date,
                                   COUNT() as messages_received, 
                                   COUNT(DISTINCT user_id) as users_received
                            FROM simulator_20230620.message_actions
                            WHERE toDate(time) = yesterday()
                            GROUP BY reciever_id, toDate(time)) t2
                  USING(user_id, event_date)
                  format TSVWithNames"""
 df_cube_message = ch_get_df(query=query)
 return df_cube_message
```

Следующий task после выгрузки данных объединяет таблицы. Еще три task-а отвечают за получение срезов по полу, возрасту и операционной системе, в последствии эти данные объединяются. И финальный task выгружает итоговый результат в одну таблицу.

```python 
@task
 def load(df_all):
      #записывает данные
      query_t = """CREATE TABLE IF NOT EXISTS test.rajts
                  (dimension String,
                   dimension_value String,
                   event_date Date,
                   likes UInt64,
                   views UInt64,
                   messages_received UInt64,
                   messages_sent UInt64,
                   users_received UInt64,
                   users_sent UInt64)
                   ENGINE = MergeTree()
                   ORDER BY event_date"""
      ph.execute(query = query_t, connection = connection)
      #Заливаю табличку в базу тест
      ph.to_clickhouse(df=df_all, table='rajts', index = False, connection = connection)
      context = get_current_context()
      ds = context['ds']
      print(f'Actions for {ds}')
      print(df_all.to_csv(index=False, sep='\t'))
  
 df_cube_feed = extract_feed_actions()
 df_cube_message = extract_message_actions()
 df_cube_merge = transform_merge(df_cube_feed, df_cube_message)
  
 df_cube_gender = transform_gender(df_cube_merge)
 df_cube_os = transform_os(df_cube_merge)
 df_cube_age = transform_age(df_cube_merge)
  
 df_all = transform_union(df_cube_os, df_cube_gender, df_cube_age)
 load(df_all)
  
dag_rajts = dag_rajts()
```
## Ежедневная выгрузка

В airflow настроена ежедневная выгрузка. Граф выглядит следующим образом.
![2023-09-15_17-43-45](https://github.com/Macharaits/My_project/assets/117433497/e2ec1a8f-126c-4237-beae-44d46a35a84f)
![2023-09-15_17-45-19](https://github.com/Macharaits/My_project/assets/117433497/15b14b96-e657-4320-856f-564ce61b1740)



