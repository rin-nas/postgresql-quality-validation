# TODO Валидация качества схемы БД

1. Рефакторинг
   1. ✅ Вынести валидатор в отдельную схему `db_validation`, а функцию переименовать в `schema_validate()`. 
   1. ✅ Конфигурацию хранить в отдельной служебной таблице `db_validation.schema_validate_config`.
   1. Распилить валидатор на отдельные view / функции.
1. Архитектурные доработки (нужно добавить настройки в конфиг)
   1. 🚨 Добавить возможность валидации только добавленных, изменённых и удалённых объектов БД (для начала только таблицы и колонки) для одной транзакции.
      Для этого перед миграцией нужно запускать функцию `select db_validation.schema_validate_prepare()`,
      которая будет сохранять список всех существующих объектов БД во временную таблицу (таблица автоматически удалится в конце транзакции).
   1. 🚨 Добавить возможность возвращать список всех проблем в виде таблицы или ошибку. Пустая таблица означает, что всё ок.
   1. Добавить автотесты для каждого правила, для этого cделать тестовую схему `db_validation_test`.
   1. Добавить в таблицу `schema_validate_config` колонки (новые опции):   
      ```sql
      views_ignore_regexp  text check ( db_validation.is_regexp(views_ignore_regexp) ),
      views_ignore         regclass[],
      columns_ignore       ?
      --columns_ignore_regexp --не нужна, а то сложно
      ```
      Добавить обработку этих опций в `schema_validate()` в каждую проверку. 
      Тип колонки `columns_ignore` должен быть [составным типом](https://postgrespro.ru/docs/postgresql/14/rowtypes): `row(table_name regclass, column_name text)`.
      Для валидации существования колонки нужен домен, т.к. в составных типах `check()` не работает. 
1. Описания объектов БД (`COMMENT ON ...`)
   1. Добавить проверку наличия описаний для БД, схем, представлений (view), типов, функций, процедур, триггеров. В миграциях БД забывают это делать.
   1. Описание не должно быть пустым и не должно совпадать с названием объекта БД.
      Описание должно иметь осмысленный текст и начинаться на строчную (большую) букву. Проверять так: `{comment} != lower({comment})`
   1. Описания для всех колонок одной таблицы должны быть уникальными.
1. Названия объектов БД
   1. 🚨 Названия объектов БД должны содержать только английские буквы, цифры, дефис, тире: `select 'col_name5' ~ '^[a-z][a-z\d_\-]*[a-z\d]$';`.
      Нужно добавить в конфиг 2 регулярки, по которым проверять названия таблиц и колонок.
      Был случай, когда в названии колонки не заметили русскую букву "c".
      Названия объектов БД д.б. только в нижнем регистре (Postgres makes everything lower case unless you double quote it).
   1. 🚨 Названия колонок БД НЕ должны совпадать с названиями существующих типов в БД (это плохая практика). Название должно показывать суть.
      ```sql
      select table_name, column_name --, data_type, udt_name
      from information_schema.columns
      where true
          and table_schema not in ('pg_catalog', 'migration', 'test')
          and table_name !~ 'pg_'
          --and column_name = udt_name
          and column_name in (
              SELECT unnest(array_cat(
                         array_agg(typname),
                         array_agg(sql_name) filter (where sql_name is not null)
                     )) as types
              FROM pg_type
              LEFT JOIN format_type(oid, NULL::integer) as f(sql_name)
                     on sql_name !~ '[" ]' and sql_name != typname
              WHERE true
                  AND typname != 'name'
                  AND typtype = 'b'
                  AND typarray != 0
                  AND typcategory not in ('E', 'A')
          );
      ```
   1. Название колонки для первичного несоставного ключа должно соответствовать шаблону рег. выражения, например, заканчиваться на `id` или `guid` (сделать настраиваемым)
   1. Название колонки `guid` должно иметь тип `uuid` (сделать настраиваемым)
      Название колонки `ids` должно быть массивом (сделать настраиваемым)
   1. Название колонки с FK должно содержать часть названия из FK. Пример для `author integer REFERENCES test.forum__user (id) on delete set null` должно быть `author_id`
   1. Название таблицы или колонки не может начинаться на `id_`, оно может так заканчиваться (сделать настраиваемым)
   1. Названия таблиц типа reviewed_resume д.б. невалидными, первое слово д.б. существительным, а не глаголом
   1. Добавить проверку именования последовательностей по шаблону `{table}_{column}_seq` (сделать настраиваемым). Некоторые фреймворки закладываются на эти названия. Для получения названия последовательности правильно использовать функцию `pg_get_serial_sequence('{table}', '{column}')`
   1. Взять соглашения по именованию из [API](https://wiki.rabota.space/pages/viewpage.action?pageId=25789378), см. последний раздел
1. Названия элементов `enum`
   1. Внутри одного `enum` названия его элементов д.б. в едином стиле. Пример: `has_pk_uk` (snake case) или `has-pk-uk` (kebab case) или `hasPkUk` (camel case) или `HasPkUk` (pascal case).
   1. Для всей БД в `enum` названия их элементов д.б. в едином стиле.
1. Индексы
   1. ✅ Добавить проверку на наличие невалидных (битых) индексов. Сейчас для таких индексов возвращается ошибка "Отсутствует индекс для внешнего ключа".
   1. Добавить проверку отсутствия полных дубликатов индексов с разными названиями.
   1. Добавить проверку уникальных индексов, чтобы все колонки из индекса были с ограничением `NOT NULL`. Иначе ограничение уникальности не работает и нужно делать [по другому](https://github.com/rin-nas/postgresql-patterns-library/tree/master#%D0%BA%D0%B0%D0%BA-%D1%81%D0%B4%D0%B5%D0%BB%D0%B0%D1%82%D1%8C-%D1%81%D0%BE%D1%81%D1%82%D0%B0%D0%B2%D0%BD%D0%BE%D0%B9-%D1%83%D0%BD%D0%B8%D0%BA%D0%B0%D0%BB%D1%8C%D0%BD%D1%8B%D0%B9-%D0%B8%D0%BD%D0%B4%D0%B5%D0%BA%D1%81-%D0%B3%D0%B4%D0%B5-%D0%BE%D0%B4%D0%BD%D0%BE-%D0%B8%D0%B7-%D0%BF%D0%BE%D0%BB%D0%B5%D0%B9-%D0%BC%D0%BE%D0%B6%D0%B5%D1%82-%D0%B1%D1%8B%D1%82%D1%8C-null).
   1. Добавить параметр для игнорирования индексов `indexes_ignore_regexp`. Пример: `^pgcompact_index_\d+$`.
   1. При обнаружении избыточных индексов рекомендовать удалять индексы с названием по маске `(_ccnew$|^pgcompact_index_\d+$)`, а не индексы с другими названиями
   1. B-деревья подходят для индексирования только скалярных данных. Для массивов должен быть GIN индекс вместо btree.
   1. Добавить проверку на вероятно избыточный индекс, если для `field` есть `lower(field)`, `upper(field)`, `date(field)`. Рекомендовать удалить индекс на `field`!
   1. Обнаруживать (и рекомендовать удалять, а не выдавать ошибку) такие избыточные индексы:
      1. `CREATE UNIQUE INDEX ON paid_services.snapshot_package_limit USING btree (order_item_id, service_id, sort(uniq(zone_ids)))`
      1. `CREATE UNIQUE INDEX ON paid_services.snapshot_package_limit USING btree (order_item_id, service_id, zone_ids)`
1. Рекомендации. Кроме ошибок можно возвращать рекомендации:
   1. Для колонок типа `json[b]` рекомендовать делать валидацию для верхнего уровня.
   1. Валидатор даёт неправильную рекомендацию по удалению избыточного индекса. Нужно смотреть на связанные ограничения?
      ```sql
      ERROR:  Таблица public.preset__currency уже имеет индекс CREATE UNIQUE INDEX preset__currency_id_uindex ON public.preset__currency USING btree (id)
      Удалите избыточный индекс CREATE UNIQUE INDEX preset__currency_pk ON public.preset__currency USING btree (id)
      CONTEXT:  PL/pgSQL function db_validation.schema_validate() line 109 at RAISE 
      ```
1. Типы колонок
   1. 🚨 Вместо устаревшего `CHAR(n) / VARCHAR(n)` нужно использовать `TEXT` с ограничением, например: `check(length(col) between 0 and 100)`
   1. 🚨 Для текстовых колонок `TEXT` и `VARCHAR` без ограничения длины и с отсутствием ограничения `check(...)` необходимо делать ограничение с валидацией `check(length(col) between X and Y)`
   1. 🚨 Вместо устаревшего `TIMESTAMP` (WITHOUT TIME ZONE) нужно использовать `TIMESTAMPTZ` (TIMESTAMP WITH TIME ZONE)
   1. Вместо проблемного `MONEY` нужно использовать `NUMERIC` and store currency in another column
   1. Вместо устаревшего `SERIAL` нужно использовать `[BIG]INT GENERATED`
   1. Вместо `JSON` рекомендовать использовать `JSONB`
   1. В текстовое поле нельзя записать `null` или пустую строку на выбор (когда нет ни одного ограничения типа `check` на колонку), д.б. только 1 способ. 
      Пример проблемной миграции: `alter table {table} add {column} varchar(10);`
   1. Для колонки `updated_at` (название задать в конфиге) должен быть триггер, который устанавливает значение `now()` при создании или обновлении записи
6. Взять идеи из 
   1. [DBA: находим бесполезные индексы](https://habr.com/ru/company/tensor/blog/488104/)
   1. https://github.com/ankane/strong_migrations
   1. https://github.com/kristiandupont/schemalint/tree/master/src/rules
   1. https://github.com/IMRSVDataLabs/imrsv-schema-linter
   1. https://gitlab.com/depesz/pgWikiDont/-/tree/master/
   1. https://www.google.com/search?q=postgresq+schema+linter - ещё ссылки здесь
1. Объекты БД в одной схеме должны принадлежать одному владельцу (опциональная проверка), см. [`pg_object_owner.sql`](../../views/pg_object_owner.sql)
1. Добавить проверку для запрещения возможности вставки в таблицу (название не заканчивается на `_log` или `_history`) дубликатов строк, если есть PK на колонку id и нет UK без id. В этой проверке не участвуют колонка с PK, колонки с датой, датой-временем.
1. Добавить проверку при наличии расширения https://github.com/okbob/plpgsql_check/
1. Добавить проверку отсутствия триггерных функций, которые нигде не используются. Пример: удалили триггер, который вызывал триггерную функцию. Теперь функция нигде не используется.
1. Добавить проверку отсутствия дубликатов ограничений таблицы (`has_not_duplicate_constraint`). 
   Причины появления таких ошибок:
   * повторные накаты миграции БД, если в коде отсутствует название ограничения; 
   * копирование кода ограничения из предыдущей колонки, где в `check(...)` забыли изменить название колонки. 
   ```sql
   with s as (
      SELECT s.conrelid                                                             as table_oid,
             array_length(array_agg(pg_get_constraintdef(s.oid, true)), 1)          as def_count,
             array_length(array_agg(distinct pg_get_constraintdef(s.oid, true)), 1) as def_uniq_count,
             array_agg(pg_get_constraintdef(s.oid, true))                           as def
      FROM pg_constraint as s
      WHERE connamespace::regnamespace not in ('pg_catalog', 'information_schema')
        AND s.conrelid != 0
      GROUP BY s.conrelid
   )
   select s.table_oid::regclass, t.*
   from s
   cross join lateral (
       select u.value,
              count(*) as duplicate_count
       from unnest(s.def) as u(value)
       group by u.value
       having count(*) > 1
   ) as t
   where def_count != def_uniq_count;
   ```
1. Добавить проверку: если для последовательностей процент достижения своего максимального значения > N%, то рекоментовать сменить `int` на `bigint`.
   Ссылки по теме: 
   [1](https://stackoverflow.com/questions/54795701/migrating-int-to-bigint-in-postgressql-without-any-downtime),
   [2](http://zemanta.github.io/2021/08/25/column-migration-from-int-to-bigint-in-postgresql/),
   [3](https://engineering.silverfin.com/pg-zero-downtime-bigint-migration/).
   ```sql
   select schemaname as schema,
          sequencename as sequence_name,
          data_type,
          used_percent
   from pg_sequences
   cross join round(last_value * 100.0 / max_value, 2) as used_percent
   where last_value is not null /*null means access denied*/ and used_percent > 33
   order by used_percent desc;
   ```
1. Ограничение с условиями между разными колонками на уровне строки таблицы смотрится понятнее. Пример:
   ```sql
   CREATE TABLE test.test1
   (
       day_from int check(day_from >= 0 and coalesce(day_from, day_to) is not null),
       day_to   int check(day_to >= 0 AND day_from <= day_to)
   );
   --vs
   CREATE TABLE test.test1
   (
       day_from int check(day_from >= 0),
       day_to   int check(day_to >= 0), --тут запятая!
       check (coalesce(day_from, day_to) is not null and day_from <= day_to)
   );
   -- TEST
   insert into test.test1 values (null, null); --error
   insert into test.test1 values (1, null); --ok
   insert into test.test1 values (null, 1); --ok
   insert into test.test1 values (1, 2); --ok
   insert into test.test1 values (2, 1); --error
   ```
