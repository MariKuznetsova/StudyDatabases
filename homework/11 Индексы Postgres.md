### Подготовка тестового фрагмента
Для выполнения ДЗ по индексам потребовалось увеличить тестовый фрагмент базы с 30-130 записей до 50-100 тыс. Дополнялись таблицы Work (произведение) и Book (книга издание). Данные генерировались случайные, с учетом целостности данных, ограничений и смысла. Названия книг формировались по алгоритму случайное прилагательное (1 из 100) + случайное существительное (1 из 100)

Пример использовавшегося запроса для генерации данных
```SQL
-- Вставляем записи в таблицу book (печатная книга)
INSERT INTO catalog.book(isbn,
                         bbk,
                         udk,
                         booktitle,
                         publisherid,
                         publisherseriesid,
                         cover,
                         pagecounts,
                         illustrations,
                         size,
                         agelimit,
                         weight,
                         status,
                         publishyear,
                         bookprice)                      
WITH arr AS (
    SELECT '{"4Пол", "4Гем", "2Рос=Рус", "7Кан", "7Сое"}'::varchar[] bbk_arr,
           '{"Суперобложка","Мягкая обложка","Твердая обложка"}'::catalog.covertype[] cover_arr,
           '{"готовится к печати","печать","есть на складе","нет в продаже"}'::catalog.statusbook[] status_arr,
           '{"любопытный","положительный","организационный","коренной","стальной","красный","магнитный","петербургский","плохой","различный","пожилой","частный","здешний","полноценный","металлургический","бывший","добровольный","нехороший","финансовый","чуждый","иркутский","криминальный","социалистический","людской","казенный","роскошный","восточный","ответный","суровый","московский","уважаемый","неизбежный","комплексный","излишний","пищевой","невидимый","покойный","неясный","прекрасный","любой","боковой","злой","детальный","острый","пестрый","видный","новый","рыжий","экологический","оригинальный","практический","отрицательный","немалый","ближний","северный","народный","драматический","комсомольский","железнодорожный","программный","властный","учебный","дневной","красивый","виноватый","прочий","самостоятельный","возможный","печатный","чувствительный","внезапный","перспективный","колючий","дружеский","чудесный","бешеный","иностранный","важный","реальный","целесообразный","головной","свежий","родной","чрезвычайный","зенитный","оранжевый","линейный","сомнительный","минимальный","социальный","довольный","максимальный","необыкновенный","поэтический","похожий","болезненный","научный","дальнейший","последовательный","нужный"}'::varchar[] word1_arr,
           '{"месяц","газ","рубеж","блок","ноябрь","роман","экран","коллектив","стол","перевод","кандидат","день","путь","спор","суть","редактор","гражданин","участник","август","гость","штаб","механизм","памятник","фонд","мальчишка","суд","союз","препарат","инженер","взрыв","май","стакан","коллега","храм","штат","восток","тип","куст","январь","ученик","папа","выпуск","прокурор","сюжет","театр","рассказ","представитель","металл","доллар","царь","источник","генерал","урок","рисунок","москвич","объект","билет","музей","грех","художник","декабрь","режим","страх","вагон","комитет","образ","праздник","врач","язык","волос","профессор","коммунист","знак","род","материал","результат","дядя","шум","лоб","ящик","университет","журнал","полковник","город","русский","бутерброд","блокнот","полюс","остров","архипелаг","конверт","магазин","бордюр","философ","эфир","рок","бурундук","дождь","снег","костюм"}'::varchar[] word2_arr
)
SELECT '978-5-03-'||lpad((round(random()*(190000-80000)+80000))::varchar, 6, '0')||'-'||(round(random()*(8-0)+0)) AS isbn,
       '84('||arr.bbk_arr[round(random()*(array_length(arr.bbk_arr,1)-1)+1)]||')' AS bbk,
       '' AS udk,
       arr.word1_arr[round(random()*(array_length(arr.word1_arr,1)-1)+1)]||' '||
       arr.word2_arr[round(random()*(array_length(arr.word2_arr,1)-1)+1)] AS booktitle,
       1 AS publisherid,
       (round(random()*(6-1)+1)) AS publisherseriesid,
       arr.cover_arr[round(random()*(array_length(arr.cover_arr,1)-1)+1)] AS cover,
       (round(random()*(1000-100)+100)) AS pagecounts,
       false as illustrations,
       '84x108 1/32' AS size,
       '16+' AS agelimit,
       (round(random()*(700-200)+200)) AS weight,
       arr.status_arr[round(random()*(array_length(arr.status_arr,1)-1)+1)] AS status,
       (round(random()*(2022-2000)+2000)) AS publishyear,
       (round(random()*(1200-130)+130)) AS bookprice
FROM generate_series(1,50000,1) AS w(id1),
     arr
ON conflict do nothing;
```

