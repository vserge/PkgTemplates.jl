``` @meta
CurrentModule = PkgTemplates
```

# PkgTemplates руководство разработчика

``` @contents
Pages = ["developer.md"]
```

Описание проблем и запросы на изменения приветствуются! Новые участники должны обязательно ознакомиться с [Руководством ColPrac для разработчиков](https://github.com/SciML/ColPrac).

[PkgTemplates](https://github.com/JuliaCI/PkgTemplates.jl/) может быть легко расширен путем добавления новых [Плагин](@ref)ов.

Существует три типа плагинов: [`Plugin`](@ref), [`FilePlugin`](@ref) и [`BadgePlugin`](@ref).

``` @docs
Plugin
FilePlugin
BadgePlugin
```

## Конвейер создания Шаблона + Пакета

Конструктор [Шаблона](@ref) в основном делает это:

```         
- извлекать значения из аргументов ключевых слов
- создает Шаблон (Template) из значений
- для каждого плагина:
  - проверяет плагин на соответствие шаблону
```

На этапе проверки плагина используется функция [validate](@ref). Это позволяет нам выявлять ошибки до того, как мы попытаемся сгенерировать пакеты.

``` @docs
validate
```

Процесс генерации пакета выглядит следующим образом:

```         
- создаёт пустой каталог для пакета
- для каждого плагина, упорядоченного по приоритету:
  - запускает пред-задачи (prehook) плагина 
- для каждого плагина, упорядоченного по приоритету:
  - запускает задачи (hook) плагина
- для каждого плагина, упорядоченного по приоритетуy:
  - запускает пост-задачи (posthook)
```

Как вы можете судить, плагины играют центральную роль в настройке пакета.

Тремя основными точками входа для работы плагинов являются пред-перехватчики ([`prehook`](@ref)), перехватчики ([`hook`](@ref)), и пост-перехватчики ([`posthook`](@ref))[^1]. Как следует из названий, они в основном означают "перед основным этапом", "основной этап" и "после основного этапа" соответственно.

[^1]: Прим. переводчика: В некоторой литературе используется не перевод "Перехватчики", а заимствование слова - "хуки".

Каждый этап в основном идентичен, поскольку функции принимают одни и те же аргументы. Однако множественность этапов позволяет нам зависеть от артефактов предыдущих этапов. Например, плагин [Git](@ref) использует пост-перехватчики ([posthook](@ref)) для фиксации (commit) всех сгенерированных файлов, но не имеет смысла делать это до того, как файлы будут сгенерированы.

Но как насчет зависимостей внутри одной и той же стадии? В этом случае у нас есть приоритет ([`priority`](@ref)) в определении того, какие плагины когда будут запущены. Плагин [`Git`](@ref) также использует эту функцию для снижения приоритета своих пост-перехватчиков (posthook), так что даже если другие плагины генерируют файлы в своих пост-перехватчики (posthooks), они все равно фиксируются (при условии, что эти плагины не установили еще более низкий приоритет).

``` @docs
prehook
hook
posthook
priority
```

## Пошаговое руководство по применению подтипа `Plugin` 

Конкретные типы, которые наследуют подтип [`Plugin`](@ref) напрямую, могут делать практически все, что угодно. Чтобы понять, как они реализованы, давайте рассмотрим упрощенные версии двух плагинов: [`Documenter`](@ref) для изучения шаблонизирования и [`Git`](@ref) для дальнейшего разъяснения многоступенчатого конвейера.

### Пример: `Documenter`

``` julia
@plugin struct Documenter <: Plugin
    make_jl::String = default_file("docs", "make.jl")
    index_md::String = default_file("docs", "src", "index.md")
end

gitignore(::Documenter) = ["/docs/build/"]

badges(::Documenter) = [
    Badge(
        "Stable",
        "https://img.shields.io/badge/docs-stable-blue.svg",
        "https://{{{USER}}}.github.io/{{{PKG}}}.jl/stable",
    ),
    Badge(
        "Dev",
        "https://img.shields.io/badge/docs-dev-blue.svg",
        "https://{{{USER}}}.github.io/{{{PKG}}}.jl/dev",
    ),
]

view(p::Documenter, t::Template, pkg::AbstractString) = Dict(
    "AUTHORS" => join(t.authors, ", "),
    "PKG" => pkg,
    "REPO" => "$(t.host)/$(t.user)/$pkg.jl",
    "USER" => t.user,
)

function hook(p::Documenter, t::Template, pkg_dir::AbstractString)
    pkg = pkg_name(pkg_dir)
    docs_dir = joinpath(pkg_dir, "docs")

    make = render_file(p.make_jl, combined_view(p, t, pkg), tags(p))
    gen_file(joinpath(docs_dir, "make.jl"), make)

    index = render_file(p.index_md, combined_view(p, t, pkg), tags(p))
    gen_file(joinpath(docs_dir, "src", "index.md"), index)

    # What this function does is not relevant here.
    create_documentation_project()
end
```

Макрос `@plugin` определяет некоторые полезные для нас методы. Внутри нашего определения структуры мы используем [`default_file`](@ref) для ссылки на файлы в этом репозитории.

``` @docs
@plugin
default_file
```

Первый метод, который мы реализуем для `Documenter`, - это [`gitignore`](@ref), так что пакеты, созданные с помощью этого плагина, игнорируют артефакты сборки документации.

``` @docs
gitignore
```

Во-вторых, мы реализуем значки ([`badges`](@ref) ), чтобы добавить пару значков в файлы README новых пакетов.

``` @docs
badges
Badge
```

Эти две функции, [`gitignore`](@ref) и [`badges`](@ref), в настоящее время являются единственными "специальными" функциями для взаимодействия между плагинами. В других случаях вы все еще можете получить доступ к плагинам Шаблона ([`Template`](@ref)) в зависимости от наличия/свойств других плагинов через [`getplugin`](@ref), хотя это менее эффективно.

``` @docs
getplugin
```

В-третьих, мы реализуем виды ([`view`](@ref)), которые используются для заполнения заполнителей в значках и отображаемых файлах.

``` @docs
view
```

Наконец, мы реализуем задачу ([`hook`](@ref)), которая является настоящей рабочей лошадкой для плагина. Внутри этой функции мы генерируем пару файлов с помощью еще нескольких функций создания текстовых шаблонов.

``` @docs
render_file
render_text
gen_file
combined_view
tags
pkg_name
```

Дополнительные сведения о создании текстовых шаблонов см. в пошаговом руководстве FilePlugin ( [`FilePlugin` Walkthrough](@ref)) и разделе, посвященном Файлам Пользовательских Шаблонов ( [Custom Template Files](@ref)).

### Пример: `Git`

``` julia
struct Git <: Plugin end

priority(::Git, ::typeof(posthook)) = 5

function validate(::Git, ::Template)
    foreach(("user.name", "user.email")) do k
        if isempty(LibGit2.getconfig(k, ""))
            throw(ArgumentError("Git: Global Git config is missing required value '$k'"))
        end
    end
end

function prehook(::Git, t::Template, pkg_dir::AbstractString)
    LibGit2.with(LibGit2.init(pkg_dir)) do repo
        LibGit2.commit(repo, "Initial commit")
        pkg = pkg_name(pkg_dir)
        url = "https://$(t.host)/$(t.user)/$pkg.jl"
        close(GitRemote(repo, "origin", url))
    end
end

function hook(::Git, t::Template, pkg_dir::AbstractString)
    ignore = mapreduce(gitignore, append!, t.plugins)
    unique!(sort!(ignore))
    gen_file(joinpath(pkg_dir, ".gitignore"), join(ignore, "\n"))
end

function posthook(::Git, ::Template, pkg_dir::AbstractString)
    LibGit2.with(GitRepo(pkg_dir)) do repo
        LibGit2.add!(repo, ".")
        LibGit2.commit(repo, "Files generated by PkgTemplates")
    end
end
```

Мы не использовали макрос `@plugin` для этого, потому что там нет полей. Проверка и все три перехвата реализованы:

-   [`validate`](@ref) удостоверяется, что присутствует вся необходимая конфигурация Git.
-   [`prehook`](@ref) создает репозиторий Git для пакета.
-   [`hook`](@ref) генерирует файл `.gitignore` , используя специальную функцию [`gitignore`](@ref).
-   [`posthook`](@ref) добавляет и фиксирует все сгенерированные файлы.

Как упоминалось ранее, мы используем приоритет ([`priority`](@ref)), чтобы убедиться, что мы дождемся, пока все остальные плагины завершат свою работу, прежде чем фиксировать файлы.

Надеюсь, это демонстрирует уровень контроля, который вы имеете над процессом генерации пакетов при разработке плагинов, и когда имеет смысл использовать эту власть!

## Пошаговое руководство по применению подтипа `FilePlugin` 

В большинстве случаев вам на самом деле не нужны все элементы управления, которые мы продемонстрировали выше. Плагины, имеющие подтип [`FilePlugin`](@ref), выполняют гораздо более ограниченную задачу. В общем, они просто генерируют один файл шаблона.

Для иллюстрации давайте посмотрим на плагин [`Citation`](@ref), который создает файл `CITATION.bib`.

``` julia
@plugin struct Citation <: FilePlugin
    file::String = default_file("CITATION.bib")
end

source(p::Citation) = p.file
destination(::Citation) = "CITATION.bib"

tags(::Citation) = "<<", ">>"

view(::Citation, t::Template, pkg::AbstractString) = Dict(
    "AUTHORS" => join(t.authors, ", "),
    "MONTH" => month(today()),
    "PKG" => pkg,
    "URL" => "https://$(t.host)/$(t.user)/$pkg.jl",
    "YEAR" => year(today()),
)
```

Аналогично приведенному выше примеру с `Documenter`, мы определяем конструктор ключевых слов и назначаем файл шаблона по умолчанию из этого репозитория. Этот плагин ничего не добавляет в `.gitignore`, и он не добавляет никаких значков, поэтому реализации для [`gitignore`](@ref) и [`badges`](@ref) опущены.

Сначала мы реализуем [`source`](@ref) и [`destination`](@ref), чтобы определить, откуда берется файл шаблона и куда он направляется. Эти функции специфичны для плагинов, наследующих подтип [`FilePlugin`](@ref), и по умолчанию не влияют на обычные плагины наследующие подтип [`Plugin`](@ref).

``` @docs
source
destination
```

Далее мы реализуем теги ([`tags`](@ref)). Мы кратко ознакомились с этой функцией ранее, но в данном случае необходимо изменить ее поведение по умолчанию. Чтобы понять почему, возможно, полезно просмотреть файл шаблона целиком:

```         
@misc{<<&PKG>>.jl,
    author  = {<<&AUTHORS>>},
    title   = {<<&PKG>>.jl},
    url     = {<<&URL>>},
    version = {v0.1.0},
    year    = {<<&YEAR>>},
    month   = {<<&MONTH>>}
}
```

Поскольку файл содержит свои собственные разделители `{}`, нам нужно использовать разные разделители для правильной работы шаблонов.

Наконец, мы реализуем [`view`](@ref) для заполнения полей, которые мы видели в файле шаблона.

## Выполнение Дополнительной Работы с Плагинами наследующими подтип `FilePlugin`

Обратите внимание, что нам не нужно было реализовывать перехватичик [`hook`](@ref) для нашего плагина. Это реализовано для всех [`FilePlugin`](@ref), например, так:

``` julia
function render_plugin(p::FilePlugin, t::Template, pkg::AbstractString)
    return render_file(source(p), combined_view(p, t, pkg), tags(p))
end

function hook(p::FilePlugin, t::Template, pkg_dir::AbstractString)
    source(p) === nothing && return
    pkg = pkg_name(pkg_dir)
    path = joinpath(pkg_dir, destination(p))
    text = render_plugin(p, t, pkg)
    gen_file(path, text)
end
```

Но что, если мы хотим сделать немного больше, чем просто сгенерировать один файл?

Хорошим примером этого является плагин [`Tests`](@ref). Он создает `runtests.jl`, но также изменяет`Project.toml`, чтобы включить зависимость для тестов `Test`.

Конечно, мы могли бы использовать обычный тип [`Plugin`](@ref), но, оказывается, есть способ избежать этого, сохраняя при этом желаемые дополнительные возможности.

Плагин реализует свой собственный перехватчик (`hook`), но использует `invoke`, чтобы избежать дублирования кода создания файла::

``` julia
@plugin struct Tests <: FilePlugin
    file::String = default_file("runtests.jl")
end

source(p::Tests) = p.file
destination(::Tests) = joinpath("test", "runtests.jl")
view(::Tests, ::Template, pkg::AbstractString) = Dict("PKG" => pkg)

function hook(p::Tests, t::Template, pkg_dir::AbstractString)
    # Do the normal FilePlugin behaviour to create the test script.
    invoke(hook, Tuple{FilePlugin, Template, AbstractString}, p, t, pkg_dir)
    # Do some other work.
    add_test_dependency()
end
```

Существует также реализация проверки ([`validate`](@ref)) по умолчанию для подтипа [`FilePlugin`](@ref), которая проверяет, существует ли исходный файл плагина ([`source`](@ref)), и в противном случае выдает `ArgumentError`. Если вы хотите продлить проверку, но сохранить проверку существования файла, используйте метод `invoke`, как описано выше.

For more examples, see the plugins in the [Continuous Integration (CI)](@ref) and [Code Coverage](@ref) sections. Дополнительные примеры смотрите в разделе плагины в разделах Непрерывная интеграция (CI) ([Continuous Integration (CI)](@ref)) и покрытие кода ([Code Coverage](@ref)).

## Поддержка интерактивного режима

Когда дело доходит до поддержки интерактивного режима для ваших пользовательских плагинов, у вас есть два варианта: написать свой собственный интерактивный ([`interactive`](@ref)) метод или использовать метод по умолчанию. Если вы выберете первый вариант, то вы вольны реализовать метод так, как вам хочется. Если вы хотите использовать реализацию по умолчанию, то есть несколько функций, о которых вам следует знать, хотя во многих случаях вам не нужно будет добавлять какие-либо новые методы.

``` @docs
interactive
prompt
customizable
input_tips
convert_input
```

## Разные советы

### Запись файлов шаблонов

Обзор написания файлов шаблонов для Mustache.jl приведен в разделе Пользовательские файлы шаблонов ([Custom Template Files](@ref)) в руководстве пользователя.

### Предикаты

Существует несколько функций-предикатов для плагинов, которые иногда используются для ответа на вопросы типа "есть ли в этом шаблоне (`Template`) какие-либо плагины для покрытия кода?". Если вы реализуете плагин, который подпадает под одну из следующих категорий, было бы разумно реализовать соответствующую функцию предиката, возвращающую значение `true` для экземпляров вашего типа.

``` @docs
needs_username
is_ci
is_coverage
```

### Форматирование номеров версий

При написании конфигурационных файлов для служб CI часто требуется работа с номерами версий. Есть несколько удобных функций, которые можно использовать, чтобы немного упростить это.

``` @docs
compat_version
format_version
collect_versions
```

## Тестирование

Если вы пишете новый классный плагин, который может быть полезен другим людям, или находите и исправляете ошибку, мы рекомендуем вам открыть запрос на удаление с вашими изменениями. Вот несколько советов по тестированию, чтобы убедиться, что ваш PR проходит как можно более гладко.

### Обновление Эталонных тестов & Приспособлений

Если вы добавили или изменили плагины, вам следует обновить эталонные тесты и связанные с ними тестовые приспособления. В `test/reference.jl` вы найдете набор тестов "Эталонных тестов" ("Reference tests"), который в основном генерирует набор пакетов, а затем проверяет каждый файл на соответствие ссылочному файлу, который хранится где-то в `test/fixtures`. Обратите внимание, что эталонные тесты выполняются только в одной конкретной версии Julia; проверьте `test/runtests.jl`, чтобы увидеть используемую текущую версию.

Для новых плагинов вам следует добавить экземпляр вашего плагина в тестовые наборы "Все плагины" и "Дурацкие опции", затем запустить тесты с помощью `Pkg.test`. Они должны пройти, и в `test/fixtures` появятся новые файлы. Проверьте их, чтобы убедиться, что они содержат именно то, что вы ожидали!

Для внесения изменений в существующие плагины соответствующим образом обновите параметры плагина в тестовом наборе "Дурацкие параметры". Неудачные тесты дадут вам возможность просмотреть и принять изменения в настройках, автоматически обновив файлы для вас.

### Запуск эталонных тестов локально

In the file `test/runtests.jl`, there is a variable called `REFERENCE_JULIA_VERSION`, currently set to `v"1.7.2"`. If you use any other Julia version (even the latest stable one) to launch the test suite, the reference tests mentioned above will not run, and you will miss a crucial correctness check for your code. Therefore, we strongly suggest you test PkgTemplates locally against Julia 1.7.2. This version can be easily installed and started with [juliaup](https://github.com/JuliaLang/juliaup) В файле `test/runtests.jl` есть переменная с именем `REFERENCE_JULIA_VERSION`, в настоящее время установленная в `v"1.7.2"`. Если вы используете любую другую версию Julia (даже последнюю стабильную) для запуска набора тестов, упомянутые выше эталонные тесты не будут запущены, и вы пропустите важную проверку корректности вашего кода. Поэтому мы настоятельно рекомендуем вам протестировать PkgTemplates локально на Julia 1.7.2. Эту версию можно легко установить и запустить с помощью [juliaup](https://github.com/JuliaLang/juliaup):

``` bash
juliaup add 1.7.2
julia +1.7.2
```

### Обновление тестов "Show"

В зависимости от того, что вы изменили, тесты в `test/show.jl` могут завершиться неудачей. Чтобы исправить это, вам нужно обновить `expected` значение, чтобы оно соответствовало тому, что фактически отображается в Julia REPL (при условии, что новое значение правильное).