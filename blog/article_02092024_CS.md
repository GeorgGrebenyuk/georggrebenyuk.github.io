# Программная сторона публикации данных в CADLib из сторонней САПР

Статья-наблюдение, какие действия происходят при передаче файлов в СОД CADLib (CSoft CADLib: Модель и Архив) из САПР вне экосистемы CSoft.

## 1. Постановка вопроса

При комплексном подходе к проектированию объектов капитального или промышленного строительства часто оказывается недостаточной моно-вендорный подход -- когда используется ПО от одного производителя, в экосистеме которого передача данных осуществляется значительно лучше, чем из сторонних программ вне данной экосистемы, либо при помощи обменных форматов (в основном, IFC).

Объектом интереса в данной небольшой статье будет выступать продукт CSoft Development CADLib: МиА (Модель и Архив), представляющий собой оболочку вокруг базы данных (на PostgreSQL или MS SQL Server) и процесс передачи данных в него из некой неназванной САПР от стороннего вендора.

Изучение цепочки процедур передачи данных поможет создать свою реализацию программного экспорта из другой среды, к которой ещё не существует транслятора данных. 

При этом возможны 2 сценария работы: создается **экспортер** над целевой САПР, осуществляющий выгрузку напрямую в БД CADLib или **импортер** в самом CADLib в виде плагина, читающего файл и импортирующего данные в CADLib. Сценарий импорта характерен для случаев, когда формат файла импорта простой, или на него нет API.

## 2. Последовательность работы

**Шаг 1** - Предзагрузка в память вспомогательных библиотек CADLib (mstudioData.dll, mstudioUI.dll, mstudioDB.dll, mstProjectBuildingHierarchy.dll, mstProjectStructureHierarchy.dll, mstProjectDocuments.dll,mstCoreLoader.dll). Если экспорт осуществляется из стороннего ПО (для плагина над CADLib это не требуется);

Для публикации также используется вспомогательная библиотека MsBinaryPublisher.dll.

```csharp
MsBinaryPublisher m_publisher = new MsBinaryPublisher();
```

**Шаг 2** - Установка параметров подключения к целевой БД CADLib (на PostgreSQL или MS SQL Server) и попытка подключения к ней. Также к БД подключается класс-публикатор:

```csharp
string dbtype;
if (dbtype == "MSSQL")
{
	m_publisher.Connect(server, dbname, user, password, 
        "MSSQL", (int)port, null);
}
else if (dbtype == "PGSQL")
{
	this.m_publisher.Connect(server, dbname, user, password, 
        "PGSQL", (int)port, null);
}
```

**Шаг 3** - Определение объектов для передачи в CADLib и установка количества для публикатора;

```csharp
IList objects;
m_publisher.BeginPublication((long)(objects.Count + 5));
//+6 отведено для элементов иерархии (см. добавление их ниже)
```

**Шаг 4** - Формирование в CADLib структуры для импорта (участка в иерархии начиная с Площадки). Ниже приведена типовая структура, но по идее она может быть и произвольной для возможных значений BUILDINGS_STRUCT_LEVEL: 

