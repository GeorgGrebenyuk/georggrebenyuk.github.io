# NVP - Подводя промежуточные итоги

В [первой части](article_12082024_NVP.md) цикла статей про NVP я писал, что первой очередью были выпущены пакеты нодов для Renga, nanoCAD и ModelStudio CS, при этом все примеры приводил в основном о Renga, особенно во [второй статье](article_26082024_NVP.md), рассказывая о сложностях реализации универсальных сценариев создания нодов, в этой статье, частично повторяя изложенные мысли, подвожу итоги, прикладываю рекомендации по созданию своих пакетов нодов и временно приостанавливаю работу над NVP.

## 1. Варианты автоматизации задач в САПР

Любая автоматизация возможна только при наличии в данной САПР на то возможностей (API). По этой причине я не пишу, например, статей про продукцию от ИндорСофт или Кредо-диалог (у них такой радости нет и не планируется). При этом само API по виду взаимодействия пользователя с программой может делиться на "внутреннее" и "внешнее". 

Внутренним будем называть такое API, вызывать функции которого можно из интерфейса самой запущенной программы, и не так важно, будет это внутренний скриптовый язык или компилируемая библиотека классов.

Внешним будем называть возможность подключиться к программе из стороннего процесса или даже сети. Если речь о связи в пределах одного ПК, то здесь можно выделить специфичную Windows технологию COM или более общую (в.т.ч. на Linux -- технологию D-Bus). Для программ, развернутых на серверах или доступных через браузер это может быть, например,  REST API или иной web-протокол.

Ниже я попробовал накидать табличку в меру своих знаний о различных САПР на рынке и вариантах автоматизации для них:

