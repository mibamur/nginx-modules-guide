Emiller's Руководство по разработке модуля Nginx
================================================

By [Evan Miller](/)

Первая публикация: 28 Апреля, 2007 (Последняя редакция: 16 Января, 2013 – [changes](#changes))

Bruce Wayne: *What's that?*
 Lucius Fox: *The Tumbler? Oh… you wouldn't be interested in that.*

Чтобы полностью оценить Nginx, it helps to understand Batman, the comic book character.

Batman is fast. Nginx is fast. Batman fights crime. Nginx fights wasted CPU cycles and memory leaks. Batman performs well under pressure. Nginx, for its part, excels under heavy server loads.

But Batman would be almost nothing without the **Batman utility belt**.

![](images/utility_belt.jpg)

**Figure 1**: The Batman utility belt, gripping Christian Bale's love handles.

At any given time, Batman's utility belt might contain a lock pick, several batarangs, bat-cuffs, a bat-tracer, bat-darts, night vision goggles, thermite grenades, smoke pellets, a flashlight, a kryptonite ring, an acetylene torch, or an Apple iPhone. When Batman needs to tranquilize, blind, deafen, stun, track, stop, smoke out, or text-message the enemy, you better believe he's reaching down for his bat-belt. The belt is so crucial to Batman's operations that if Batman had to choose between wearing pants and wearing the utility belt, he would definitely choose the belt. In fact, he \*did\* choose the utility belt, and that's why Batman wears rubber tights instead of pants (Fig. 1).

Вместо пояса принадлежностей, Nginx имеет модули расширения. Когда Nginx должен применить GZIP или перекодировать ответ (response), он передает эту работу модулю. Когда Nginx блокирует доступ к ресурсам на основе IP-адреса или HTTP-аутентификации учетных данных, это делает вместо него модуль. Когда Nginx общается с Memcache или FastCGI серверами, это тоже делает модуль.

Batman's utility belt holds a lot of doo-hickeys, but occasionally Batman needs a new tool. Maybe there's a new enemy against whom bat-cuffs and batarangs are ineffectual. Or Batman needs a new ability, like being able to breathe underwater. That's when Batman rings up **Lucius Fox** to engineer the appropriate bat-gadget.

![](images/lucius.jpg)

**Figure 2**: Bruce Wayne (née Batman) consults with his engineer, Lucius Fox

Целью данного руководства является научить Вас деталям модуля Nginx. Когда вы закончите с этим руководством, вы будете в состоянии проектировать и производить высококачественные модули, которые позволяют Nginx делать вещи, которые он не мог раньше. Модульная система Nginx имеет много мельчайших нюансов, так что вы, вероятно, захотите часто возвращаться к этому документу. Я постарался сделать концепцию как можно более понятной, но скажу прямо - написание модулей Nginx тяжелая работа.

Но кто сказал, что это будет легко?

Содержание
-----------------

[Предпосылки](#prerequisites)

[High-Level высокоуровневая передача полномочий модулю Nginx's](#overview)

[Компоненты Nginx Module](#components)

[Структура Конфигурации Модуля](#configuration-structs)

[Директивы Модуля](#directives)

[Содержание Модуля](#context)

1.  [create\_loc\_conf](#create_loc_conf)
2.  [merge\_loc\_conf](#merge_loc_conf)

[Определение Модуля](#definition)

[Установка Модуля](#installation)

[Указателя](#handlers)

[Анатомия Указателя (Non-proxying)](#non-proxying)

1.  [Получение конфигурации location](#non-proxying-config)
2.  [Генерация ответа-response](#non-proxying-response)
3.  [Отправка Заголовка-header](#non-proxying-header)
4.  [Отправка тела-body](#non-proxying-body)

[Анатомия Upstream (a.k.a. Proxy) Указателя](#proxying)

1.  [Краткое изложение upstream callbacks](#proxying-summary)
2.  [The create\_request callback](#create_request)
3.  [The process\_header callback](#process_header)
4.  [Хранение состояния](#keeping-state)

[Установка Указателя](#handler-installation)

[Фильтры](#filters)

1.  [Анатомия Заголовка Фильтра Header Filter](#filters-header)
2.  [Анатомия Тело Фильтра Body Filter](#filters-body)
3.  [Установка Фильтра](#filters-installation)

[Балансировка нагрузки Load-Balancers](#load_balancers)

1.  [Применение дериктив](#lb-directive)
2.  [Функция регистрации](#lb-registration)
3.  [Функция инициализации upstream-а](#lb-upstream)
4.  [Функция инициализации peer](#lb-peer)
5.  [Функция Балансировка нагрузки load-balancing](#lb-function)
6.  [Функция инициализации peer release ](#lb-release)

[Написание и Сборка нового модуля Nginx](#compiling)

[Дополнительные материалы](#advanced)

[Ссылки на источник](#code)

0. Предпосылки
--------------

Вы должны хорошо знять C. Не просто "C-syntax"; Вы должны ориентироваться в структуре и не пугадться от ссылок на указатели и функции, и должы быть осводомлениы о работе пре-процессора preprocessor. Если Вы хотите освежить знания, Нет ничего лучше чем [K&R](http://en.wikipedia.org/wiki/The_C_Programming_Language_(book)).

Основы работы HTTP будут полезны. В конце-концов мы работаем на web сервере.

Вам также следует знать конфигурационные файлы Nginx. Если Вы не знаете, то в кратце существует четыре контекста *contexts* (называемые: основа *main*, сервер *server*, вышестоящий пункт назначения *upstream*, и расположение *location*) которые могут содержать директивы с один и более аргументом. Directives in the main context apply to everything; directives in the server context apply to a particular host/port; directives in the upstream context refer to a set of backend servers; and directives in a location context apply only to matching web locations (e.g., "/", "/images", etc.) A location context inherits from the surrounding server context, and a server context inherits from the main context. The upstream context neither inherits nor imparts its properties; it has its own special directives that don't really apply elsewhere. I'll refer to these four contexts quite a bit, so… don't forget them.

Давайте начнем.

1. High-Level высокоуровневая передача полномочий модулю Nginx's
----------------------------------------------------------------

Nginx модули работают с:

-   *handlers* обработчики запроса и получения выходных данных
-   *filters* манипулирование выходных данных полученных от обработчика *handler*
-   *load-balancers* выбор внутренего сервера для передачи запроса, когда серверов несколько

Всю "реальную работу", которую вы могли бы связать с веб-сервером, делают Модули: всякий раз, когда Nginx обслуживает файл или проксирует запрос на другой сервер, выполняет модуль обработки, а когда Nginx архивирует вывод или выполняет включение серверной стороны server-side, то это делается с помощью модуля фильтра. «Ядро» Nginx просто заботится о всех сетевых протоколах и протоколах приложения и устанавливает последовательность выполнения модулей, которые имеют право для обработки запроса. Децентрализованная архитектура позволяет Вам, сделать хороший автономный блок, который делает то, что вы хотите.
Заметка: В отличии от модулей Apache, модулю Nginx не связываются динамически. (Другими словами, они скомпилированы прямо в бинарник Nginx.)

Как модуль вызывается? Как правило, при запуске сервера, каждый обработчик handler получает возможность прикрепиться к конкретных местам, определенным в конфигурации, если более одного обработчика прикрепляется к кокретному месту, то только один "победит" (но хороший писатель конфигурации не позволит случится конфликту). Обработчики Handlers могут отреагировать тремя способами: "все хорошо", "была ошибка", или "отклонить чтобы обработать запрос и отложить обработчик handler по умолчанию (как правило, это нечто, что обслуживает статические файлы).
Если обработчик случается с обратным прокси-сервером для некоторых рабочих ролей, есть место для еще одного типа модуля: балансировки нагрузки. Балансировик нагрузки принимает запрос, передает его backend серверам и решает какой сервер получит запрос. Nginx поставляется с двумя модулями балансировщиков нагрузки: Круговой (round-robin), которая раздает запросы вида карт в начале игры в покер и "IP хэш" метод, который гарантирует, что конкретный клиент получит ответ от одного из внутренних серверов при нескольких запросах.
Если обработчик не вызывает ошибки, вызывается фильтр. Несколько фильтров можно подключить в каждом месте, так что (например) ответ может быть сжат и затем разбит. Порядок их выполнения определяется во время компиляции. Фильтры имеют классическую "цепочку ответственности" шаблон проектирования следующий: один фильтр вызывается, делает свою работу, а затем вызывает следующий фильтр, пока последний фильтр не будет вызван, и затем Nginx заканчивает подготовку ответа.
Действительно хорошая часть работы цепочки фильтров - это то, что каждый фильтр не ждет предыдущий фильтр до конца, он может обрабатывать вывод предыдущего фильтра, как это происходит при использовании Unix pipe. Фильтры работают с * буферами *, которые, как правило, имеют размер страницы (4Кб), хотя вы можете изменить это в вашем nginx.conf. Это означает, например, что модуль может начать сжатие ответа от внутреннего сервера и передавать его клиенту прежде чем модулем получит весь ответ от внутреннего сервера.. Здорово!
Таким образом, чтобы завершить обзор концепции, типичный цикл выглядит слудующим образом:

Клиент посылает HTTP запрос - Nginx выбирает подходящий обработчик, основываясь на location в конфигурации → (если применимо) балансировщик нагрузки выбирает внутренний сервер → обработчик делает свое дело и передает каждый выходной буфер для первого фильтра → Первый фильтр пропускает вывод на второй фильтр → второго на третье → третьей на четвертую и т.д. → окончательный ответ посылается клиенту.

Модуль Nginx *чрезвычайно* настраиваемый. Это возглает большую ответственность на разработчика модуля, он должен точно определить, как и когда модуль должен работать.
Вызов модуля фактически осуществляется через вызов серии обратных вызовов, а их много.
То есть, Вы должны понимать, что и когда функция должна выполнить:
-   Just before the server reads the config file
-   For every configuration directive for the location and server for which it appears;
-   When Nginx initializes the main configuration
-   When Nginx initializes the server (i.e., host/port) configuration
-   When Nginx merges the server configuration with the main configuration
-   When Nginx initializes the location configuration
-   When Nginx merges the location configuration with its parent server configuration
-   When Nginx's master process starts
-   When a new worker process starts
-   When a worker process exits
-   When the master exits
-   Handling a request
-   Filtering response headers
-   Filtering the response body
-   Picking a backend server
-   Initiating a request to a backend server
-   *Re*-initiating a request to a backend server
-   Processing the response from a backend server
-   Finishing an interaction with a backend server
- Непосредственно перед тем, как сервер считает конфигурационный файл
- Для каждой ли директивы в конфигурации location и server
- Когда Nginx инициализирует основную конфигурацию
- Когда Nginx инициализирует сервер (т.е. хост / порт) конфигурациию
- Когда Nginx объединяет конфигурацию сервера с основной конфигурацией
- Когда Nginx инициализирует конфигурацию location
- Когда Nginx объединяет location конфигурацию с его конфигурацией родительского сервера
- Когда мастер-процесс Nginx стартует
- Когда новый worker стартует
- Когда рабочий процесс завершается
- Когда мастер-процесс завершает выполнение
- Обработчики запроса
- Фильтрация заголовки ответа
- Фильтрация тела ответа
- Подбор внутреннего сервера
- Инициирование *запроса* для внутреннего сервера
- *Пере*-инициирование *запроса* для внутреннего сервера
- Обработка *ответа* от внутреннего сервера
- Окончательное взаимодействие с внутренним сервером

Holy mackerel! It's a bit overwhelming. You've got a lot of power at your disposal, but you can still do something useful using only a couple of these hooks and a couple of corresponding functions. Time to dive into some modules.
Святая скумбрия! Это выглядит немного удручающим. Вы должны очень хорошо постараться, чтобы получить результат, используя только пару хуков и вызвав пару соответствующих функций.
Ну чтож, не стоит хмуриться. Самое время, погрузиться в описание некоторых модулей.

2. Компоненты модуля Nginx
--------------------------

As I said, you have a *lot* of flexibility when it comes to making an Nginx module. This section will describe the parts that are almost always present. It's intended as a guide for understanding a module, and a reference for when you think you're ready to start writing a module.
Как я уже сказал, Вы должны обладать хорошим кунг-фу когда дело доходит до разработке модуля Nginx. Этот раздел описывает части, которые почти всегда присутствуют. Он предназначен в качестве руководства для понимания модуля - это ссылка, для тех кто думает, что готов к написанию модуля Nginx.

### 2.1. Структура конфигурации модуля)

Модули могут содержать до трех описаний в структуре конфигурации. В контексте: один для основного main, для сервера server и для расположения location. 
Для большинства модулей необходимо место в конфигурации. Называться они могут как `ngx_http_<module name>_(main|srv|loc)_conf_t`. Вот пример, взятый из модуля DAV:

```c
    typedef struct {
        ngx_uint_t  methods;
        ngx_flag_t  create_full_put_path;
        ngx_uint_t  access;
    } ngx_http_dav_loc_conf_t;
```

Обратите внимание, что Nginx имеет специальные типы данных (`ngx_uint_t` и `ngx_flag_t`);
это всего лишь псевдонимы для примитивных типов данных, которые вы знаете и любите :)
(см. [core/ngx\_config.h](http://lxr.evanmiller.org/http/source/core/ngx_config.h#L79) если Вам интересно).

Эелементы структуры конфигурации наполняются директивами модуля.

### 2.2. Директивы модуля

Директивы модуля доолжны быть описаны в статическом массиве `ngx_command_t`.
Вот пример того, как это объявляется, взятые из небольшого модуля, что я написал:

```c

    static ngx_command_t  ngx_http_circle_gif_commands[] = {
        { ngx_string("circle_gif"),
          NGX_HTTP_LOC_CONF|NGX_CONF_NOARGS,
          ngx_http_circle_gif,
          NGX_HTTP_LOC_CONF_OFFSET,
          0,
          NULL },

        { ngx_string("circle_gif_min_radius"),
          NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_TAKE1,
          ngx_conf_set_num_slot,
          NGX_HTTP_LOC_CONF_OFFSET,
          offsetof(ngx_http_circle_gif_loc_conf_t, min_radius),
          NULL },
          ...
          ngx_null_command
    };
```

В `ngx_command_t` описываем структуру с которой будет работать модуль, смотрите [core/ngx\_conf\_file.h](http://lxr.evanmiller.org/http/source/core/ngx_conf_file.h#L77):

```c

    struct ngx_command_t {
        ngx_str_t             name;
        ngx_uint_t            type;
        char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
        ngx_uint_t            conf;
        ngx_uint_t            offset;
        void                 *post;
    };
```

Кажется, что многовато, но каждый элемент имеет свою цель.

Имя директивы `name` строка без пробелов - тип её данных `ngx_str_t`. Например `ngx_str("proxy_pass")`.
Заметка: `ngx_str_t` структура  с элементами данных `data`, в виде строк, и длинной `len` этой строки. 
Nginx использует эту структуру данных - так что в большинстве мест можно ожидать, что это именно эта строка.

`type` is a set of flags that indicate where the directive is legal and how many arguments the directive takes. Applicable flags, which are bitwise-OR'd, are:

-   `NGX_HTTP_MAIN_CONF`: directive is valid in the main config
-   `NGX_HTTP_SRV_CONF`: directive is valid in the server (host) config
-   `NGX_HTTP_LOC_CONF`: directive is valid in a location config
-   `NGX_HTTP_UPS_CONF`: directive is valid in an upstream config

-   `NGX_CONF_NOARGS`: directive can take 0 arguments
-   `NGX_CONF_TAKE1`: directive can take exactly 1 argument
-   `NGX_CONF_TAKE2`: directive can take exactly 2 arguments
-   …
-   `NGX_CONF_TAKE7`: directive can take exactly 7 arguments

-   `NGX_CONF_FLAG`: directive takes a boolean ("on" or "off")
-   `NGX_CONF_1MORE`: directive must be passed at least one argument
-   `NGX_CONF_2MORE`: directive must be passed at least two arguments

There are a few other options, too, see [core/ngx\_conf\_file.h](http://lxr.evanmiller.org/http/source/core/ngx_conf_file.h#L1).

The `set` struct element is a pointer to a function for setting up part of the module's configuration; typically this function will translate the arguments passed to this directive and save an appropriate value in its configuration struct. This setup function will take three arguments:

1.  a pointer to an `ngx_conf_t` struct, which contains the arguments passed to the directive
2.  a pointer to the current `ngx_command_t` struct
3.  a pointer to the module's custom configuration struct

This setup function will be called when the directive is encountered. Nginx provides a number of functions for setting particular types of values in the custom configuration struct. These functions include:

-   `ngx_conf_set_flag_slot`: translates "on" or "off" to 1 or 0
-   `ngx_conf_set_str_slot`: saves a string as an `ngx_str_t`
-   `ngx_conf_set_num_slot`: parses a number and saves it to an `int`
-   `ngx_conf_set_size_slot`: parses a data size ("8k", "1m", etc.) and saves it to a `size_t`

There are several others, and they're quite handy (see [core/ngx\_conf\_file.h](http://lxr.evanmiller.org/http/source/core/ngx_conf_file.h#L329)). Modules can also put a reference to their own function here, if the built-ins aren't quite good enough.

How do these built-in functions know where to save the data? That's where the next two elements of `ngx_command_t` come in, `conf` and `offset`. `conf` tells Nginx whether this value will get saved to the module's main configuration, server configuration, or location configuration (with `NGX_HTTP_MAIN_CONF_OFFSET`, `NGX_HTTP_SRV_CONF_OFFSET`, or `NGX_HTTP_LOC_CONF_OFFSET`). `offset` then specifies which part of this configuration struct to write to.

*Finally*, `post` is just a pointer to other crap the module might need while it's reading the configuration. It's often `NULL`.

The commands array is terminated with `ngx_null_command` as the last element.

### 2.3. The Module Context

This is a static `ngx_http_module_t` struct, which just has a bunch of function references for creating the three configurations and merging them together. Its name is `ngx_http_<module name>_module_ctx`. In order, the function references are:

-   preconfiguration
-   postconfiguration
-   creating the main conf (i.e., do a malloc and set defaults)
-   initializing the main conf (i.e., override the defaults with what's in nginx.conf)
-   creating the server conf
-   merging it with the main conf
-   creating the location conf
-   merging it with the server conf

These take different arguments depending on what they're doing. Here's the struct definition, taken from [http/ngx\_http\_config.h](http://lxr.evanmiller.org/http/source/http/ngx_http_config.h#L22), so you can see the different function signatures of the callbacks:

``

    typedef struct {
        ngx_int_t   (*preconfiguration)(ngx_conf_t *cf);
        ngx_int_t   (*postconfiguration)(ngx_conf_t *cf);

        void       *(*create_main_conf)(ngx_conf_t *cf);
        char       *(*init_main_conf)(ngx_conf_t *cf, void *conf);

        void       *(*create_srv_conf)(ngx_conf_t *cf);
        char       *(*merge_srv_conf)(ngx_conf_t *cf, void *prev, void *conf);

        void       *(*create_loc_conf)(ngx_conf_t *cf);
        char       *(*merge_loc_conf)(ngx_conf_t *cf, void *prev, void *conf);
    } ngx_http_module_t;

You can set functions you don't need to `NULL`, and Nginx will figure it out.

Most handlers just use the last two: a function to allocate memory for location-specific configuration (called `ngx_http_<module name>_create_loc_conf`), and a function to set defaults and merge this configuration with any inherited configuration (called `ngx_http_<module name >_merge_loc_conf`). The merge function is also responsible for producing an error if the configuration is invalid; these errors halt server startup.

Here's an example module context struct:

``

    static ngx_http_module_t  ngx_http_circle_gif_module_ctx = {
        NULL,                          /* preconfiguration */
        NULL,                          /* postconfiguration */

        NULL,                          /* create main configuration */
        NULL,                          /* init main configuration */

        NULL,                          /* create server configuration */
        NULL,                          /* merge server configuration */

        ngx_http_circle_gif_create_loc_conf,  /* create location configuration */
        ngx_http_circle_gif_merge_loc_conf /* merge location configuration */
    };

Time to dig in deep a little bit. These configuration callbacks look quite similar across all modules and use the same parts of the Nginx API, so they're worth knowing about.

#### 2.3.1. create\_loc\_conf

Here's what a bare-bones create\_loc\_conf function looks like, taken from the circle\_gif module I wrote (see the [the source](/nginx/ngx_http_circle_gif_module.c.txt)). It takes a directive struct (`ngx_conf_t`) and returns a newly created module configuration struct (in this case `ngx_http_circle_gif_loc_conf_t`).

``

    static void *
    ngx_http_circle_gif_create_loc_conf(ngx_conf_t *cf)
    {
        ngx_http_circle_gif_loc_conf_t  *conf;

        conf = ngx_pcalloc(cf->pool, sizeof(ngx_http_circle_gif_loc_conf_t));
        if (conf == NULL) {
            return NGX_CONF_ERROR;
        }
        conf->min_radius = NGX_CONF_UNSET_UINT;
        conf->max_radius = NGX_CONF_UNSET_UINT;
        return conf;
    }

First thing to notice is Nginx's memory allocation; it takes care of the `free`'ing as long as the module uses `ngx_palloc` (a `malloc` wrapper) or `ngx_pcalloc` (a `calloc` wrapper).

The possible UNSET constants are `NGX_CONF_UNSET_UINT`, `NGX_CONF_UNSET_PTR`, `NGX_CONF_UNSET_SIZE`, `NGX_CONF_UNSET_MSEC`, and the catch-all `NGX_CONF_UNSET`. UNSET tell the merging function that the value should be overridden.

#### 2.3.2. merge\_loc\_conf

Here's the merging function used in the circle\_gif module:

``

    static char *
    ngx_http_circle_gif_merge_loc_conf(ngx_conf_t *cf, void *parent, void *child)
    {
        ngx_http_circle_gif_loc_conf_t *prev = parent;
        ngx_http_circle_gif_loc_conf_t *conf = child;

        ngx_conf_merge_uint_value(conf->min_radius, prev->min_radius, 10);
        ngx_conf_merge_uint_value(conf->max_radius, prev->max_radius, 20);

        if (conf->min_radius < 1) {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, 
                "min_radius must be equal or more than 1");
            return NGX_CONF_ERROR;
        }
        if (conf->max_radius < conf->min_radius) {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, 
                "max_radius must be equal or more than min_radius");
            return NGX_CONF_ERROR;
        }

        return NGX_CONF_OK;
    }

Notice first that Nginx provides nice merging functions for different data types (`ngx_conf_merge_<data type>_value`); the arguments are

1.  *this* location's value
2.  the value to inherit if \#1 is not set
3.  the default if neither \#1 nor \#2 is set

The result is then stored in the first argument. Available merge functions include `ngx_conf_merge_size_value`, `ngx_conf_merge_msec_value`, and others. See [core/ngx\_conf\_file.h](http://lxr.evanmiller.org/http/source/core/ngx_conf_file.h#L254) for a full list.

Trivia question: How do these functions write to the first argument, since the first argument is passed in by value?

Answer: these functions are defined by the preprocessor (so they expand to a few "if" statements and assignments before reaching the compiler).

Notice also how errors are produced; the function writes something to the log file, and returns `NGX_CONF_ERROR`. That return code halts server startup. (Since the message is logged at level `NGX_LOG_EMERG`, the message will also go to standard out; FYI, [core/ngx\_log.h](http://lxr.evanmiller.org/http/source/core/ngx_log.h#L1) has a list of log levels.)

### 2.4. The Module Definition

Next we add one more layer of indirection, the `ngx_module_t` struct. The variable is called `ngx_http_<module name>_module`. This is where references to the context and directives go, as well as the remaining callbacks (exit thread, exit process, etc.). The module definition is sometimes used as a key to look up data associated with a particular module. The module definition usually looks like this:

``

    ngx_module_t  ngx_http_<module name>_module = {
        NGX_MODULE_V1,
        &ngx_http_<module name>_module_ctx, /* module context */
        ngx_http_<module name>_commands,   /* module directives */
        NGX_HTTP_MODULE,               /* module type */
        NULL,                          /* init master */
        NULL,                          /* init module */
        NULL,                          /* init process */
        NULL,                          /* init thread */
        NULL,                          /* exit thread */
        NULL,                          /* exit process */
        NULL,                          /* exit master */
        NGX_MODULE_V1_PADDING
    };

…substituting \<module name\> appropriately. Modules can add callbacks for process/thread creation and death, but most modules keep things simple. (For the arguments passed to each callback, see [core/ngx\_conf\_file.h](http://lxr.evanmiller.org/http/source/core/ngx_conf_file.h#L110).)

### 2.5. Module Installation

The proper way to install a module depends on whether the module is a handler, filter, or load-balancer; so the details are reserved for those respective sections.

3. Handlers
-----------

Now we'll put some trivial modules under the microscope to see how they work.

### 3.1. Anatomy of a Handler (Non-proxying)

Handlers typically do four things: get the location configuration, generate an appropriate response, send the header, and send the body. A handler has one argument, the request struct. A request struct has a lot of useful information about the client request, such as the request method, URI, and headers. We'll go over these steps one by one.

#### 3.1.1. Getting the location configuration

This part's easy. All you need to do is call `ngx_http_get_module_loc_conf` and pass in the current request struct and the module definition. Here's the relevant part of my circle gif handler:

``

    static ngx_int_t
    ngx_http_circle_gif_handler(ngx_http_request_t *r)
    {
        ngx_http_circle_gif_loc_conf_t  *circle_gif_config;
        circle_gif_config = ngx_http_get_module_loc_conf(r, ngx_http_circle_gif_module);
        ...

Now I've got access to all the variables that I set up in my merge function.

#### 3.1.2. Generating a response

This is the interesting part where modules actually do work.

The request struct will be helpful here, particularly these elements:

``

    typedef struct {
    ...
    /* the memory pool, used in the ngx_palloc functions */
        ngx_pool_t                       *pool; 
        ngx_str_t                         uri;
        ngx_str_t                         args;
        ngx_http_headers_in_t             headers_in;

    ...
    } ngx_http_request_t;

`uri` is the path of the request, e.g. "/query.cgi".

`args` is the part of the request after the question mark (e.g. "name=john").

`headers_in` has a lot of useful stuff, such as cookies and browser information, but many modules don't need anything from it. See [http/ngx\_http\_request.h](http://lxr.evanmiller.org/http/source/http/ngx_http_request.h#L158) if you're interested.

This should be enough information to produce some useful output. The full `ngx_http_request_t` struct can be found in [http/ngx\_http\_request.h](http://lxr.evanmiller.org/http/source/http/ngx_http_request.h#L316).

#### 3.1.3. Sending the header

The response headers live in a struct called `headers_out` referenced by the request struct. The handler sets the ones it wants and then calls `ngx_http_send_header(r)`. Some useful parts of `headers_out` include:

``

    typedef stuct {
    ...
        ngx_uint_t                        status;
        size_t                            content_type_len;
        ngx_str_t                         content_type;
        ngx_table_elt_t                  *content_encoding;
        off_t                             content_length_n;
        time_t                            date_time;
        time_t                            last_modified_time;
    ..
    } ngx_http_headers_out_t;

(The rest can be found in [http/ngx\_http\_request.h](http://lxr.evanmiller.org/http/source/http/ngx_http_request.h#L220).)

So for example, if a module were to set the Content-Type to "image/gif", Content-Length to 100, and return a 200 OK response, this code would do the trick:

``

        r->headers_out.status = NGX_HTTP_OK;
        r->headers_out.content_length_n = 100;
        r->headers_out.content_type.len = sizeof("image/gif") - 1;
        r->headers_out.content_type.data = (u_char *) "image/gif";
        ngx_http_send_header(r);

Most legal HTTP headers are available (somewhere) for your setting pleasure. However, some headers are a bit trickier to set than the ones you see above; for example, `content_encoding` has type `(ngx_table_elt_t*)`, so the module must allocate memory for it. This is done with a function called `ngx_list_push`, which takes in an `ngx_list_t` (similar to an array) and returns a reference to a newly created member of the list (of type `ngx_table_elt_t`). The following code sets the Content-Encoding to "deflate" and sends the header:

``

        r->headers_out.content_encoding = ngx_list_push(&r->headers_out.headers);
        if (r->headers_out.content_encoding == NULL) {
            return NGX_ERROR;
        }
        r->headers_out.content_encoding->hash = 1;
        r->headers_out.content_encoding->key.len = sizeof("Content-Encoding") - 1;
        r->headers_out.content_encoding->key.data = (u_char *) "Content-Encoding";
        r->headers_out.content_encoding->value.len = sizeof("deflate") - 1;
        r->headers_out.content_encoding->value.data = (u_char *) "deflate";
        ngx_http_send_header(r);

This mechanism is usually used when a header can have multiple values simultaneously; it (theoretically) makes it easier for filter modules to add and delete certain values while preserving others, because they don't have to resort to string manipulation.

#### 3.1.4. Sending the body

Now that the module has generated a response and put it in memory, it needs to assign the response to a special buffer, and then assign the buffer to a *chain link*, and *then* call the "send body" function on the chain link.

What are the chain links for? Nginx lets handler modules generate (and filter modules process) responses one buffer at a time; each chain link keeps a pointer to the next link in the chain, or `NULL` if it's the last one. We'll keep it simple and assume there is just one buffer.

First, a module will declare the buffer and the chain link:

``

        ngx_buf_t    *b;
        ngx_chain_t   out;

The next step is to allocate the buffer and point our response data to it:

``

        b = ngx_pcalloc(r->pool, sizeof(ngx_buf_t));
        if (b == NULL) {
            ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, 
                "Failed to allocate response buffer.");
            return NGX_HTTP_INTERNAL_SERVER_ERROR;
        }

        b->pos = some_bytes; /* first position in memory of the data */
        b->last = some_bytes + some_bytes_length; /* last position */

        b->memory = 1; /* content is in read-only memory */
        /* (i.e., filters should copy it rather than rewrite in place) */

        b->last_buf = 1; /* there will be no more buffers in the request */

Now the module attaches it to the chain link:

``

        out.buf = b;
        out.next = NULL;

FINALLY, we send the body, and return the status code of the output filter chain all in one go:

``

        return ngx_http_output_filter(r, &out);

Buffer chains are a critical part of Nginx's IO model, so you should be comfortable with how they work.

Trivia question: Why does the buffer have the `last_buf` variable, when we can tell we're at the end of a chain by checking "next" for `NULL`?

Answer: A chain might be incomplete, i.e., have multiple buffers, but not all the buffers in this request or response. So some buffers are at the end of the chain but not the end of a request. This brings us to…

### 3.2. Anatomy of an Upstream (a.k.a Proxy) Handler

I waved my hands a bit about having your handler generate a response. Sometimes you'll be able to get that response just with a chunk of C code, but often you'll want to talk to another server (for example, if you're writing a module to implement another network protocol). You *could* do all of the network programming yourself, but what happens if you receive a partial response? You don't want to block the primary event loop with your own event loop while you're waiting for the rest of the response. You'd kill the Nginx's performance. Fortunately, Nginx lets you hook right into its own mechanisms for dealing with back-end servers (called "upstreams"), so your module can talk to another server without getting in the way of other requests. This section describes how a module talks to an upstream, such as Memcached, FastCGI, or another HTTP server.

#### 3.2.1. Summary of upstream callbacks

Unlike the handler function for other modules, the handler function of an upstream module does little "real work". It does *not* call `ngx_http_output_filter`. It merely sets callbacks that will be invoked when the upstream server is ready to be written to and read from. There are actually 6 available hooks:

`create_request` crafts a request buffer (or chain of them) to be sent to the upstream

`reinit_request` is called if the connection to the back-end is reset (just before `create_request` is called for the second time)

`process_header` processes the first bit of the upstream's response, and usually saves a pointer to the upstream's "payload"

`abort_request` is called if the client aborts the request

`finalize_request` is called when Nginx is finished reading from the upstream

`input_filter` is a body filter that can be called on the response body (e.g., to remove a trailer)

How do these get attached? An example is in order. Here's a simplified version of the proxy module's handler:

``

    static ngx_int_t
    ngx_http_proxy_handler(ngx_http_request_t *r)
    {
        ngx_int_t                   rc;
        ngx_http_upstream_t        *u;
        ngx_http_proxy_loc_conf_t  *plcf;

        plcf = ngx_http_get_module_loc_conf(r, ngx_http_proxy_module);

    /* set up our upstream struct */
        u = ngx_pcalloc(r->pool, sizeof(ngx_http_upstream_t));
        if (u == NULL) {
            return NGX_HTTP_INTERNAL_SERVER_ERROR;
        }

        u->peer.log = r->connection->log;
        u->peer.log_error = NGX_ERROR_ERR;

        u->output.tag = (ngx_buf_tag_t) &ngx_http_proxy_module;

        u->conf = &plcf->upstream;

    /* attach the callback functions */
        u->create_request = ngx_http_proxy_create_request;
        u->reinit_request = ngx_http_proxy_reinit_request;
        u->process_header = ngx_http_proxy_process_status_line;
        u->abort_request = ngx_http_proxy_abort_request;
        u->finalize_request = ngx_http_proxy_finalize_request;

        r->upstream = u;

        rc = ngx_http_read_client_request_body(r, ngx_http_upstream_init);

        if (rc >= NGX_HTTP_SPECIAL_RESPONSE) {
            return rc;
        }

        return NGX_DONE;
    }

It does a bit of housekeeping, but the important parts are the callbacks. Also notice the bit about `ngx_http_read_client_request_body`. That's setting another callback for when Nginx has finished reading from the client.

What will each of these callbacks do? Usually, `reinit_request`, `abort_request`, and `finalize_request` will set or reset some sort of internal state and are only a few lines long. The real workhorses are `create_request` and `process_header`.

#### 3.2.2. The create\_request callback

For the sake of simplicity, let's suppose I have an upstream server that reads in one character and prints out two characters. What would my functions look like?

The `create_request` needs to allocate a buffer for the single-character request, allocate a chain link for that buffer, and then point the upstream struct to that chain link. It would look like this:

``

    static ngx_int_t
    ngx_http_character_server_create_request(ngx_http_request_t *r)
    {
    /* make a buffer and chain */
        ngx_buf_t *b;
        ngx_chain_t *cl;

        b = ngx_create_temp_buf(r->pool, sizeof("a") - 1);
        if (b == NULL)
            return NGX_ERROR;

        cl = ngx_alloc_chain_link(r->pool);
        if (cl == NULL)
            return NGX_ERROR;

    /* hook the buffer to the chain */
        cl->buf = b;
    /* chain to the upstream */
        r->upstream->request_bufs = cl;

    /* now write to the buffer */
        b->pos = "a";
        b->last = b->pos + sizeof("a") - 1;

        return NGX_OK;
    }

That wasn't so bad, was it? Of course, in reality you'll probably want to use the request URI in some meaningful way. It's available as an `ngx_str_t` in `r->uri`, and the GET paramaters are in `r->args`, and don't forget you also have access to the request headers and cookies.

#### 3.2.3. The process\_header callback

Now it's time for the `process_header`. Just as `create_request` added a pointer to the request body, `process_header` *shifts the response pointer to the part that the client will receive*. It also reads in the header from the upstream and sets the client response headers accordingly.

Here's a bare-minimum example, reading in that two-character response. Let's suppose the first character is the "status" character. If it's a question mark, we want to return a 404 File Not Found to the client and disregard the other character. If it's a space, then we want to return the other character to the client along with a 200 OK response. All right, it's not the most useful protocol, but it's a good demonstration. How would we write this `process_header` function?

``

    static ngx_int_t
    ngx_http_character_server_process_header(ngx_http_request_t *r)
    {
        ngx_http_upstream_t       *u;
        u = r->upstream;

        /* read the first character */
        switch(u->buffer.pos[0]) {
            case '?':
                r->header_only; /* suppress this buffer from the client */
                u->headers_in.status_n = 404;
                break;
            case ' ':
                u->buffer.pos++; /* move the buffer to point to the next character */
                u->headers_in.status_n = 200;
                break;
        }

        return NGX_OK;
    }

That's it. Manipulate the header, change the pointer, it's done. Notice that `headers_in` is actually a response header struct like we've seen before (cf. [http/ngx\_http\_request.h](http://lxr.evanmiller.org/http/source/http/ngx_http_request.h#L158)), but it can be populated with the headers from the upstream. A real proxying module will do a lot more header processing, not to mention error handling, but you get the main idea.

But.. what if we don't have the whole header from the upstream in one buffer?

#### 3.2.4. Keeping state

Well, remember how I said that `abort_request`, `reinit_request`, and `finalize_request` could be used for resetting internal state? That's because many upstream modules *have* internal state. The module will need to define a *custom context struct* to keep track of what it has read so far from an upstream. This is NOT the same as the "Module Context" referred to above. That's of a pre-defined type, whereas the custom context can have whatever elements and data you need (it's your struct). This context struct should be instantiated inside the `create_request` function, perhaps like this:

``

        ngx_http_character_server_ctx_t   *p;   /* my custom context struct */

        p = ngx_pcalloc(r->pool, sizeof(ngx_http_character_server_ctx_t));
        if (p == NULL) {
            return NGX_HTTP_INTERNAL_SERVER_ERROR;
        }

        ngx_http_set_ctx(r, p, ngx_http_character_server_module);

That last line essentially registers the custom context struct with a particular request and module name for easy retrieval later. Whenever you need this context struct (probably in all the other callbacks), just do:

``

        ngx_http_proxy_ctx_t  *p;
        p = ngx_http_get_module_ctx(r, ngx_http_proxy_module);

And `p` will have the current state. Set it, reset it, increment, decrement, shove arbitrary data in there, whatever you want. This is a great way to use a persistent state machine when reading from an upstream that returns data in chunks, again without blocking the primary event loop. Nice!

### 3.3. Handler Installation

Handlers are installed by adding code to the callback of the directive that enables the module. For example, my circle gif `ngx_command_t` looks like this:

``

        { ngx_string("circle_gif"),
          NGX_HTTP_LOC_CONF|NGX_CONF_NOARGS,
          ngx_http_circle_gif,
          0,
          0,
          NULL }

The callback is the third element, in this case `ngx_http_circle_gif`. Recall that the arguments to this callback are the directive struct (`ngx_conf_t`, which holds the user's arguments), the relevant `ngx_command_t` struct, and a pointer to the module's custom configuration struct. For my circle gif module, the function looks like:

``

    static char *
    ngx_http_circle_gif(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
    {
        ngx_http_core_loc_conf_t  *clcf;

        clcf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
        clcf->handler = ngx_http_circle_gif_handler;

        return NGX_CONF_OK;
    }

There are two steps here: first, get the "core" struct for this location, then assign a handler to it. Pretty simple, eh?

I've said all I know about handler modules. It's time to move onto filter modules, the components in the output filter chain.

4. Filters
----------

Filters manipulate responses generated by handlers. Header filters manipulate the HTTP headers, and body filters manipulate the response content.

### 4.1. Anatomy of a Header Filter

A header filter consists of three basic steps:

1.  Decide whether to operate on this response
2.  Operate on the response
3.  Call the next filter

To take an example, here's a simplified version of the "not modified" header filter, which sets the status to 304 Not Modified if the client's If-Modified-Since header matches the response's Last-Modified header. Note that header filters take in the `ngx_http_request_t` struct as the only argument, which gets us access to both the client headers and soon-to-be-sent response headers.

``

    static
    ngx_int_t ngx_http_not_modified_header_filter(ngx_http_request_t *r)
    {
        time_t  if_modified_since;

        if_modified_since = ngx_http_parse_time(r->headers_in.if_modified_since->value.data,
                                  r->headers_in.if_modified_since->value.len);

    /* step 1: decide whether to operate */
        if (if_modified_since != NGX_ERROR && 
            if_modified_since == r->headers_out.last_modified_time) {

    /* step 2: operate on the header */
            r->headers_out.status = NGX_HTTP_NOT_MODIFIED;
            r->headers_out.content_type.len = 0;
            ngx_http_clear_content_length(r);
            ngx_http_clear_accept_ranges(r);
        }

    /* step 3: call the next filter */
        return ngx_http_next_header_filter(r);
    }

The `headers_out` structure is just the same as we saw in the section about handlers (cf. [http/ngx\_http\_request.h](http://lxr.evanmiller.org/http/source/http/ngx_http_request.h#L220)), and can be manipulated to no end.

### 4.2. Anatomy of a Body Filter

The buffer chain makes it a little tricky to write a body filter, because the body filter can only operate on one buffer (chain link) at a time. The module must decide whether to *overwrite* the input buffer, *replace* the buffer with a newly allocated buffer, or *insert* a new buffer before or after the buffer in question. To complicate things, sometimes a module will receive several buffers so that it has an *incomplete buffer chain* that it must operate on. Unfortunately, Nginx does not provide a high-level API for manipulating the buffer chain, so body filters can be difficult to understand (and to write). But, here are some operations you might see in action.

A body filter's prototype might look like this (example taken from the "chunked" filter in the Nginx source):

``

    static ngx_int_t ngx_http_chunked_body_filter(ngx_http_request_t *r, ngx_chain_t *in);

The first argument is our old friend the request struct. The second argument is a pointer to the head of the current partial chain (which could contain 0, 1, or more buffers).

Let's take a simple example. Suppose we want to insert the text "\<l!-- Served by Nginx --\>" to the end of every request. First, we need to figure out if the response's final buffer is included in the buffer chain we were given. Like I said, there's not a fancy API, so we'll be rolling our own for loop:

``

        ngx_chain_t *chain_link;
        int chain_contains_last_buffer = 0;

        chain_link = in;
        for ( ; ; ) {
            if (chain_link->buf->last_buf)
                chain_contains_last_buffer = 1;
            if (chain_link->next == NULL)
                break;
            chain_link = chain_link->next;
        }

Now let's bail out if we don't have that last buffer:

``

        if (!chain_contains_last_buffer)
            return ngx_http_next_body_filter(r, in);

Super, now the last buffer is stored in chain\_link. Now we allocate a new buffer:

``

        ngx_buf_t    *b;
        b = ngx_calloc_buf(r->pool);
        if (b == NULL) {
            return NGX_ERROR;
        }

And put some data in it:

``

        b->pos = (u_char *) "<!-- Served by Nginx -->";
        b->last = b->pos + sizeof("<!-- Served by Nginx -->") - 1;

And hook the buffer into a new chain link:

``

        ngx_chain_t   *added_link;

        added_link = ngx_alloc_chain_link(r->pool);
        if (added_link == NULL)
            return NGX_ERROR;

        added_link->buf = b;
        added_link->next = NULL;

Finally, hook the new chain link to the final chain link we found before:

``

        chain_link->next = added_link;

And reset the "last\_buf" variables to reflect reality:

``

        chain_link->buf->last_buf = 0;
        added_link->buf->last_buf = 1;

And pass along the modified chain to the next output filter:

``

        return ngx_http_next_body_filter(r, in);

The resulting function takes much more effort than what you'd do with, say, mod\_perl (`$response->body =~ s/$/<!-- Served by mod_perl -->/`), but the buffer chain is a very powerful construct, allowing programmers to process data incrementally so that the client gets something as soon as possible. However, in my opinion, the buffer chain desperately needs a cleaner interface so that programmers can't leave the chain in an inconsistent state. For now, manipulate it at your own risk.

### 4.3. Filter Installation

Filters are installed in the post-configuration step. We install both header filters and body filters in the same place.

Let's take a look at the chunked filter module for a simple example. Its module context looks like this:

``

    static ngx_http_module_t  ngx_http_chunked_filter_module_ctx = {
        NULL,                                  /* preconfiguration */
        ngx_http_chunked_filter_init,          /* postconfiguration */
      ...
    };

Here's what happens in `ngx_http_chunked_filter_init`: ``

    static ngx_int_t
    ngx_http_chunked_filter_init(ngx_conf_t *cf)
    {
        ngx_http_next_header_filter = ngx_http_top_header_filter;
        ngx_http_top_header_filter = ngx_http_chunked_header_filter;

        ngx_http_next_body_filter = ngx_http_top_body_filter;
        ngx_http_top_body_filter = ngx_http_chunked_body_filter;

        return NGX_OK;
    }

What's going on here? Well, if you remember, filters are set up with a CHAIN OF RESPONSIBILITY. When a handler generates a response, it calls two functions: `ngx_http_output_filter`, which calls the global function reference `ngx_http_top_body_filter`; and `ngx_http_send_header`, which calls the global function reference `ngx_http_top_header_filter`.

`ngx_http_top_body_filter` and `ngx_http_top_header_filter` are the respective "heads" of the body and header filter chains. Each "link" on the chain keeps a function reference to the next link in the chain (the references are called `ngx_http_next_body_filter` and `ngx_http_next_header_filter`). When a filter is finished executing, it just calls the next filter, until a specially defined "write" filter is called, which wraps up the HTTP response. What you see in this filter\_init function is the module adding itself to the filter chains; it keeps a reference to the old "top" filters in its own "next" variables and declares *its* functions to be the new "top" filters. (Thus, the last filter to be installed is the first to be executed.)

Side note: how does this work exactly?

Each filter either returns an error code or uses this as the return statement:

`return ngx_http_next_body_filter();`

Thus, if the filter chain reaches the (specially-defined) end of the chain, an "OK" response is returned, but if there's an error along the way, the chain is cut short and Nginx serves up the appropriate error message. It's a singly-linked list with fast failures implemented solely with function references. Brilliant.

5. Load-Balancers
-----------------

A load-balancer is just a way to decide which backend server will receive a particular request; implementations exist for distributing requests in round-robin fashion or hashing some information about the request. This section will describe both a load-balancer's installation and its invocation, using the upstream\_hash module ([full source](/nginx/ngx_http_upstream_hash_module.c.txt)) as an example. upstream\_hash chooses a backend by hashing a variable specified in nginx.conf.

A load-balancing module has six pieces:

1.  The enabling configuration directive (e.g, `hash;`) will call a *registration function*
2.  The registration function will define the legal `server` options (e.g., `weight=`) and register an *upstream initialization function*
3.  The upstream initialization function is called just after the configuration is validated, and it:
    -   resolves the `server` names to particular IP addresses
    -   allocates space for sockets
    -   sets a callback to the *peer initialization function*

4.  the peer initialization function, called once per request, sets up data structures that the *load-balancing function* will access and manipulate;
5.  the load-balancing function decides where to route requests; it is called at least once per client request (more, if a backend request fails). This is where the interesting stuff happens.
6.  and finally, the *peer release function* can update statistics after communication with a particular backend server has finished (whether successfully or not)

It's a lot, but I'll break it down into pieces.

### 5.1. The enabling directive

Directive declarations, recall, specify both where they're valid and a function to call when they're encountered. A directive for a load-balancer should have the `NGX_HTTP_UPS_CONF` flag set, so that Nginx knows this directive is only valid inside an `upstream` block. It should provide a pointer to a *registration function*. Here's the directive declaration from the upstream\_hash module:

``

        { ngx_string("hash"),
          NGX_HTTP_UPS_CONF|NGX_CONF_NOARGS,
          ngx_http_upstream_hash,
          0,
          0,
          NULL },

Nothing new there.

### 5.2. The registration function

The callback `ngx_http_upstream_hash` above is the registration function, so named (by me) because it registers an *upstream initialization function* with the surrounding `upstream` configuration. In addition, the registration function defines which options to the `server` directive are legal inside this particular `upstream` block (e.g., `weight=`, `fail_timeout=`). Here's the registration function of the upstream\_hash module:

``

    ngx_http_upstream_hash(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
     {
        ngx_http_upstream_srv_conf_t  *uscf;
        ngx_http_script_compile_t      sc;
        ngx_str_t                     *value;
        ngx_array_t                   *vars_lengths, *vars_values;

        value = cf->args->elts;

        /* the following is necessary to evaluate the argument to "hash" as a $variable */
        ngx_memzero(&sc, sizeof(ngx_http_script_compile_t));

        vars_lengths = NULL;
        vars_values = NULL;

        sc.cf = cf;
        sc.source = &value[1];
        sc.lengths = &vars_lengths;
        sc.values = &vars_values;
        sc.complete_lengths = 1;
        sc.complete_values = 1;

        if (ngx_http_script_compile(&sc) != NGX_OK) {
            return NGX_CONF_ERROR;
        }
        /* end of $variable stuff */

        uscf = ngx_http_conf_get_module_srv_conf(cf, ngx_http_upstream_module);

        /* the upstream initialization function */
        uscf->peer.init_upstream = ngx_http_upstream_init_hash;

        uscf->flags = NGX_HTTP_UPSTREAM_CREATE;

        /* OK, more $variable stuff */
        uscf->values = vars_values->elts;
        uscf->lengths = vars_lengths->elts;

        /* set a default value for "hash_method" */
        if (uscf->hash_function == NULL) {
            uscf->hash_function = ngx_hash_key;
        }

        return NGX_CONF_OK;
     }

Aside from jumping through hoops so we can evaluation `$variable` later, it's pretty straightforward; assign a callback, set some flags. What flags are available?

-   `NGX_HTTP_UPSTREAM_CREATE`: let there be `server` directives in this upstream block. I can't think of a situation where you wouldn't use this.
-   `NGX_HTTP_UPSTREAM_WEIGHT`: let the `server` directives take a `weight=` option
-   `NGX_HTTP_UPSTREAM_MAX_FAILS`: allow the `max_fails=` option
-   `NGX_HTTP_UPSTREAM_FAIL_TIMEOUT`: allow the `fail_timeout=` option
-   `NGX_HTTP_UPSTREAM_DOWN`: allow the `down` option
-   `NGX_HTTP_UPSTREAM_BACKUP`: allow the `backup` option

Each module will have access to these configuration values. *It's up to the module to decide what to do with them.* That is, `max_fails` will not be automatically enforced; all the failure logic is up to the module author. More on that later. For now, we still haven't finished followed the trail of callbacks. Next up, we have the upstream initialization function (the `init_upstream` callback in the previous function).

### 5.3. The upstream initialization function

The purpose of the upstream initialization function is to resolve the host names, allocate space for sockets, and assign (yet another) callback. Here's how upstream\_hash does it:

``

    ngx_int_t
    ngx_http_upstream_init_hash(ngx_conf_t *cf, ngx_http_upstream_srv_conf_t *us)
    {
        ngx_uint_t                       i, j, n;
        ngx_http_upstream_server_t      *server;
        ngx_http_upstream_hash_peers_t  *peers;

        /* set the callback */
        us->peer.init = ngx_http_upstream_init_upstream_hash_peer;

        if (!us->servers) {
            return NGX_ERROR;
        }

        server = us->servers->elts;

        /* figure out how many IP addresses are in this upstream block. */
        /* remember a domain name can resolve to multiple IP addresses. */
        for (n = 0, i = 0; i < us->servers->nelts; i++) {
            n += server[i].naddrs;
        }

        /* allocate space for sockets, etc */
        peers = ngx_pcalloc(cf->pool, sizeof(ngx_http_upstream_hash_peers_t)
                + sizeof(ngx_peer_addr_t) * (n - 1));

        if (peers == NULL) {
            return NGX_ERROR;
        }

        peers->number = n;

        /* one port/IP address per peer */
        for (n = 0, i = 0; i < us->servers->nelts; i++) {
            for (j = 0; j < server[i].naddrs; j++, n++) {
                peers->peer[n].sockaddr = server[i].addrs[j].sockaddr;
                peers->peer[n].socklen = server[i].addrs[j].socklen;
                peers->peer[n].name = server[i].addrs[j].name;
            }
        }

        /* save a pointer to our peers for later */
        us->peer.data = peers;

        return NGX_OK;
    }

This function is a bit more involved than one might hope. Most of the work seems like it should be abstracted, but it's not, so that's what we live with. One strategy for simplifying things is to call the upstream initialization function of another module, have it do all the dirty work (peer allocation, etc), and then override the `us->peer.init` callback afterwards. For an example, see [http/modules/ngx\_http\_upstream\_ip\_hash\_module.c](http://lxr.evanmiller.org/http/source/http/modules/ngx_http_upstream_ip_hash_module.c#L80).

The important bit from our point of view is setting a pointer to the *peer initialization function*, in this case `ngx_http_upstream_init_upstream_hash_peer`.

### 5.4. The peer initialization function

The peer initialization function is called once per request. It sets up a data structure that the module will use as it tries to find an appropriate backend server to service that request; this structure is persistent across backend re-tries, so it's a convenient place to keep track of the number of connection failures, or a computed hash value. By convention, this struct is called `ngx_http_upstream_<module name>_peer_data_t`.

In addition, the peer initalization function sets up two callbacks:

-   `get`: the load-balancing function
-   `free`: the peer release function (usually just updates some statistics when a connection finishes)

As if that weren't enough, it also initalizes a variable called `tries`. As long as `tries` is positive, nginx will keep retrying this load-balancer. When `tries` is zero, nginx will give up. It's up to the `get` and `free` functions to set `tries` appropriately.

Here's a peer initialization function from the upstream\_hash module:

``

    static ngx_int_t
    ngx_http_upstream_init_hash_peer(ngx_http_request_t *r,
        ngx_http_upstream_srv_conf_t *us)
    {
        ngx_http_upstream_hash_peer_data_t     *uhpd;
        
        ngx_str_t val;

        /* evaluate the argument to "hash" */
        if (ngx_http_script_run(r, &val, us->lengths, 0, us->values) == NULL) {
            return NGX_ERROR;
        }

        /* data persistent through the request */
        uhpd = ngx_pcalloc(r->pool, sizeof(ngx_http_upstream_hash_peer_data_t)
            + sizeof(uintptr_t) 
              * ((ngx_http_upstream_hash_peers_t *)us->peer.data)->number 
                      / (8 * sizeof(uintptr_t)));
        if (uhpd == NULL) {
            return NGX_ERROR;
        }

        /* save our struct for later */
        r->upstream->peer.data = uhpd;

        uhpd->peers = us->peer.data;

        /* set the callbacks and initialize "tries" to "hash_again" + 1*/
        r->upstream->peer.free = ngx_http_upstream_free_hash_peer;
        r->upstream->peer.get = ngx_http_upstream_get_hash_peer;
        r->upstream->peer.tries = us->retries + 1;

        /* do the hash and save the result */
        uhpd->hash = us->hash_function(val.data, val.len);

        return NGX_OK;
    }

That wasn't so bad. Now we're ready to pick an upstream server.

### 5.5. The load-balancing function

It's time for the main course. The real meat and potatoes. This is where the module picks an upstream. The load-balancing function's prototype looks like:

``

    static ngx_int_t 
    ngx_http_upstream_get_<module_name>_peer(ngx_peer_connection_t *pc, void *data);

`data` is our struct of useful information concerning this client connection. `pc` will have information about the server we're going to connect to. The job of the load-balancing function is to fill in values for `pc->sockaddr`, `pc->socklen`, and `pc->name`. If you know some network programming, then those variable names might be familiar; but they're actually not very important to the task at hand. We don't care what they stand for; we just want to know where to find appropriate values to fill them.

This function must find a list of available servers, choose one, and assign its values to `pc`. Let's look at how upstream\_hash does it.

upstream\_hash previously stashed the server list into the `ngx_http_upstream_hash_peer_data_t` struct back in the call to `ngx_http_upstream_init_hash` (above). This struct is now available as `data`:

``

        ngx_http_upstream_hash_peer_data_t *uhpd = data;

The list of peers is now stored in `uhpd->peers->peer`. Let's pick a peer from this array by dividing the computed hash value by the number of servers:

``

        ngx_peer_addr_t *peer = &uhpd->peers->peer[uhpd->hash % uhpd->peers->number];

Now for the grand finale:

``

        pc->sockaddr = peer->sockaddr;
        pc->socklen  = peer->socklen;
        pc->name     = &peer->name;

        return NGX_OK;

That's all! If the load-balancer returns `NGX_OK`, it means, "go ahead and try this server". If it returns `NGX_BUSY`, it means all the backend hosts are unavailable, and Nginx should try again.

But… how do we keep track of what's unavailable? And what if we don't want it to try again?

### 5.6. The peer release function

The peer release function operates after an upstream connection takes place; its purpose is to track failures. Here is its function prototype:

``

    void 
    ngx_http_upstream_free_<module name>_peer(ngx_peer_connection_t *pc, void *data, 
        ngx_uint_t state);

The first two parameters are just the same as we saw in the load-balancer function. The third parameter is a `state` variable, which indicates whether the connection was successful. It may contain two values bitwise OR'd together: `NGX_PEER_FAILED` (the connection failed) and `NGX_PEER_NEXT` (either the connection failed, or it succeeded but the application returned an error). Zero means the connection succeeded.

It's up to the module author to decide what to do about these failure events. If they are to be used at all, the results should be stored in `data`, a pointer to the custom per-request data struct.

But the crucial purpose of the peer release function is to set `pc->tries` to zero if you don't want Nginx to keep trying this load-balancer during this request. The simplest peer release function would look like this:

``

        pc->tries = 0;

That would ensure that if there's ever an error reaching a backend server, a 502 Bad Proxy error will be returned to the client.

Here's a more complicated example, taken from the upstream\_hash module. If a backend connection fails, it marks it as failed in a bit-vector (called `tried`, an array of type `uintptr_t`), then keeps choosing a new backend until it finds one that has not failed.

``

    #define ngx_bitvector_index(index) index / (8 * sizeof(uintptr_t))
    #define ngx_bitvector_bit(index) (uintptr_t) 1 << index % (8 * sizeof(uintptr_t))

    static void
    ngx_http_upstream_free_hash_peer(ngx_peer_connection_t *pc, void *data,
        ngx_uint_t state)
    {
        ngx_http_upstream_hash_peer_data_t  *uhpd = data;
        ngx_uint_t                           current;

        if (state & NGX_PEER_FAILED
                && --pc->tries)
        {
            /* the backend that failed */
            current = uhpd->hash % uhpd->peers->number;

           /* mark it in the bit-vector */
            uhpd->tried[ngx_bitvector_index(current)] |= ngx_bitvector_bit(current);

            do { /* rehash until we're out of retries or we find one that hasn't been tried */
                uhpd->hash = ngx_hash_key((u_char *)&uhpd->hash, sizeof(ngx_uint_t));
                current = uhpd->hash % uhpd->peers->number;
            } while ((uhpd->tried[ngx_bitvector_index(current)] & ngx_bitvector_bit(current)) && --pc->tries);
        }
    }

This works because the load-balancing function will just look at the new value of `uhpd->hash`.

Many applications won't need retry or high-availability logic, but it's possible to provide it with just a few lines of code like you see here.

6. Writing and Compiling a New Nginx Module
-------------------------------------------

So by now, you should be prepared to look at an Nginx module and try to understand what's going on (and you'll know where to look for help). Take a look in [src/http/modules/](http://lxr.evanmiller.org/http/source/http/modules/) to see the available modules. Pick a module that's similar to what you're trying to accomplish and look through it. Stuff look familiar? It should. Refer between this guide and the module source to get an understanding about what's going on.

But Emiller didn't write a *Balls-In Guide to Reading Nginx Modules*. Hell no. This is a *Balls-Out Guide*. We're not reading. We're writing. Creating. Sharing with the world.

First thing, you're going to need a place to work on your module. Make a folder for your module anywhere on your hard drive, but separate from the Nginx source (and make sure you have the latest copy from [nginx.net](http://nginx.net)). Your new folder should contain two files to start with:

-   "config"
-   "ngx\_http\_\<your module\>\_module.c"

The "config" file will be included by `./configure`, and its contents will depend on the type of module.

**"config" for filter modules:**

``

    ngx_addon_name=ngx_http_<your module>_module
    HTTP_AUX_FILTER_MODULES="$HTTP_AUX_FILTER_MODULES ngx_http_<your module>_module"
    NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_<your module>_module.c"

**"config" for other modules:**

``

    ngx_addon_name=ngx_http_<your module>_module
    HTTP_MODULES="$HTTP_MODULES ngx_http_<your module>_module"
    NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_<your module>_module.c"

Now for your C file. I recommend copying an existing module that does something similar to what you want, but rename it "ngx\_http\_\<your module\>\_module.c". Let this be your model as you change the behavior to suit your needs, and refer to this guide as you understand and refashion the different pieces.

When you're ready to compile, just go into the Nginx directory and type

``

    ./configure --add-module=path/to/your/new/module/directory

and then `make` and `make install` like you normally would. If all goes well, your module will be compiled right in. Nice, huh? No need to muck with the Nginx source, and adding your module to new versions of Nginx is a snap, just use that same `./configure` command. By the way, if your module needs any dynamically linked libraries, you can add this to your "config" file:

``

    CORE_LIBS="$CORE_LIBS -lfoo"

Where `foo` is the library you need. If you make a cool or useful module, be sure to send a note to the [Nginx mailing list](http://wiki.codemongers.com/MailinglistSubscribe) and share your work.

7. Advanced Topics
------------------

This guide covers the basics of Nginx module development. For tips on writing more sophisticated modules, be sure to check out *[Emiller's Advanced Topics In Nginx Module Development](nginx-modules-guide-advanced.html)*.

Appendix A: Code References
---------------------------

[Nginx source tree (cross-referenced)](http://lxr.evanmiller.org/http/source/)

[Nginx module directory (cross-referenced)](http://lxr.evanmiller.org/http/source/http/modules/)

[Example addon: circle\_gif](/nginx/ngx_http_circle_gif_module.c.txt)

[Example addon: upstream\_hash](/nginx/ngx_http_upstream_hash_module.c.txt)

[Example addon: upstream\_fair](http://github.com/gnosek/nginx-upstream-fair/tree/master)

Appendix B: Changelog
---------------------

-   January 16, 2013: Corrected code sample in 5.5.
-   December 20, 2011: Corrected code sample in 4.2 (one more time).
-   March 14, 2011: Corrected code sample in 4.2 (again).
-   November 11, 2009: Corrected code sample in 4.2.
-   August 13, 2009: Reorganized, and moved *[Advanced Topics](nginx-modules-guide-advanced.html)* to a separate article.
-   July 23, 2009: Corrected code sample in 3.5.3.
-   December 24, 2008: Corrected code sample in 3.4.
-   July 14, 2008: Added information about subrequests; slight reorganization
-   July 12, 2008: Added [Grzegorz Nosek](http://localdomain.pl/)'s guide to shared memory
-   July 2, 2008: Corrected "config" file for filter modules; rewrote introduction; added TODO section
-   May 28, 2007: Changed the load-balancing example to the simpler upstream\_hash module
-   May 19, 2007: Corrected bug in body filter example
-   May 4, 2007: Added information about load-balancers
-   April 28, 2007: Initial draft

* * * * *

[Back to Evan Miller's home page](/) – [Follow on Twitter](http://twitter.com/EvMill) – [Subscribe to RSS](/news.xml)
