# ModelStudio CS  - Исследуем C++ API. Часть 1

*23 сентября 2024г.*

Исследовательская статья, как устроено C++ API у CSoft ModelStudio (на примере последней актуальной версии от мая 2024г.)

> Дисклеймер: настоящий материал является личным мнением, его не следует интерпретировать, как официальную позицию разработчика (ООО "Нанософт разработка" или ГК СиСофт Девелопмент)

Термины и сокращения: 

- MST: ModelStudio CS;

## 1. Состав SDK

Версии MST SDK предоставляются по запросу и имеются как правило не для каждого стабильного релиза CSoft ModelStudio (*а может, по запросу, могут давать и для каждого*). Следует понимать, что вся линейка ModelStudio - это объекты на базе платформы nanoCAD (AutoCAD). С прошлой осени (2023г), как CSdev [включили в SDN-список OFAC](https://sanctionssearch.ofac.treas.gov/Details.aspx?id=48745) версии под AutoCAD перестали распространяться, и приложения могут быть собраны только под nanoCAD.

Говоря о типе API, все объекты ModelStudio написаны на NRX (Object ARX у AutoCAD); напомню, это C++ API, требующий для сборки также SDK базового продукта -- здесь, платформы nanoCAD (а ранее и AutoCAD).

Так как релизы CSoft ModelStudio выпускаются чаще платформенных, особенно с использованием промежуточных версий платформы nanoCAD в части доработок API, то и SDK для ModelStudio помимо непосредственно NRX-API самой ModelStudio включает в себя также компоненты платформенного API, отличного от [стабильных выпусков](https://developer.nanocad.ru/redmine/projects/ncadsdk/files), размещаемых в Клубе разработчиков. Более того, сам состав NRX API, идущий в комплекте с MST SDK существенно шире платформенного.

Типовой релиз SDK имеет следующую структуру папок:

SDK

    |--External (*внешние зависимости кроме платформенного API*)

    |--Model Studio

        |--Common (*общие компоненты для всей линейки MST*)

        |--Source (*API для отдельных модулей, общие файлы props*)

    |--NCProject (*расширенная версия NRX платформы nanoCAD*)

## 2. Сборка проекта

Для удобства сборки, разработчики CSoft оставили скрипт автосборки в `\Model Studio\Source\StartVS16_NCProject.bat`. Он запускается, создает временные переменные окружения и открывает демонстрационный проект, идущий в SDK. Если проект успешно компилируется, значит SDK корректный, и в системе имеются необходимые компоненты для разработки.

**Примечание**: так как релизы MST распространяются под несколько версий платформы nanoCAD, то и SDK имеет конфиги для нескольких версий nanoCAD (ncProject). В силу различий платформенного NRX в последних релизах (23, 24, 24.1), то рекомендовать собирать проекты под несколько версий nanoCAD, а также несколько версий самого MST будет опрометчиво. Надо быть готовым, что в таком случае структура проекта библиотеки будет очень сложной.

Обычно MST SDK поставляется только в Release-версии, хоть конфиг демонстрационного проекта и имеет различные конфигурации. Соответственно, пользовательские Debug, и Release-конфигурации будут собираться на одной и той же версии SDK (если данный SDK не включает Debug-версию помимо Release).

Некоторым разработчикам (и мне) неудобен путь настройки рабочего окружения через предварительный запуск bat-файла, я привык к упоминанию всех зависимостей в локальном \*.props файле. Поэтому давайте рассмотрим, какие переменные окружения скриптом устанавливаются:

```batch
::Установка расширенной версии NRX
set WORKROOT=%~dp0..\..\NCProject21\Version7
::Конструкция "%~dp0" является макросом, возвращаюшим текущий 
::каталог, здесь -- "SDK\Model Studio\Source"

::Установка системных библиотек для сборки
set MSExternal=%~dp0..\..\External
set SQLITE=%MSExternal%\SQLite3240000

::Установка переменных MST
set CommonCS=%~dp0..\Common\Common_vc7
set CSOFT=%~dp0

::Установка переменных NCProject и NRX
set NCADAPI10=%WORKROOT%
set NANOAPPS_NCAD10=%WORKROOT%\NApps\NCad
set OPENDWG10=%NCADAPI10%\Common\DwgDirect21
set MAGMA6=%NCADAPI10%\Magma\Core
set NTHUNKLIB10=%NCADAPI10%\sources\NRX\NrxGate
```

Остальное окружение типа boost и IfcSdkLibPath признано несущественным (собиралось и без него), так как упомянутые bat-файлы по-видимому служат для компиляции не только файла примера но и чего-то ещё. 

Далее я "перевел" инструкции из bat-файла в props-определение. Исходный путь, по которому у меня размещено SDK, это "SDK_WORKROOT", и у же от неё получаются переменные, упомянутые в bat-файле.

```xml
<?xml version="1.0" encoding="utf-8"?>
<Project DefaultTargets="Build" ToolsVersion="4.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <ImportGroup Label="PropertySheets">
  </ImportGroup>
  <PropertyGroup>
    <SDK_WORKROOT>D:\PROCESSING\SDK\ModelStudio\MSSDK_08052024</SDK_WORKROOT>
    <WORKROOT>$(SDK_WORKROOT)\NCProject21\Version7</WORKROOT>
    <MSExternal>$(SDK_WORKROOT)\External</MSExternal>
    <SQLITE>$(MSExternal)\SQLite3240000</SQLITE>
    <CommonCS>$(SDK_WORKROOT)\Model Studio\Common\Common_vc7</CommonCS>
    <CSOFT>$(SDK_WORKROOT)\Model Studio\Source</CSOFT>
    <NCADAPI10>$(WORKROOT)</NCADAPI10>
    <NANOAPPS_NCAD10>$(WORKROOT)\NApps\NCad</NANOAPPS_NCAD10>
    <OPENDWG10>$(NCADAPI10)\Common\DwgDirect21</OPENDWG10>
    <MAGMA6>$(NCADAPI10)\Magma\Core</MAGMA6>
    <NTHUNKLIB10>$(NCADAPI10)\sources\NRX\NrxGate</NTHUNKLIB10>
  </PropertyGroup>
</Project>
```

Также в корне SDK (папка `\Model Studio\Source`) имеются props-конфиги для подключения библиотек платформы nanoCAD (ncProject) и самого MST. Будем рассматривать для 24 версии. Имеются отдельные конфиги для Debug и Release: `vs16_ncproject_fulldebug_x64.props` и  `vs16_ncproject_release_x64.props` соответственно. Отметим, что конфиги настроены на случай подключения именно тестового проекта в составе SDK, поэтому для случая внешнего использования SDK эти конфиги надо слегка изменить. 

Изменению подлежит только файл `vs16_ms_common.props`. В частности, надо изменить данные в Link:

```xml
<Link>
      <AdditionalLibraryDirectories>$(CSOFT)\lib\VS16\Release NCAD\$(Platform);%(AdditionalLibraryDirectories)</AdditionalLibraryDirectories>
      <ImportLibrary>$(CSOFT)\lib\VS16\$(Configuration)\$(Platform)\$(ProjectName).lib</ImportLibrary>
      <AdditionalDependencies>version.lib;msxml2.lib;htmlHelp.lib</AdditionalDependencies>
    </Link>
```

добавив переменную окружения `$(CSOFT)`, которая как раз и указывает на папку с библиотеками модулей MST (папка `\Model Studio\Source`). Также необходимо отредактировать путь к статическим библиотекам, явно указав папку `Release NCAD` вместо внутренней переменной `$(Configuration)`.

Менять конфиги прямо в папке SDK не рекомендуется; надо их скопировать к себе и изменить. Например, я произвел копирование конфигов в отдельную папку репозитория с активным проектом, откуда их уже подключаю к проекту.

Порядок подключения props-конфигов простой: сперва подключается сформированный props, эмулирующий окружение bat-файла, я назвал его `MS_SDK_Common_08052024.props`, а затем уже `vs16_ncproject_release_x64.props`, обращающийся к измененным `vs16_ms_common.props`и оригинальному `vs16_ncproject_common.props`.

Сам проект создается на основе C++ шаблона для Dynamic Link Library и имеет схожие настройки с NRX. Отмечу обнаруженные настройки, которые были у тестового проекта в составе SDK, без которых он не собирается:

- General ➝ SDL checks = пустое значение (не указывать ничего);

- Language ➝ Conformance mode = Default;

Также можно скопировать содержимое файлов stdafx примера в SDK, я пока не разбирался в подробностях их содержимого, это вопрос следующих статей.

## 3. Подводя итоги ...

MST SDK -- это платформа + непосредственно MST API. То есть мешать SDK от платформы с этим не требуется и даже вредно, так как из-за несовместимости версий NRX будут ошибки компиляции или работы функций.

Выносить регистрацию переменных окружения из bat-файла в props-конфиг можно, и такой проект будет собираться.

Настраивать проект и решение для нескольких версий nanoCAD и MST будет очень сложным, я пока о таком думать не буду. 
