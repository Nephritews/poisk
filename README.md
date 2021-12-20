# Оглавление

1. [Поиск товаров](#_Toc90547690)

	1. [Краткое описание поиска](#_Toc90547691)

	2. [Особенности поиска](#_Toc90547694)

		1. [Смена раскладки](#_Toc90547698)

		2. [Блочная структура](#_Toc90547700)

		3. [Поиск с приоритетами](#_Toc90547699)

		4. [Сортировка по наличию в аптеках](#_Toc90547700)

		5. [Экранирование спец символов](#_Toc90547702)

		6. [Алгоритм поиска](#_Toc90547703)

		7. [Короткие запросы](#_Toc90547695)

		8. [Длинные запросы](#_Toc90547696)

		9. [Символы в запросах](#_Toc90547697)

2. [Бэкенд часть](#_Toc90547704)

	1. [Описание данных (Способ хранения в elasticsearch)](#_Toc90547692)

	2. [Анализаторы поиска](#_Toc90547693)

	3. [Формирование запроса к elasticsearch](#_Toc90547705)

	4. [Формирование поисковой части](#_Toc90547706)

	5. [Формирование фильтров](#_Toc90547707)

	6. [Поиск фильтров](#_Toc90547708)

# Поиск товаров

## Краткое описание сервиса поиска

Сервис поиска является вебсервисом прослойкой между фронтендом (сайт, мобильное приложение) и elasticsearch.
Сервис принимает запросы от фронта, трансформирует их в json запрос к elasticsearch, принимает ответ, разбирает его и отдаёт обратно на фронтенд

## Особенности поиска

### Короткие запросы

Поиск на сайте не обрабатывает короткие запросы, в которых каждое слово содержат в себе ⩽ 2 символов.
Исключением является запрос b4 (запросы: b4, б4, в4, B4, Б4 и В4 заменяется на «б4 коммерция»).
В случае, когда запрос состоит из нескольких слов (где хотя бы одно не меньше 3х символов),
поиск осуществляется по всему запросу целиком, не исключая слова, состоящие из менее чем 3х символов.

Для примера запрос вида “aa bb cс” не обрабатывается, запрос вида “aaa bb cc” обрабатывается, не отбрасывая слова менее 3х символов.

### Длинные запросы

Длинными запросами считаются любые запросы, состоящие из более чем 50 символов.
При получении запроса более 50 символов, поиск осуществляется по первым 50, остальная часть запроса игнорируется.

### Символы в запросах

Символы + и - в запросе заменяются на пробел.

### Смена раскладки

Для обработки поисковых запросов в неверной раскладке реализован следующий алгоритм.
Любой поисковый запрос на сайте, формирует для сервиса elasticsearch два запроса, с оригинальной и измененной раскладкой.
Из результатов поиска выбирается более подходящий вариант. Более подробно описано в разделе «Алгоритм поиска».

### Блочная структура.

Поисковая выдача по своей сути разделена на блоки, каждый из которых удовлетворяет конкретным критериям.
Более подробно критерии описаны в разделе "Формирование поисковой части".

### Поиск с приоритетами.

Первый блок выдачи отдаётся для приоритетных товаров по действующему веществу.

### Сортировка по наличию в аптеках.

Товары внутри одного блока отсортированы по количеству аптек в которых их можно купить.

### Экранирование спец символов

Есть два вида экранирования символов - замена в тексте управляющих символов на соответствующие текстовые подстановки.
Одно из них происходит на фронтенде и нужно для того, чтобы формировалась правильная ссылка (символ % заменяется на %25).
Вторая экранирует следующие символы: +, -, =, &, |, !, (, ), {, }, [,], ^, ", ~, *, <, >, ?, :, \, / потому что они
являются спецсимволами Elasticsearch, для экранирования применяется символ \. Например, для символа + экранирование будет выглядеть как \+.

# Данные в elasticsearch

## Описание данных (Способ хранения в elasticsearch)

Данные в elasticsearchусловно можно разбить на две составляющие:

- Данные о наличии остатка в конкретной аптеке
- Данные о товаре, содержит название товара, форму выпуска, производителя и т.д.

Итоговая JSON с информацией по индексу:

```json
{
    "settings": {
        "analysis": {
            "tokenizer": {
                "class": {
                    "max_token_length": "15",
                    "type": "classic"
                },
                "engram": {
                    "max_gram": "15",
                    "type": "edge_ngram",
                    "token_chars": [
                        "letter",
                        "digit",
                        "punctuation",
                        "symbol"
                    ]
                }
            },
            "analyzer": {
                "words_analizer": {
                    "tokenizer": "whitespace",
                    "type": "custom",
                    "char_filter": [
                        "my_char_filter"
                    ],
                    "filter": [
                        "words",
                        "lowercase",
                        "unique",
                        "trunc"
                    ]
                },
                "default": {
                    "tokenizer": "whitespace",
                    "type": "custom",
                    "char_filter": [
                        "my_char_filter"
                    ],
                    "filter": [
                        "words",
                        "engram",
                        "lowercase",
                        "unique"
                    ]
                },
                "default_search": {
                    "tokenizer": "whitespace",
                    "type": "custom",
                    "filter": [
                        "lowercase",
                        "unique",
                        "trunc"
                    ]
                }
            },
            "char_filter": {
                "my_char_filter": {
                    "mappings": [
                        "( => ",
                        ") => "
                    ],
                    "type": "mapping"
                }
            },
            "filter": {
                "words": {
                    "catenate_words": "true",
                    "preserve_original": "true",
                    "type": "word_delimiter"
                },
                "trunc": {
                    "type": "truncate",
                    "length": "15"
                },
                "engram": {
                    "max_gram": "15",
                    "type": "edgeNGram"
                }
            }
        }
    },
    "mappings": {
        "catalog": {
            "properties": {
                "url": {
                    "ignore_above": 256,
                    "type": "keyword"
                },
                "trade_name": {
                    "search_analyzer": "default_search",
                    "analyzer": "default",
                    "fields": {
                        "words": {
                            "search_analyzer": "default_search",
                            "analyzer": "words_analizer",
                            "type": "text"
                        },
                        "keyword": {
                            "ignore_above": 256,
                            "type": "keyword"
                        }
                    },
                    "type": "text"
                },
                "symptoms": {
                    "search_analyzer": "default_search",
                    "analyzer": "default",
                    "fields": {
                        "keyword": {
                            "ignore_above": 256,
                            "type": "keyword"
                        }
                    },
                    "type": "text"
                },
                "super_stock": {
                    "type": "long"
                },
                "subsection": {
                    "search_analyzer": "default_search",
                    "analyzer": "default",
                    "fields": {
                        "keyword": {
                            "ignore_above": 256,
                            "type": "keyword"
                        }
                    },
                    "type": "text"
                },
                "stock": {
                    "type": "long"
                },
                "section": {
                    "search_analyzer": "default_search",
                    "analyzer": "default",
                    "fields": {
                        "keyword": {
                            "ignore_above": 256,
                            "type": "keyword"
                        }
                    },
                    "type": "text"
                },
                "search_rang_symptoms": {
                    "type": "long"
                },
                "search_rang": {
                    "ignore_above": 256,
                    "type": "keyword"
                },
                "rzv": {
                    "search_analyzer": "default_search",
                    "analyzer": "default",
                    "fields": {
                        "keyword": {
                            "ignore_above": 256,
                            "type": "keyword"
                        }
                    },
                    "type": "text"
                },
                "price": {
                    "type": "double"
                },
                "pharm_id": {
                    "ignore_above": 256,
                    "type": "keyword"
                },
                "pharm": {
                    "relations": {
                        "product": "pharm"
                    },
                    "eager_global_ordinals": true,
                    "type": "join"
                },
                "osno": {
                    "search_analyzer": "default_search",
                    "analyzer": "default",
                    "fields": {
                        "keyword": {
                            "ignore_above": 256,
                            "type": "keyword"
                        }
                    },
                    "type": "text"
                },
                "name": {
                    "search_analyzer": "default_search",
                    "analyzer": "default",
                    "fields": {
                        "words": {
                            "search_analyzer": "default_search",
                            "analyzer": "words_analizer",
                            "type": "text"
                        },
                        "keyword": {
                            "ignore_above": 256,
                            "type": "keyword"
                        }
                    },
                    "type": "text"
                },
                "mnn": {
                    "search_analyzer": "default_search",
                    "analyzer": "default",
                    "fields": {
                        "words": {
                            "search_analyzer": "default_search",
                            "analyzer": "words_analizer",
                            "type": "text"
                        },
                        "keyword": {
                            "ignore_above": 256,
                            "type": "keyword"
                        }
                    },
                    "type": "text"
                },
                "maker": {
                    "ignore_above": 256,
                    "type": "keyword"
                },
                "lat_name": {
                    "fields": {
                        "words": {
                            "search_analyzer": "default_search",
                            "analyzer": "words_analizer",
                            "type": "text"
                        }
                    },
                    "search_analyzer": "default_search",
                    "analyzer": "default",
                    "type": "text"
                },
                "idtovar": {
                    "ignore_above": 256,
                    "type": "keyword"
                },
                "id": {
                    "ignore_above": 256,
                    "type": "keyword"
                },
                "form": {
                    "ignore_above": 256,
                    "type": "keyword"
                },
                "ean13": {
                    "search_analyzer": "default_search",
                    "analyzer": "default",
                    "fields": {
                        "keyword": {
                            "ignore_above": 256,
                            "type": "keyword"
                        }
                    },
                    "type": "text"
                },
                "cont": {
                    "search_analyzer": "default_search",
                    "analyzer": "default",
                    "fields": {
                        "keyword": {
                            "ignore_above": 256,
                            "type": "keyword"
                        }
                    },
                    "type": "text"
                }
            }
        }
    }
}
```

Из данной JSON можно выделить две части с информаций об остатках и информаций о товарах. Поле pharm, по которому можно отличить какая информация содержится в JSON.

Пример части JSON с информацией об остатках товаров:

``` json
{
    "_source": {
        "pharm": {
            "parent": "179833",
            "name": "pharm"
        },
        "price": 0,
        "id": "179833",
        "stock": 0,
        "pharm_id": "7407"
    }
}
```

- pharm_id - id аптеки, в которой находится этот остаток
- stock - Остаток товара в аптеке
- price - Цена товара в аптеке (соответствует онлайн цене в соответствующем регионе)
- id - Id\_mp код товара. Используется для выдачи.
- parent - Id\_mp товара

Пример части JSON с информацией о товарах:

```json
{
    "_source": {
        "url": "/catalog/lekarstva-i-bady/zdorovoe-serdtse-i-sosudy/karvedilol-akrikhin-tab-12-5mg-30",
        "super_stock": 0,
        "search_rang": 0,
        "mnn": [
            "Карведилол"
        ],
        "form": "Таблетки",
        "maker": "АКРИХИН ХФК АО",
        "lat_name": [],
        "id": "159780",
        "ean13": [
            "4601969007480"
        ],
        "price": [
            189
        ],
        "trade_name": [],
        "pharm": "product",
        "name": "Карведилол Акрихин таблетки 12,5мг, №30 Акрихин"
    },
    "_score": 2,
    "_version": 1,
    "_id": "159780",
    "_type": "catalog",
    "_index": "ve-site-search"
}
```

- id - id_mp - Код товара. Используется для выдачи.
- name - Полное название товара. Используется для поиска и выдачи.
- lat_name - Название товара на латинице. Используется для поиска.
- trade_name - Коммерческое наименование товара. Используется для поиска.
- form - Форма выпуска. Используется для выдачи и фильтрации.
- maker- Производитель. Используется для выдачи и фильтрации.
- mnn - Активное вещество. Используется для поиска, выдачи и фильтрации.
- search_rang - Поле для приоритета при поиске по mnn.
- ean13 - Штрих-код товара. Поле для поиска по штрих-коду.
- url - URL товара. Используется в выдаче.
- price - Цена товара в аптеке (соответствует онлайн цене в соответствующем регионе)
- idtovar - Аптечный id товара.
- symptoms – Поле для поиска по симптомам. Не используется.
- search_rang_symptoms - Поле для приоритета поиска по симптомам. Не используется.
- subsection - ID подсекции товара. Не используется.
- section - ID секции товара, сейчас не используется.

## Анализаторы поиска

Для всех полей, которые используются для поиска указаны три анализатора: default, default_search и words_analizer.
Разбор исходных данных - words_analizer.
```json
{
    "tokenizer": {
        "engram": {
            "max_gram": "15",
            "type": "edge_ngram",
            "token_chars": [
                "letter",
                "digit",
                "punctuation",
                "symbol"
            ]
        }
    },
    "analyzer": {
        "words_analizer": {
            "tokenizer": "whitespace",
            "type": "custom",
            "char_filter": [
                "my_char_filter"
            ],
            "filter": [
                "words",
                "lowercase",
                "unique",
                "trunc"
            ]
        }
    },
    "char_filter": {
        "my_char_filter": {
            "mappings": [
                "( => ",
                ") => "
            ],
            "type": "mapping"
        }
    },
    "filter": {
        "words": {
            "catenate_words": "true",
            "preserve_original": "true",
            "type": "word_delimiter"
        },
        "trunc": {
            "type": "truncate",
            "length": "15"
        }
    }
}
```

tokenizer разбирает поисковый запрос по пробелам (whitespace)

filter разбивает на слова (words) (например Но-шпа разбивается на [Но-шпа, Ношпа, Но, шпа]).
Частью слова являются буквы, числа, знаки препинания и символы, кроме пробелов (letter, digit, punctuation, symbol).
Далее приводит все токены к нижнему регистру (lowercase) и удаляет повторения (unique)

char-filter удаляет все ( и )

Разбор исходных данных – default.

```json
{
    "analyzer": {
        "default": {
            "tokenizer": "whitespace",
            "type": "custom",
            "char_filter": [
                "my_char_filter"
            ],
            "filter": [
                "words",
                "engram",
                "lowercase",
                "unique"
            ]
        }
    },
    "filter": {
        "words": {
            "preserve_original": true,
            "catenate_words": true,
            "type": "word_delimiter"
        },
        "engram": {
            "type": "edgeNGram",
            "max_gram": "15"
        }
    }
}
```

tokenizer разбирает поисковый запрос по пробелам (whitespace)

filter разбивает на слова (words) (например Но-шпа разбивается на [Но-шпа, Ношпа, Но, шпа]), после разбираем их как префиксы (engram) длины до 15 символов (max_gram).
Частью слова являются буквы, числа, знаки препинания и символы, кроме пробелов (letter, digit, punctuation, symbol).
Например нуро1!& разбирается в [н, ну, нур, нуро, нуро1, нуро1!, нуро1!&]. Далее приводит все токены к нижнему регистру (lowercase) и удаляет повторения (unique)

char-filter удаляет все ( и )

Разбор поисковой строки - search_analyzer

```json
{
    "analyzer": {
        "default_search": {
            "tokenizer": "whitespace",
            "type": "custom",
            "filter": [
                "lowercase",
                "unique",
                "trunc"
            ]
        }
    },
    "filter": {
        "trunc": {
            "type": "truncate",
            "length": "15"
        }
    }
}
```

Tokenizer разбирает поисковый запрос по пробелам (whitespace)

filter приводит все токены к нижнему регистру (lowercase), удаляет повторения (unique), обрезает все токены до 15 символов.

# Алгоритм поиска


## Бэкенд часть

На веб сервис поиска посылается запрос следующего вида:

``` json
{
    "sessionId": "",
    "sort": "",
    "min": "0",
    "max": "0",
    "mnn": [
        {
            "show": "true",
            "ch": "true",
            "doc_count": "2",
            "key": "Парацетамол"
        }
    ],
    "ean13": "",
    "rev": "true",
    "type": "0",
    "from": "0",
    "size": "24",
    "q": "нурофен"
}
```

- q - непосредственно поисковый запрос
- size - количество товаров, которые нужно вернуть
- from - с какой позиции начинать искать
- type - булева переменная, 0 - ничего не происходит, 2 - добавляет поисковый запрос в фильтры подействующему веществу (mnn)
- rev - булева переменная, означающая, надо ли попытаться поменять раскладку пользователя
- ean13 - штрихкод
- mnn, form, maker - фильтрация по полям действующее вещество, форма выпуска и производитель соответственно (в зависимости от запроса подставляется mnn и/ или form и/или maker)
- key - ключ фильтрации
- doc_count - количество товаров, удовлетворяющих этому фильтру
- ch - выбран ли фильтр
- show - надо ли этот фильтр показывать
- max - фильтрация по максимальной цене
- min - фильтрация по минимальной цене
- sort- сортировка (если есть)
- SessionId - необязательное поле, помогающее определиться с нодой elasticsearch, в которую пойдёт запрос

Оригинальный запрос сразу обрезается до 50 символов, а + и - заменяются на пробел. Далее создаются две новые переменные:

- query_front - поисковый запрос, который нужно отобразить в поисковой строке

- query_filter - реальный поисковый запрос, который в дальнейшем отправится в запрос на получение фильтров. Первоначально эти переменные заполняются оригинальной строкой поиска. Дальше отрабатывает исключение поb4, если оригинальный запрос это:b4, б4, в4, B4, Б4 и В4 - то он и значение поляquery\_filterзаменяется на «б4 коммерция». После этого происходит проверка на короткий запрос: если в оригинальном запросе нет ни одного слова длиннее двух символов, то возвращается ответ:

``` json
{
    "IS_REVERTED": false,
    "SORT": null,
    "QUERY_FRONT": "fc fc fc",
    "QUERY_FILTER": "fc fc fc",
    "MESSAGE": "Поисковый запрос должен быть не менее 3 символов!"
}
```

- from - с какой позиции надо будет искать при нажатии на кнопку ещё
- paginate - надо ли обрисовывать кнопку ещё
- isReverted - была ли изменена раскладка
- total - общее количество результатов поиска
- max_score - максимальное значение score в результатах поиска
- goods- непосредственно результат поиска

Внешний вид элемента goods:

``` json

{
    "isAlcohol": false,
    "isTermo": false,
    "limit_type": 0,
    "count": 5,
    "MAX_PRICE": false,
    "DELIVERY": false,
    "VITAMIX": false,
    "ICONS": [],
    "STRONG_RECIPE": false,
    "RECIPE": false,
    "PRICE": 230,
    "PKU": false,
    "PREVIEW_PICTURE": "https://pics.vitaexpress.ru/public/images/medium/146323.jpg",
    "URL": "/catalog/lekarstva-i-bady/obezbolivayushchie/nurofen-tab-200mg-20/",
    "NAME": "Нурофен таблетки 200мг, №20",
    "XML_ID": 146323,
    "ID": 41325
}
```

- ID - ID товара из 1С-Битрикс
- XML\_ID - ID\_MP товара
- NAME - Название товара
- URL - Ссылка на товар
- PREVIEW\_PICTURE - Картинка товара
- PKU – Маркер «Доступен только в аптеке»
- PRICE - Цена товара
- RECIPE - Шильдик рецептурного препарата
- STRONG\_RECIPE - Шильдик строго рецептурного препарата
- ICONS - Массив с шильдиками для акционных товаров
- VITAMIX - Путь до шильдика Витамикс
- DELIVERY - Булево значение, маркер доставки (товар доступен для доставки клиенту)
- MAX\_PRICE - Зачеркнутая цена
- count - Максимальное число товара, доступное для заказа
- limit\_type - Поле не используется
- isTermo - Маркер термолабильности
- isAlcohol - Маркер спиртосодержащего товара

Далее из результатов поиска забираются id товаров, и по ним запрашиваются данные в 1С-Битрикс. Общий вид результата, следующий:

``` json
 {
    "VITAMIX_LOGO": "/upload/iblock/c62/c62e5769befb97925665f5b7625cdd89.svg",
    "SVG_SPRITE": "102156415",
    "TOTAL": 26,
    "IS_REVERTED": false,
    "SORT": {
        "direction": "desc",
        "field": "_score"
    },
	"GOODS": [{…}, {…}, {…}, {…}, {…}, …],
    "QUERY_FRONT": "парацетамол",
    "QUERY_FILTER": "парацетамол",
    "MESSAGE": ""
}
```

- MESSAGE - сообщение, которое нужно вывести перед результатом поиска
- QUERY_FILTER - поисковый запрос, который нужно отправить в поиск фильтров
- QUERY_FRONT - поисковый запрос, который нужно от рисовать пользователю
- SORT - сортировка результата
- IS_REVERTED - маркер смены раскладки
- TOTAL - количество результатов поиска
- GOODS - товары
- SVG_SPRITE - код отображения цены
- VITAMIX_LOGO - ссылка на шильдик Витамикс

## Формирование запроса к elasticsearch

### Формирование поисковой части

Поисковый запрос логически можно разделить на две части: 
1. Вычисление количества аптек, в которых доступен товары,
2. Блоки. 

Вмесие эти две части формируют `score` по которой по умолчанию сортируются товары в поисковой выдаче.
Формируется он по следующей логике. Наличие в аптеке прибавляет к `score` единицу. Попадание же в блок
прибавляет к `score` число равное: (количество запрашиваемых аптек) * (номер блока, в чью выдачу попал этот товар).

Блоки имеют следующие критерии (блоки для удобства перечислены в порядке от большего номера к меньшему):

1. Поиск по всем словам в mnn с ошибками с приоритетом (Работает на сайте и в мп, но не работает в быстром поиске на сайте)
2. Поиск по всем словам в name или по всем словам в mnn без ошибок
3. Поиск по всем словам в name с ошибками
4. Поиск по приоритетному слову как префикс в name
5. Поиск по приоритетному слову в name c ошибками
6. Поиск чистого запроса в name
7. Поиск чистого запроса в mnn как префикс
8. Поиск чистого запроса в mnn с ошибками
9. Поиск чистого запроса по name, lat_name, trade_name, mnn c одной ошибкой
10. Поиск чистого запроса по name, lat_name, trade_name, mnn как префикс

В json формате часть с количеством аптек выглядит так:
```json
{
    "has_child": {
        "query": {
            "constant_score": {
                "filter": {
                    "bool": {
                        "filter": [
                            {
                                "terms": {
                                    "pharm_id": [
                                        7475,
                                        7458,
                                        7419
                                    ]
                                }
                            },
                            {
                                "range": {
                                    "stock": {
                                        "gt": 0
                                    }
                                }
                            }
                        ]
                    }
                }
            }
        },
        "score_mode": "sum",
        "type": "pharm"
    }
}
```

А блоки так:

``` json
{
    "functions": [
        {
            "filter": {
                "bool": {
                    "must": [
                        {
                            "match": {
                                "mnn.words": {
                                    "fuzziness": "AUTO:6,9",
                                    "query": "нурофен"
                                }
                            }
                        },
                        {
                            "range": {
                                "search_rang": {
                                    "gt": 0
                                }
                            }
                        }
                    ]
                }
            },
            "weight": 36
        },
        {
            "filter": {
                "match": {
                    "name.words": {
                        "operator": "AND",
                        "query": "нурофен"
                    }
                }
            },
            "weight": 33
        },
        {
            "filter": {
                "match": {
                    "trade_name.words": {
                        "operator": "AND",
                        "query": "нурофен"
                    }
                }
            },
            "weight": 30
        },
        {
            "filter": {
                "match": {
                    "name.words": {
                        "operator": "AND",
                        "fuzziness": "AUTO:4,6",
                        "query": "нурофен"
                    }
                }
            },
            "weight": 27
        },
        {
            "filter": {
                "match": {
                    "trade_name.words": {
                        "operator": "AND",
                        "fuzziness": "AUTO:4,6",
                        "query": "нурофен"
                    }
                }
            },
            "weight": 24
        },
        {
            "filter": {
                "match": {
                    "name": {
                        "query": "нурофен"
                    }
                }
            },
            "weight": 21
        },
        {
            "filter": {
                "match": {
                    "name.words": {
                        "fuzziness": "AUTO:4,6",
                        "query": "нурофен"
                    }
                }
            },
            "weight": 18
        },
        {
            "filter": {
                "match": {
                    "name": {
                        "query": "нурофен"
                    }
                }
            },
            "weight": 15
        },
        {
            "filter": {
                "match": {
                    "mnn": {
                        "query": "нурофен"
                    }
                }
            },
            "weight": 12
        },
        {
            "filter": {
                "match": {
                    "mnn.words": {
                        "fuzziness": "AUTO:4,6",
                        "query": "нурофен"
                    }
                }
            },
            "weight": 9
        },
        {
            "filter": {
                "multi_match": {
                    "fuzziness": "AUTO:4,6",
                    "fields": [
                        "name.words",
                        "mnn.words",
                        "lat_name.words",
                        "trade_name.words"
                    ],
                    "query": "нурофен"
                }
            },
            "weight": 6
        },
        {
            "filter": {
                "multi_match": {
                    "fields": [
                        "name",
                        "mnn",
                        "lat_name",
                        "trade_name"
                    ],
                    "query": "нурофен"
                }
            },
            "weight": 3
        }
    ]
}
```

### Формирование фильтров

Фильтр по наличию, применяется всегда:

``` json
{
    "has_child": {
        "query": {
            "bool": {
                "filter": [
                    {
                        "range": {
                            "stock": {
                                "gt": 0
                            }
                        }
                    },
                    {
                        "terms": {
                            "pharm_id": [
                                1335,
                                12,
                                143
                            ]
                        }
                    },
                    {
                        "range": {
                            "price": {
                                "lte": 150,
                                "gte": 0
                            }
                        }
                    }
                ]
            }
        },
        "inner_hits": {
            "size": 1
        },
        "type": "pharm"
    }
}
```

С его помощь происходит проверка на наличие товара, хотя бы в одной из аптек, (в данном случае [1335, 12, 143]) а именно, существуют ли данные о товаре, в перечисленных аптеках, у которых значение поля stock больше 0. В этот же момент формируются фильтры по цене, в данном примере, фильтруется по минимальной и максимальной цене 0 и 150 соответственно (qte и lte)Отдельно формируются фильтры по действующему веществу (mnn):

[{&quot;term&quot;: {

&quot;mnn&quot;: &quot;Ибупрофен&quot;}},{

&quot;term&quot;: {

&quot;mnn&quot;: &quot;Аскорил&quot;}}]

И отдельно формируются все остальные:

{&quot;terms&quot;: {

&quot;form&quot;: [

&quot;таблетки&quot;,

&quot;суспензия&quot;],

&quot;maker&quot;: [

&quot;Рус био пара гомео фарм&quot;]}}

Фильтры формируются по отдельности, потому что у них разная логика объединения. По действующему веществу (mnn) товар должен удовлетворять всем выбранным фильтрам, а по форме выпуска (form) и производителю (maker) хотя бы одному.

## **Поиск фильтров**

Поиск фильтров можно описать как агрегацию поиска товаров. То есть на основе результатов поиска, собираются все поля mnn, form и maker (действующее вещество, форма выпуска и производитель соответственно), и передаются на фронтенд сайта, для отображения фильтров на странице Результатов поиска.

Теперь рассмотрим этот вопрос более подробно. Бэкенд сайта не агрегирует результат поиска, а формирует запрос к elasticsearch. В первую очередь надо заметить, что поисковая часть запроса фильтров совпадает с поисковой частью поиска товаров (см. раздел Алгоритм поиска), поэтому её разбирать не будем.

Фильтрация не применяется для первичного поискового запроса. Сделано это для того, чтобы получить полные данные для формирования панели фильтрации на странице Результаты поиска. При применении фильтра на фронтенде, панель фильтрации должна быть обновлена в соответствии с примененными фильтрами.

Есть несколько агрегаций, которые идут с общим префиксом all, а именно запросы, которые относятся к чистому неотфильтрованному запросу:

[{&quot;all\_mnns&quot;: {

&quot;terms&quot;: {

&quot;field&quot;: &quot;mnn.keyword&quot;,

&quot;size&quot;: 500}}},{

&quot;all\_makers&quot;: {

&quot;terms&quot;: {

&quot;field&quot;: &quot;maker&quot;,

&quot;size&quot;: 500}}},{

&quot;all\_forms&quot;: {

&quot;terms&quot;: {

&quot;field&quot;: &quot;form&quot;,

&quot;size&quot;: 500}}},{

&quot;all\_max&quot;: {

&quot;children&quot;: {

&quot;type&quot;: &quot;pharm&quot;},

&quot;aggs&quot;: {

&quot;ch&quot;: {

&quot;max&quot;: {

&quot;field&quot;: &quot;price&quot;}}}}},{

&quot;all\_min&quot;: {

&quot;children&quot;: {

&quot;type&quot;: &quot;pharm&quot;},

&quot;aggs&quot;: {

&quot;ch&quot;: {

&quot;min&quot;: {

&quot;field&quot;: &quot;price&quot;}}}}}]

В случае с mnn, maker и form это сбор значений из соответствующих полей, а в случае с all\_min и all\_max это сбор минимального и максимального значения из данных об остатках.

Далее существуют и другие агрегации, которые нужны как раз для того, чтобы отфильтровывать текущий результат. То есть лишь те фильтры, которые можно применить к текущей выдаче.

В примере ниже результат поиска фильтруется по действующему веществу (mnn), форме выпуска (form) и наличию (более подробно описано в разделе «Формирование фильтров»), в результате из него собираются все производители (maker):

{&quot;makers&quot;: {

&quot;filter&quot;: {

&quot;bool&quot;: {

&quot;must&quot;: [{

&quot;bool&quot;: {

&quot;must&quot;: &quot;Фильтрыпо mnn&quot;}},{

&quot;bool&quot;: {

&quot;should&quot;: &quot;Фильтрыпо form&quot;}},{

&quot;bool&quot;: {

&quot;should&quot;: &quot;Фильтры по наличию&quot;}}],

&quot;aggs&quot;: {

&quot;val&quot;: {

&quot;terms&quot;: {

&quot;field&quot;: &quot;maker&quot;,

&quot;size&quot;: 500}}}}}}}

Аналогично выглядит запрос для формы выпуска (form), разницатолько в том, что результаты фильтруются не по форме выпуска (form) а по производителю (maker):

{&quot;forms&quot;: {

&quot;filter&quot;: {

&quot;bool&quot;: {

&quot;must&quot;: [{

&quot;bool&quot;: {

&quot;must&quot;: &quot;Фильтрыпо mnn&quot;}},{

&quot;bool&quot;: {

&quot;should&quot;: &quot;Фильтрыпо maker&quot;}},{

&quot;bool&quot;: {

&quot;should&quot;: &quot;Фильтры по наличию&quot;}}],

&quot;aggs&quot;: {

&quot;val&quot;: {

&quot;terms&quot;: {

&quot;field&quot;: &quot;form&quot;,

&quot;size&quot;: 500}}}}}}}

Запрос по действующему веществу (mnn)отличается, потому что для действующего вещества (mnn)используется конъюнкция:

{&quot;mnn&quot;: {

&quot;filter&quot;: {

&quot;bool&quot;: {

&quot;must&quot;: [{

&quot;bool&quot;: {

&quot;must&quot;: &quot;Фильтрыпо mnn&quot;}},{

&quot;bool&quot;: {

&quot;should&quot;: &quot;Фильтрыпо form&quot;}},{

&quot;bool&quot;: {

&quot;should&quot;: &quot;Фильтрыпо maker&quot;}},{

&quot;bool&quot;: {

&quot;should&quot;: &quot;Фильтры по наличию&quot;}}],

&quot;aggs&quot;: {

&quot;val&quot;: {

&quot;terms&quot;: {

&quot;field&quot;: &quot;mnn&quot;,

&quot;size&quot;: 500}}}}}}}

В запросе для цен, агрегировать нужно записи о наличие данного товара в аптеках (более подробно описано в разделе «Формирование фильтров»), а не поля из записей о товарах (mnn, maker, form):

{&quot;price&quot;: {

&quot;filter&quot;: {

&quot;bool&quot;: {

&quot;must&quot;: [{

&quot;bool&quot;: {

&quot;should&quot;: &quot;Фильтрыпо maker&quot;}},{

&quot;bool&quot;: {

&quot;should&quot;: &quot;Фильрыпо form&quot;}},{

&quot;bool&quot;: {

&quot;must&quot;: &quot;Фильтыпо mnn&quot;}}]}}},

&quot;aggs&quot;: {

&quot;ch&quot;: {

&quot;children&quot;: {

&quot;type&quot;: &quot;pharm&quot;},

&quot;aggs&quot;: {

&quot;stock&quot;: {

&quot;filter&quot;: {

&quot;bool&quot;: {

&quot;filter&quot;: &quot;Фильтрыпоналичию&quot;}},

&quot;aggs&quot;: {

&quot;max&quot;: {

&quot;max&quot;: {

&quot;field&quot;: &quot;price&quot;}},

&quot;min&quot;: {

&quot;min&quot;: {

&quot;field&quot;: &quot;price&quot;}