### 1. Создать индекс к какой-либо из таблиц вашей БД
```SQL
-- Создаем индекс для таблицы book (печатная книга) по полю цена
CREATE INDEX if not exists Book_BookPrice_idx ON catalog.Book (BookPrice);
```
### 2. Прислать текстом результат команды explain, в которой используется данный индекс
```SQL
-- Выбираем книги с ценой от 200 до 300 рублей
SELECT Book.ISBN,
       Book.BookTitle,
       Book.BookPrice
FROM catalog.Book
WHERE Book.BookPrice BETWEEN 200 AND 300;
```
**Результат команды Explain Analyze без индекса**  
```md
QUERY PLAN  
Seq Scan on book  (cost=0.00..2491.42 rows=4676 width=55) (actual time=0.026..32.348 rows=4670 loops=1)  
  Filter: ((bookprice >= '200'::numeric) AND (bookprice <= '300'::numeric))  
  Rows Removed by Filter: 45358  
Planning Time: 0.167 ms  
Execution Time: 32.572 ms  
```  

**Результат команды Explain Analyze с использованием индекса**. При выполнении запроса также используется битовая карта.  
```md
QUERY PLAN  
Bitmap Heap Scan on book  (cost=104.48..1915.99 rows=4701 width=55) (actual time=1.450..4.805 rows=4670 loops=1)  
  Recheck Cond: ((bookprice >= '200'::numeric) AND (bookprice <= '300'::numeric))  
  Heap Blocks: exact=1139  
  ->  Bitmap Index Scan on book_bookprice_idx  (cost=0.00..103.30 rows=4701 width=0) (actual time=1.229..1.229 rows=4670 loops=1)  
        Index Cond: ((bookprice >= '200'::numeric) AND (bookprice <= '300'::numeric))  
Planning Time: 0.223 ms  
Execution Time: 5.070 ms  
```

С индексом запрос отрабатывает в ~6 раз быстрее.