```csharp
CSElement cselement = new CSElement("Площадка", null);
cselement.AddParameter("SYS_CATEGORY_GROUP", "Иерархия проекта", "", "");
cselement.AddParameter("BUILDINGS_STRUCT_LEVEL", "1. Площадка (Генплан)", "Уровень иерархии", "");
cselement.AddParameter("CLP_FULL_CODE", "", "Полный шифр объекта структуры", "");
cselement.AddParameter("CLP_ITEM_CODE", "", "CLP_ITEM_CODE", "");
cselement.AddParameter("CLP_SHORT_CODE", "", "Короткий шифр объекта структуры", "");
cselement.AddParameter("SITE_LAYOUT", "", "SITE_LAYOUT", "");
cselement.AddParameter("SITE_LAYOUT_NUMBER", "", "SITE_LAYOUT_NUMBER", "");
cselement.AddParameter("USER_ACCESS_GROUPS", "", "Доступ", "");
CSElement cselement2 = cselement.AddChild("Здание");
cselement2.AddParameter("SYS_CATEGORY_GROUP", "Иерархия проекта", "", "");
cselement2.AddParameter("BUILDINGS_STRUCT_LEVEL", "2. Здание (Сооружение)", "Уровень иерархии", "");
cselement2.AddParameter("CLP_FULL_CODE", "", "Полный шифр объекта структуры", "");
cselement2.AddParameter("CLP_ITEM_CODE", "", "CLP_ITEM_CODE", "");
cselement2.AddParameter("CLP_SHORT_CODE", "", "Короткий шифр объекта структуры", "");
cselement2.AddParameter("USER_ACCESS_GROUPS", "", "Доступ", "");
cselement2.AddParameter("SITE_BUILDING", "", "SITE_BUILDING", "");
cselement2.AddParameter("SITE_BUILDING_NUMBER", "", "SITE_BUILDING_NUMBER", "");
cselement2.AddParameter("SITE_FIRE_CATEGORY", "", "SITE_FIRE_CATEGORY", "");
cselement2.AddParameter("SITE_FIRE_CATEGORY_STANDARD", "", "SITE_FIRE_CATEGORY_STANDARD", "");
CSElement cselement3 = cselement2.AddChild("Ситуация");
cselement3.AddParameter("SYS_CATEGORY_GROUP", "Иерархия проекта", "", "");
cselement3.AddParameter("BUILDINGS_STRUCT_LEVEL", "27. Ситуация", "Уровень иерархии", "");
cselement3.AddParameter("CLP_FULL_CODE", "", "Полный шифр объекта структуры", "");
cselement3.AddParameter("CLP_ITEM_CODE", "", "CLP_ITEM_CODE", "");
cselement3.AddParameter("CLP_SHORT_CODE", "", "Короткий шифр объекта структуры", "");
cselement3.AddParameter("USER_ACCESS_GROUPS", "", "Доступ", "");
CSElement cselement4 = cselement2.AddChild("Системы");
cselement4.AddParameter("SYS_CATEGORY_GROUP", "Иерархия проекта", "", "");
cselement4.AddParameter("BUILDINGS_STRUCT_LEVEL", "28. Системы", "Уровень иерархии", "");
cselement4.AddParameter("CLP_FULL_CODE", "", "Полный шифр объекта структуры", "");
cselement4.AddParameter("CLP_ITEM_CODE", "", "CLP_ITEM_CODE", "");
cselement4.AddParameter("CLP_SHORT_CODE", "", "Короткий шифр объекта структуры", "");
cselement4.AddParameter("USER_ACCESS_GROUPS", "", "Доступ", "");
CSElement cselement5 = cselement2.AddChild("Конструкции");
cselement5.AddParameter("SYS_CATEGORY_GROUP", "Иерархия проекта", "", "");
cselement5.AddParameter("BUILDINGS_STRUCT_LEVEL", "29. Конструкции", "Уровень иерархии", "");
cselement5.AddParameter("CLP_FULL_CODE", "", "Полный шифр объекта структуры", "");
cselement5.AddParameter("CLP_ITEM_CODE", "", "CLP_ITEM_CODE", "");
cselement5.AddParameter("CLP_SHORT_CODE", "", "Короткий шифр объекта структуры", "");
cselement5.AddParameter("USER_ACCESS_GROUPS", "", "Доступ", "");
CSElement cselement6 = cselement2.AddChild("Разное");
cselement6.AddParameter("SYS_CATEGORY_GROUP", "Иерархия проекта", "", "");
cselement6.AddParameter("BUILDINGS_STRUCT_LEVEL", "30. Разное", "Уровень иерархии", "");
cselement6.AddParameter("CLP_FULL_CODE", "", "Полный шифр объекта структуры", "");
cselement6.AddParameter("CLP_ITEM_CODE", "", "CLP_ITEM_CODE", "");
cselement6.AddParameter("CLP_SHORT_CODE", "", "Короткий шифр объекта структуры", "");
cselement6.AddParameter("USER_ACCESS_GROUPS", "", "Доступ", "");

int m_SituationObjId = MsBinaryPublisher.PublishElement(cselement, "structure_data", 1);
int m_SystemsObjId = m_SituationObjId + 1;
int m_ConstrObjId = m_SystemsObjId + 1;
int m_StuffObjId= m_ConstrObjId + 1;
```

**Шаг 5** - Установка по умолчанию значения Id структуры CADLib для импорта объектов методом `MsBinaryPublisher.BuildingOwnerObjId`;

```csharp
MsBinaryPublisher.BuildingOwnerObjId = m_SituationObjId  - 1;
```

**Шаг 6** - Итеративная публикация каждого из объектов

```csharp
CSElement cselement = new CSElement();
//Установка объекту свойств
cselement.AddParameter("Масса", "70.8", "PARAM_Massa");
//Создание геометрии объекта (конвертация из нативной геометрии)
CSMesh csmesh; 
//Установка Id родительского объекта/категории для объекта. 
//Имеются в в виду категории в CADLib (Ситуация, Разное, Конструкции и тд)
MsBinaryPublisher.BuildingOwnerObjId = 102;
//Отправка элемента на публикацию
int num = MsBinaryPublisher.PublishElement(cselement, "Object", 1);
//Возврат обратно Id родительского объекта/категории
MsBinaryPublisher.BuildingOwnerObjId = m_SituationObjId  - 1;
//Публикация геометрии, если она имеет хоть одну грань
if (csmesh == null || (csmesh.Faces == 0 && csmesh.CurvesGraphics.Count == 0)) return;
else
{
    CSVectorD3 item;
    item.x = 0.0;
    item.y = 0.0;
    item.z = 0.0;
    List<CSMesh> list = new List<CSMesh>(1);
    List<CSVectorD3> list2 = new List<CSVectorD3>(1);
    list2.Add(item);
    list.Add(csmesh);
    this.m_publisher.PublishGraphics(list, list2, num , -1, null, null, null, 0);
}
```

**Примечание**: если объект представляет собой набор объектов (например, армирование или Блок), то вложенные объекты должны публиковаться рекурсивно.

```csharp
//Вложенные элементы
List<CSMesh> list = new List<CSMesh>();
List<CSElement> list2 = new List<CSElement>();

int buildingOwnerObjId = MsBinaryPublisher.BuildingOwnerObjId;
MsBinaryPublisher.CommonOwnerObjId = num;
for (int i = 0; i < list2.Count; i++)
{
    num = this.m_publisher.PublishElement(list2[i], "Nested item", 1);
    this.PublishMesh(list[i], num);
}
MsBinaryPublisher.CommonOwnerObjId = buildingOwnerObjId;
```

**Шаг 7**- Завершение публикации: вызов`MsBinaryPublisher.CommitPublication()` и `MsBinaryPublisher.RunCommitUtility()`;

**Шаг 8**- Отключение от публикатора -- `MsBinaryPublisher.Disconnect()` и от целевой БД `CAD3DLibrary.Disconnect`, если речь про экспортер вне CADLib;