| Наименование продукта                | COM API | .NET API (C#) | REST API | C++ API |
| ------------------------------------ | ------- | ------------- | -------- | ------- |
| <u>Отечественные продукты</u>        |         |               |          |         |
| платформа nanoCAD                    | +       | +             | -        | +       |
| nanoCAD BIM Строительство            | ±       | +             | -        | -       |
| Renga                                | +       | -             | -        | -       |
| ModelStudio CS                       | +       | -             | -        | +       |
| CADLib CS                            | ±       | +             | -        | -       |
| SCAD                                 | ±       | -             | -        | +       |
| Tangle                               | -       | -             | +        | -       |
| Larix                                | -       | +             | ?        | -       |
| TDMS                                 | +       | +             | -        | -       |
| <u>Недоступные (де-юре) продукты</u> |         |               |          |         |
| Autodesk Revit                       | -       | +             | -        | -       |
| Autodesk Navisworks                  | +       | -             | -        | ±       |
| Autodesk AutoCAD                     | +       | +             | -        | +       |
| ...                                  |         |               |          |         |

## 2. Разработка библиотек нодов NVP под САПР

Наиболее просто и быстро реализовать в NVP взаимодействие с продуктами, если у них есть COM или REST API. Для первого сценария (COM) можно воспользоваться библиотекой [GitHub - GeorgGrebenyuk/idl2nvp_converter: Translator ActiveX lib (from IDL file) to C# NET DLL for NVP as node-package library](https://github.com/GeorgGrebenyuk/idl2nvp_converter). По этой логике я писал ноды под nanoCAD, ModelStudio CS и Renga. Взаимодействие через REST API я ещё не делал, но подход там аналогичный (кто с REST работал). 

Для загружаемых .NET библиотек всё оказалось на порядок сложнее. Ещё более сложнее, если у продукта есть С++ API -- в этом случае надо будет создавать дополнительную библиотеку на C++ CLI и мучиться уже с ней в другой .NET-библиотеке. 

Для решения частных задач вполне подходит идея "функционального программирования", то есть имеются команды\ноды, принимающие на вход поток информации и возвращающие результат или действие.

Для закрытия задач отрасли таких узких нодов чаще всего оказывается не достаточным, и нужен доступ ко всем объектам и функциям, которым располагает API. Как я уже писал в первой части статьи, под COM специально была написана утилита, генерирующая такой перечень кода, который после небольшой доработки станет пакетом нодов. В принципе, под .NET тоже можно было бы её сделать, но я уже не стал по двум причинам: время и особенности доступа к объектам. Дело в том, что в COM все типы данных почти везде были простые -- числа, строки, а сами нативные объекты (COM-интерфейсы) имели только методы. В .NET есть много команд, принимающих на вход иные целые классы, и адаптация кода для набора нодов потребовало бы огромных трудозатрат. Тут же на общую производительность NVP будет сказываться объем кода (как правило, объем NET API больше в несколько раз, чем в COM, и с такими объемами конечных нодов будет тяжело работать).

Ниже я привел тезисы, которых можно придерживаться при разработке нодов NVP под .NET API:

* каждое пространство имён переносить аналогично в NVP в виде вложенной папки, добавляя обязательный префикс (например, "\_")  к namespace;
* по максимуму постараться не использовать ноды-конструкторы, принимающие на вход неуправляемый (нативный) объект из-за риска падения приложения (причина см. ниже);
* все обёртки вокруг нативных объектов (здесь, объектов данной САПР для которых есть API) реализовывать для нейтрального представления объекта. Например, в nanoCAD и всех вертикальных приложениях объект может быть получен для записи или чтения. Попытка изменить объект, полученный для чтения приведет к вылету. Попытка хранить объекты вне транзакции с большой долей вероятности приведет к вылету и т.д. Поэтому целесообразно во-первых, хранить не сам объект а его идентификатор (в nanoCAD -- Teigha.DatabaseServices.ObjectId), а режимы доступа устанавливать самостоятельно -- при возврате свойства - READ, при записи -- WRITE;

Экспериментально я пришел к следующему подходу. Создается вспомогательный интерфейс IEditAction с методами, возвращающими или редактирующими значение нативного объекта для всех возможных процедур по API (ниже приведен пример для базового объекта nanoCAD Teigha.Entity)

```csharp
//Интерфейс для создания вспомогательного класса к нативному объекту для его чтения/изменения
public interface IEditAction
{
    object GetData(ObjectId id, object modeRaw);
    void SetData(ObjectId id, object modeRaw, object data);
}
```

```csharp
//Вспомогательный класс для работы с объектом nanoCAD Teigha.Entity
public class Entity_Helper : IEditAction
{
    public enum Mode : int
    {
        Color,
        Layer,
        Linetype,
        LinetypeScale,
        Material,
        PlotStyleName,
        Transparency,
        Visible,
        //TODO: также для методов
    }
    public void SetData(ObjectId id, object mode_raw, object data)
    {
        Mode mode = (Mode)mode_raw;
        var doc = HostMgdpplicationServices.Document.Create(CommonData.ThisDocument);
        //Продумать с целевой БД
        using (Transaction acTrans = doc.Database.TransactionManager.StartTransaction())
        {
            Entity ent = acTrans.GetObject(id, OpenMode.ForWrite) as Entity;

            if (mode == Mode.Color) ent.Color = (Teigha.Colors.Color)data;
            else if (mode == Mode.Layer) ent.Layer = (string)data;
            else if (mode == Mode.Linetype) ent.Linetype = (string)data;
            else if (mode == Mode.LinetypeScale) ent.LinetypeScale = (double)data;
            else if (mode == Mode.Material) ent.Material = (string)data;
            else if (mode == Mode.PlotStyleName) ent.PlotStyleName = (string)data;
            else if (mode == Mode.Transparency) ent.Transparency = (Teigha.Colors.Transparency)data;
            else if (mode == Mode.Visible) ent.Visible = (bool)data;

            acTrans.Commit();
        }
    }

    public object GetData(ObjectId id, object modeRaw)
    {
        Mode mode = (Mode)modeRaw;
        object Data = null;
        var doc = HostMgd.ApplicationServices.Document.Create(CommonData.ThisDocument);
        //Продумать с целевой БД
        using (Transaction acTrans = doc.Database.TransactionManager.StartTransaction())
        {
            Entity ent = acTrans.GetObject(id, OpenMode.ForRead) as Entity;

            if (mode == Mode.Color) Data = ent.Color;
            else if (mode == Mode.Layer) Data = ent.Layer;
            else if (mode == Mode.Linetype) Data = ent.Linetype ;
            else if (mode == Mode.LinetypeScale) Data = ent.LinetypeScale;
            else if (mode == Mode.Material) Data = ent.Material;
            else if (mode == Mode.PlotStyleName) Data = ent.PlotStyleName;
            else if (mode == Mode.Transparency) Data = ent.Transparency;
            else if (mode == Mode.Visible) Data = ent.Visible;

            acTrans.Commit();
        }
        return Data;
    }
}
```

```csharp
//Ноды для получения и задания слоя объекту
[NodeInput("Entity", typeof(object))]
public class Get_Layer : INode
{
    public NodeResult Execute(INVPData context, List<NodeResult> inputs)
    {
        dynamic _input0 = inputs[0].Value;
        var entId = (ObjectId)_input0._o;
        Entity_Helper h = new Entity_Helper();
        var data = h.GetData(entId, Entity_Helper.Mode.Layer);
        return new NodeResult(data);
    }
}

[NodeInput("Entity", typeof(object))]
[NodeInput("Имя слоя", typeof(object))]
public class Set_Layer : INode
{
    public NodeResult Execute(INVPData context, List<NodeResult> inputs)
    {
        dynamic _input0 = inputs[0].Value;
        string _input1 = (string)inputs[1].Value;
        var entId = (ObjectId)_input0._o;
        Entity_Helper h = new Entity_Helper();
        h.SetData(entId, Entity_Helper.Mode.Layer, _input1);

        return new NodeResult(null);
    }
}
```

Как видно выше, реализация сильно отличается от COM -- реализованы вспомогательные сценарии, направленные на безопасную эксплуатацию API.

- вводить группу нодов, облегчающих механики выбора объектов в модели/по типу/типам/классам (а также вводить ноды, возвращающие тип/класс объекта);

- компилировать библиотеку нодов только для той версии Framework, под которую написана программа. При различиях в структуре API (появлении новых/старых нодов) для Framework 4.8 и net6.0 можно оборачивать ноды в конструкции

```context
#if NET6
...
#endif
```

* для автоматизированного формирования файлов манифестов по методике, используемой мной для [GitHub - GeorgGrebenyuk/nvp_NodeLibs: Исходный код библиотек нодов NVP для САПР. https://georggrebenyuk.github.io/nvp_NodeLibs/](https://github.com/GeorgGrebenyuk/nvp_NodeLibs) использовать "живые" библиотеки классов целевой САПР, а не reference-assemblies, также при компиляции копировать их в выходной каталог для срабатывания процедуры assembly.GetTypes();

## 3. И напоследок пожелания к NVP

- Перенести генерацию манифеста **nodeitem** на сторону NVP (индексировать dll при обновлениях). Если не применять автоматизацию как делал я, то его заполнение параллельно с разработкой будет очень муторным;
- Ввести дополнительную цветовую расцветку для нодов, описывающих конструктор класса (ViewType), сейчас приходиться выделять их в списке нодов первым символом "\_";
- Ускорить внутренний поиск нодов в дереве;
- Выпустить и выпускать далее версии на Windows/Linux;

## 4. Подводя итоги

Под NVP создавать ноды можно. Для покрытия всей функциональности нужно участие разработчиков или иных инициативных пользователей (здесь, конкретных САПР для которых пакеты нодов будут создаваться) путем выделения людей на подобные задачи. При условии, конечно, если по юридическим условиям подойдет использование данной среды. 

Также не премину поделиться опасениями возможной смены курса или позицонирования NVP. Будь он чисто open-source'ным, можно было бы сделать ветку (fork) и далее развивать его самостоятельно. С грустью понимаю, что подобное крайне маловероятно в наших реалиях "всё что open-source = бесплатно и без обязательств" и потому не просил и не прошу разработчика сделать его открытым, так как это лишит продукт какой-никакой монетизации и стимулов роста, т.к. популярная за границей идея спонсорства открытых проектов у нас практически не развита. Но это всё отступления, конечно. Благодарю Руслана (автора-разработчика) NVP уже за такую мощную реализацию среды визуального программирования!

P.S. У автора есть свой [блог с интересными заметками](https://t.me/ruslan_shishmarev), рекомендую подписаться.