### 3. Реализовать индекс для полнотекстового поиска
Полнотекстовый поиск будем использовать для поиска по названию книги 
```SQL
-- Добавляем поле с раскладкой по словам
ALTER TABLE catalog.Book ADD Title_Lexeme tsvector;

-- Заполняем его на основе поля с названием книги
UPDATE catalog.Book SET Title_Lexeme = to_tsvector(BookTitle);

-- Создаем индекс GIN для полнотекстового поиска
CREATE INDEX if not exists Book_BookTitleLexeme_idx ON catalog.Book USING GIN (Title_Lexeme);

-- Поиск книг с волками в названии
SELECT Book.ISBN,
       Book.BookTitle,
       Book.Title_Lexeme
FROM catalog.Book
WHERE Book.Title_Lexeme @@ to_tsquery('волчий|волк');
```
**Результат запроса**  
![Select_booktitle_lexeme](https://github.com/MariKuznetsova/StudyDatabases/blob/main/homework/11.%20Postgres%20%D0%B8%D0%BD%D0%B4%D0%B5%D0%BA%D1%81%D1%8B/Select_booktitle_lexeme.PNG?raw=true)

**Результат команды Explain Analyze** - используется созданный GIN-индекс  
```md
QUERY PLAN  
Bitmap Heap Scan on book  (cost=25.98..727.69 rows=223 width=96) (actual time=0.082..0.087 rows=4 loops=1)  
  Recheck Cond: (title_lexeme @@ to_tsquery('волчий|волк'::text))  
  Heap Blocks: exact=2  
  ->  Bitmap Index Scan on book_booktitlelexeme_idx  (cost=0.00..25.92 rows=223 width=0) (actual time=0.068..0.068 rows=4 loops=1)  
        Index Cond: (title_lexeme @@ to_tsquery('волчий|волк'::text))  
Planning Time: 0.331 ms  
Execution Time: 0.138 ms  
```

### 4. Реализовать индекс на часть таблицы или индекс на поле с функцией

Частичный индекс создадим для поля со ссылкой на оригинальное произведение.  
У ~50% записей (произведение на языке оригинала) в этом поле NULL,  
у других ~50% записей (переводы) в этом поле содержится foreign key.

```SQL
-- Частичный индекс по полю originalworkid (на значения IS NOT NULL)
CREATE INDEX if not exists Work_OriginalWorkId_NotNull_idx ON catalog.Work (OriginalWorkId)
WHERE OriginalWorkId IS NOT NULL;
```

```SQL
-- Выберем по OriginaWorkId часть переводов  
SELECT w.WorkId,
       w.Name,
       w.Year,
       w.TranslationYear
FROM catalog.Work AS w
WHERE w.OriginalWorkId < 200;
```

**Результат команды Explain Analyze** - используется частичный индекс  
```md
QUERY PLAN  
Index Scan using work_originalworkid_notnull_idx on work w  (cost=0.29..14.41 rows=106 width=25) (actual time=0.015..0.040 rows=64 loops=1)  
  Index Cond: (originalworkid < 200)  
Planning Time: 0.302 ms  
Execution Time: 0.066 ms  
```
По сравнению с полным индексом частичный:
- не приводит к выигрышу во времени - время выполнения запросов получается примерно одинаковым,
- занимает меньше места на диске: полный индекс - 1,41Мб, частичный - 1,09Мб (на 23% меньше).  

### 5. Создать индекс на несколько полей

Для таблицы изданных книг создадим индекс на поля год публикации и наличие на складе.  
Поле с годом публикации поставим в индексе первым, так как у него более высокая кардинальность (значения с 2000 года по 2022 год) по сравнению с наличием на складе (4 возможных значения).

```SQL
-- Создаем многоколоночный индекс на поля год публикации и статус наличия 
CREATE INDEX if not exists Book_PublishYear_Status_idx ON catalog.Book(PublishYear, Status);

-- Выбираем нераспроданные книги, выпущенные в 2005 году.
SELECT Book.ISBN,
       Book.BookTitle,
       Book.BookPrice,
       Book.PublishYear,
       Book.Status
FROM catalog.Book
WHERE Book.PublishYear = 2005
AND Book.Status = 'есть на складе';
```
**Результат команды Explain Analyze**  
Используется созданный индекс по двум полям  
```md
QUERY PLAN  
Bitmap Heap Scan on book  (cost=12.18..1408.10 rows=770 width=61) (actual time=0.251..1.183 rows=740 loops=1)  
  Recheck Cond: ((publishyear = 2005) AND (status = 'есть на складе'::catalog.statusbook))  
  Heap Blocks: exact=508  
  ->  Bitmap Index Scan on book_publishyear_status_idx  (cost=0.00..11.99 rows=770 width=0) (actual time=0.166..0.166 rows=740 loops=1)  
        Index Cond: ((publishyear = 2005) AND (status = 'есть на складе'::catalog.statusbook))  
Planning Time: 0.245 ms  
Execution Time: 1.304 ms  
```

Этот же индекс может использоваться при выборке по одному из полей. 
Выберем книги по полю год публикации (publishyear, указано в индексе первым), без учета статуса.  
```SQL
-- Выборка книг, вышедших с 2000 по 2010 годы
SELECT Book.ISBN,
       Book.BookTitle,
       Book.BookPrice,
       Book.PublishYear,
       Book.Status
FROM catalog.Book
WHERE Book.PublishYear BETWEEN 2000 AND 2010;
```
Как видно индекс используется, хотя условие наложено только на одно поле из двух
```md
QUERY PLAN  
Bitmap Heap Scan on book  (cost=332.16..2430.04 rows=23792 width=61) (actual time=1.665..15.365 **rows=23928** loops=1)  
  Recheck Cond: ((publishyear >= 2000) AND (publishyear <= 2010))  
  Heap Blocks: exact=1152  
  ->  Bitmap Index Scan on book_publishyear_status_idx  (cost=0.00..326.21 rows=23792 width=0) (actual time=1.479..1.479 rows=23928 loops=1)  
        Index Cond: ((publishyear >= 2000) AND (publishyear <= 2010))  
Planning Time: 0.202 ms  
Execution Time: 16.311 ms  
```  

Этот же индекс может использоваться при выборке по полю status (наличие на складе), которое указано в индексе вторым. При условии, что выборка небольшая  

```SQL
-- Выборка книг, которые готовятся к печати, без учета года
SELECT Book.ISBN,
       Book.BookTitle,
       Book.BookPrice,
       Book.PublishYear,
       Book.Status
FROM catalog.Book
WHERE Book.Status = 'готовится к печати';
```

Выборка 202 записей (это менее 1% записей от общего числа), многоколоночный индекс используется, хотя условие наложено только на поле, которое указано вторым  
```md
QUERY PLAN  
Bitmap Heap Scan on book  (cost=559.55..1126.15 rows=197 width=61) (actual time=0.120..0.470 rows=202 loops=1)  
  Recheck Cond: (status = 'готовится к печати'::catalog.statusbook)  
  Heap Blocks: exact=175  
  ->  Bitmap Index Scan on book_publishyear_status_idx  (cost=0.00..559.50 rows=197 width=0) (actual time=0.090..0.090 rows=202 loops=1)  
        Index Cond: (status = 'готовится к печати'::catalog.statusbook)  
Planning Time: 0.147 ms  
Execution Time: 0.535 ms  
```  

При увеличении результирующей выборки (17446 записей, это ~35%)  
```SQL
-- Выборка книг, которые есть на складе, без учета года выпуска
SELECT Book.ISBN,
       Book.BookTitle,
       Book.BookPrice,
       Book.PublishYear,
       Book.Status
FROM catalog.Book
WHERE Book.Status = 'есть на складе';
```  
оптимизатор уже перестает обращаться к индексу и использует последовательное сканирование   
```md
QUERY PLAN  
Seq Scan on book  (cost=0.00..2366.35 rows=17600 width=61) (actual time=0.023..19.867 **rows=17446** loops=1)  
  Filter: (status = 'есть на складе'::catalog.statusbook)  
  Rows Removed by Filter: 32582  
Planning Time: 0.141 ms  
Execution Time: 20.555 ms
```

### 6. Написать комментарии к каждому из индексов  
  
Индексы на все поля, которые являются внешними ключами Foreign Key  
```SQL
-- Таблица Печатная книга - Индексы на коды издательства и издательской серии
CREATE INDEX if not exists Book_PublisherID_idx ON catalog.book(PublisherID);
CREATE INDEX if not exists Book_PublisherSeriesID_idx ON catalog.book(PublisherSeriesID);
-- Таблица Произведение - Индексы на коды языка, формы и серии произведения
CREATE INDEX if not exists Work_LanguageID_idx ON catalog.Work(LanguageID);
CREATE INDEX if not exists Work_FormID_idx ON catalog.Work(FormID);
CREATE INDEX if not exists Work_WorkSeriesID_idx ON catalog.Work(WorkSeriesID);
-- Таблица Произведение - Частичные индексы по полю со ссылкой на оригинальное произведение и ссылкой на исходный язык для перевода
CREATE INDEX if not exists Work_OriginalWorkId_notnull_idx ON catalog.Work (OriginalWorkId) WHERE OriginalWorkId IS NOT NULL;
CREATE INDEX if not exists Work_SourceLanguageId_notnull_idx ON catalog.Work (SourceLanguageId) WHERE SourceLanguageId IS NOT NULL;
-- Таблица Автор - Индекс на код страны
CREATE INDEX if not exists Author_CountryID_idx ON catalog.Author(CountryID);
-- Таблица Заказ на печать - Индекс на код ISBN
CREATE INDEX if not exists PrintOrder_ISBN_idx ON accounting.PrintOrder(ISBN);
-- Таблица Заказ - Индекс на код Покупателя
CREATE INDEX if not exists Order_CustomerId_idx ON accounting.Order(CustomerId);
```  
Индексы на поля, которые чаще всего используются для поиска книг по каталогу:  
название книги, название произведения, автор, цена
```SQL
-- Индекс на поле с ценой
CREATE INDEX if not exists Book_BookPrice_idx ON catalog.Book (BookPrice);

-- Индексы GIN для полнотекстового поиска по названию книги, произведения, имени автора
CREATE INDEX if not exists Book_BookTitleLexeme_idx ON catalog.Book USING GIN (Title_Lexeme);
CREATE INDEX if not exists Work_NameLexeme_idx ON catalog.Work USING GIN (Name_Lexeme);
CREATE INDEX if not exists Author_FirstLastNameLexeme_idx ON catalog.Author USING GIN (FirstLastName_Lexeme);
```

Индексы, которые нужны для построения отчетных таблиц (с использованием GROUP BY):
какие заказы делала конкретная организация, заказы за выбранный период времени, нераспроданные наименования по годам выпуска и т.п.
```SQL
-- Индекс на поле с именем заказчика
CREATE INDEX if not exists Customer_CustomerName_idx ON accounting.Customer (CustomerName);
-- Многоколоночный индекс на дату и статус заказа
CREATE INDEX if not exists Order_OrderDateStatus_idx ON accounting.Order (OrderDate, Status);
-- Многоколоночный индекс на поля год публикации и статус наличия
CREATE INDEX if not exists Book_PublishYear_Status_idx ON catalog.Book(PublishYear, Status);
```