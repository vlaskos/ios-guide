- [Data](#data)
	- [Что такое Core Data?](#core-data)
	- [В каких случаях лучше использовать SQLite, а в каких Core Data?](#sqlite-core-data)
	- [Целесообразность использования Core Data](#целесообразность-использования-core-data)
	- [Какие есть нюансы при использовании Core Data в разных потоках? Как синхронизировать данные между потоками?](#core-data-в-разных-потоках)
	- [Какие типы хранилищ поддерживает CoreData?](#типы-хранилищ)
	- [Что такое ленивая загрузка? Что ее связывает с Core Data? Опишите ситуация когда она может быть полезной? Что такое faulting?](#ленивая-загрузка-core-data)
	- [Что такое fetch result controller?](#fetch-result-controller)
	- [Как сделать миграцию БД?](#db-migration)

<a name="data"></a>
# Data
<a name="core-data"></a>
## Что такое Core Data?
Apple предоставляет гибкий фреймворк для работы с хранимыми на устройстве данными — Core Data. Большинство деталей по работе с хранилищем данных Core Data скрывает, позволяя вам сконцентрироваться на том, что действительно делает ваше приложение веселым, уникальным и удобным в использовании. Не смотря на то, что Core Data может хранить данные в реляционной базе данных вроде SQLite, Core Data не является СУБД (системой управления БД). По-правде, Core Data в качестве хранилища может вообще не использовать реляционные базы данных. Core Data скорее является оболочкой для работы с данными, которая позволяет работать с сущностями и их связями (отношениями к другим объектами), атрибутами, в том виде, который напоминает работы с объектным графом в обычном объектно-ориентированном программировании.

<img src="https://github.com/sashakid/ios-guide/blob/master/Images/nsmanagedobjectcontext.png">

_NSManagedObjectModel_

The `NSManagedObjectModel` instance describes the data that is going to be accessed by the Core Data stack. During the creation of the Core Data stack, the `NSManagedObjectModel` (often referred to as the “mom”) is loaded into memory as the first step in the creation of the stack. The example code above resolves an `NSURL` from the main application bundle using a known filename (in this example `DataModel.momd`) for the `NSManagedObjectModel`. Once the `NSManagedObjectModel` object is initialized, the `NSPersistentStoreCoordinator` object is constructed.

_NSPersistentStoreCoordinator_

The `NSPersistentStoreCoordinator` sits in the middle of the Core Data stack. The coordinator is responsible for realizing instances of entities that are defined inside of the model. It creates new instances of the entities in the model, and it retrieves existing instances from a persistent store (`NSPersistentStore`). The persistent store can be on disk or in memory. Depending on the structure of the application, it is possible, although uncommon, to have more than one persistent store being coordinated by the `NSPersistentStoreCoordinator`.

Whereas the `NSManagedObjectModel` defines the structure of the data, the `NSPersistentStoreCoordinator` realizes objects from the data in the persistent store and passes those objects off to the requesting `NSManagedObjectContext`. The `NSPersistentStoreCoordinator` also verifies that the data is in a consistent state that matches the definitions in the `NSManagedObjectModel`.

Координатор предназначен для представления фасада для управляемого контекстом объекта, так что группа постоянных хранилищ выглядит как единое совокупное хранилище. Управляемый объект контекста может создать граф объектов на основе объединения всех хранилищ данных координатора. Координатор может быть связан только с одной управляемой объектной моделью. Если Вы хотите положить различные субъекты в различные хранилища, вы должны разделить вашу модель, определяя конфигурацию в управляемой модели объекта.

_NSManagedObjectContext_

The managed object context (`NSManagedObjectContext`) is the object that your application will be interacting with the most, and therefore it is the one that is exposed to the rest of your application. Think of the managed object context as an intelligent scratch pad. When you fetch objects from a persistent store, you bring temporary copies onto the scratch pad where they form an object graph (or a collection of object graphs). You can then modify those objects however you like. Unless you actually save those changes, however, the persistent store remains unaltered.

All managed objects must be registered with a managed object context. You use the context to add objects to the object graph and remove objects from the object graph. The context tracks the changes you make, both to individual objects’ attributes and to the relationships between objects. By tracking changes, the context is able to provide undo and redo support for you. It also ensures that if you change relationships between objects, the integrity of the object graph is maintained.

If you choose to save the changes you have made, the context ensures that your objects are in a valid state. If they are, the changes are written to the persistent store (or stores), new records are added for objects you created, and records are removed for objects you deleted.

Without Core Data, you have to write methods to support archiving and unarchiving of data, to keep track of model objects, and to interact with an undo manager to support undo. In the Core Data framework, most of this functionality is provided for you automatically, primarily through the managed object context.

<a name="sqlite-core-data"></a>
## В каких случаях лучше использовать SQLite, а в каких Core Data?
_Реляционная база данных_ — база данных, основанная на реляционной модели данных. Слово «реляционный» происходит от англ. relation (отношение). Для работы с реляционными БД применяют реляционные СУБД.

_Реляционная модель данных (РМД)_ — логическая модель данных, прикладная теория построения баз данных, которая является приложением к задачам обработки данных таких разделов математики как теории множеств и логика первого порядка. Термин «реляционный» означает, что теория основана на математическом понятии отношение (relation). В качестве неформального синонима термину «отношение» часто встречается слово таблица. Необходимо помнить, что «таблица» есть понятие нестрогое и неформальное и часто означает не «отношение» как абстрактное понятие, а визуальное представление отношения на бумаге или экране. Некорректное и нестрогое использование термина «таблица» вместо термина «отношение» нередко приводит к недопониманию. Наиболее частая ошибка состоит в рассуждениях о том, что РМД имеет дело с «плоскими», или «двумерными» таблицами, тогда как таковыми могут быть только визуальные представления таблиц. Отношения же являются абстракциями, и не могут быть ни «плоскими», ни «неплоскими».

_ORM_ – Object-relational mapping, объектно-реляционное отображение – преобразование данных между ООП и реляционными базами данных.

_SQL_ – Структурированный язык запросов (Structed Query Language) – является информационно-логическим языком, предназначенным для описания, изменения и извлечения данных, хранимых в реляционных базах данных (relation, отношение).

__Пример SQL:__

Create a table to store information about weather observation stations:

-- No duplicate ID fields allowed
```sql
CREATE TABLE STATION
(ID INTEGER PRIMARY KEY,
CITY CHAR(20),
STATE CHAR(2),
LAT_N REAL,
LONG_W REAL);
```
populate the table STATION with a few rows:
```sql
INSERT INTO STATION VALUES (13, 'Phoenix', 'AZ', 33, 112);
INSERT INTO STATION VALUES (44, 'Denver', 'CO', 40, 105);
INSERT INTO STATION VALUES (66, 'Caribou', 'ME', 47, 68);
```
query to look at table STATION in undefined order:
```sql
SELECT * FROM STATION;
```
ID | CITY | STATE | LAT_N | LONG_W
---|------|-------|-------|-------
13 | Phoenix | AZ | 33 | 112
44 | Denver | CO | 40 | 105
66 | Caribou | ME | 47 | 68

query to select Northern stations (Northern latitude > 39.7):

-- selecting only certain rows is called a "restriction".
```sql
SELECT * FROM STATION
WHERE LAT_N > 39.7;
```
ID | CITY | STATE | LAT_N | LONG_W
---|------|-------|-------|-------
44 | Denver | CO | 40 | 105
66 | Caribou | ME | 47 | 68

query to select only ID, CITY, and STATE columns:

-- selecting only certain columns is called a "projection".
```sql
SELECT ID, CITY, STATE FROM STATION;
```
ID | CITY | STATE
---|------|------
13 | Phoenix | AZ
44 | Denver | CO
66 | Caribou | ME

query to both "restrict" and "project":
```sql
SELECT ID, CITY, STATE FROM STATION
WHERE LAT_N > 39.7;
```
ID | CITY | STATE
---|------|------
44 | Denver | CO
66 | Caribou | ME

<a name="целесообразность-использования-core-data"></a>
## Целесообразность использования Core Data
Core Data уменьшает количество кода, написанного для поддержки модели слоя приложения, как правило, на 50% - 70%, измеряемое в строках кода. Core Data имеет зрелый код, качество которого обеспечивается путем юнит-тестов, и используется ежедневно миллионами клиентов в широком спектре приложений. Структура была оптимизирована в течение нескольких версий. Она использует информацию, содержащуюся в модели и выполненяет функции, как правило, не работающие на уровне приложений в коде. Кроме того, в дополнение к отличной безопасности и обработке ошибок, она предлагает лучшую масштабируемость при работе с памятью, относительно любого конкурирующего решения. Другими словами: вы могли бы потратить долгое время тщательно обрабатывая Ваши собственные решения оптимизации для конкретной предметной области, вместо того, чтобы получить преимущество в производительности, которую Core Data предоставляет бесплатно для любого приложения.

__Когда нецелесообразно использовать Core Data:__
* если планируется использовать очень небольшой объем данных. В этом случае проще воспользоваться для хранения Ваших данных объектами коллекций - массивами или словарями и сохранять их в .plist файлы.
* если используется кросс-платформерная архитектура или требуется доступ к строго определенному формату файла с данными (хранилищу), например SQLite.
* использование баз данных клиент-сервер, например MySQL или PostgreSQL.

__SQLite__
* Максимальный объем хранимых данных базы SQLite составляет 2 терабайта.
* Чтение из базы данных может производиться одним и более потоками, например несколько процессов могут одновременно выполнять `SELECT`. Однако запись в базу данных может осуществляться, только, если база в данный момент не занята другим процессом.
* SQLite не накладывает ограничения на типы данных. Любые данные могут быть занесены в любой столбец. Ограничения по типам данных действуют только на `INTEGER PRIMARY KEY`, который может содержать только 64-битное знаковое целое.

SQLite версии 3.0 и выше позволяет хранить BLOB данные в любом поле, даже если оно объявлено как поле другого типа.
Обращение к SQLite базе из двух потоков одновременно неизбежно вызовет краш. Выхода два:

1. Синхронизируйте обращения при помощи директивы `@synchronized`.
2. Если задача закладывается на этапе проектирования, завести менеджер запросов на основе `NSOperationQueue`. Он страхует от ошибок автоматически, а то, что делается автоматически, часто делается без ошибок.

__Пример SQLite__

```objectivec
- (int)createTable:(NSString *)filePath {
  sqlite3 *db = NULL;
  int rc = 0;

  rc = sqlite3_open_v2([filePath cStringUsingEncoding:NSUTF8StringEncoding], &db, SQLITE_OPEN_READWRITE | SQLITE_OPEN_CREATE, NULL);
  if (SQLITE_OK != rc) {
    sqlite3_close(db);
    NSLog(@"Failed to open db connection");
  } else {
    char *query ="CREATE TABLE IF NOT EXISTS students (id INTEGER PRIMARY KEY AUTOINCREMENT, name  TEXT, age INTEGER, marks INTEGER)";
    char *errMsg;
    rc = sqlite3_exec(db, query, NULL, NULL, &errMsg);
    if (SQLITE_OK != rc) {
      NSLog(@"Failed to create table rc:%d, msg=%s",rc,errMsg);
    }
    sqlite3_close(db);
  }
  return rc;
}

- (int)insert:(NSString *)filePath withName:(NSString *)name age:(NSInteger)age marks:(NSInteger)marks {
  sqlite3 *db = NULL;
  int rc = 0;
  rc = sqlite3_open_v2([filePath cStringUsingEncoding:NSUTF8StringEncoding], &db, SQLITE_OPEN_READWRITE, NULL);
  if (SQLITE_OK != rc) {
    sqlite3_close(db);
    NSLog(@"Failed to open db connection");
  } else {
    NSString *query  = [NSString stringWithFormat:@"INSERT INTO students (name, age, marks) VALUES (\"%@\", %ld, %ld)", name, (long)age, (long)marks];
    char *errMsg;
    rc = sqlite3_exec(db, [query UTF8String], NULL, NULL, &errMsg);
    if (SQLITE_OK != rc) {
      NSLog(@"Failed to insert record  rc:%d, msg=%s",rc,errMsg);
    }
    sqlite3_close(db);
  }
  return rc;
}
```

<a name="core-data-в-разных-потоках"></a>
## Какие есть нюансы при использовании Core Data в разных потоках? Как синхронизировать данные между потоками?
In Core Data, the managed object context can be used with two concurrency patterns, defined by `NSMainQueueConcurrencyType` and `NSPrivateQueueConcurrencyType`.

* `NSMainQueueConcurrencyType` is specifically for use with your application interface and can only be used on the main queue of an application.

* The `NSPrivateQueueConcurrencyType` configuration creates its own queue upon initialization and can be used only on that queue. Because the queue is private and internal to the `NSManagedObjectContext` instance, it can only be accessed through the `performBlock:` and the `performBlockAndWait:` methods.

`NSManagedObject` instances are not intended to be passed between queues. Doing so can result in corruption of the data and termination of the application. When it is necessary to hand off a managed object reference from one queue to another, it must be done through `NSManagedObjectID` instances. You retrieve the managed object ID of a managed object by calling the `objectID` method on the `NSManagedObject` instance.

<a name="типы-хранилищ"></a>
# Какие типы хранилищ поддерживает CoreData?
Persistent Store

* SQLite
* Binary
* XML

Atomic Store

* custom type

<a name="ленивая-загрузка-core-data"></a>
## Что такое ленивая загрузка? Что ее связывает с Core Data? Опишите ситуация когда она может быть полезной? Что такое faulting?
Для загрузки данных из БД в память приложения удобно пользоваться загрузкой не только данных об объекте, но и о сопряжённых с ним объектах. Это делает загрузку данных проще для разработчика: он просто использует объект, который, тем не менее вынужден загружать все данные в явном виде. Но это ведёт к случаям, когда будет загружаться огромное количество сопряжённых объектов, что плохо скажется на производительности в случаях, когда эти данные реально не нужны. Паттерн Lazy Loading (Ленивая Загрузка) подразумевает отказ от загрузки дополнительных данных, когда в этом нет необходимости. Вместо этого ставится маркер о том, что данные не загружены и их надо загрузить в случае, если они понадобятся. Как известно, если Вы ленивы, то вы выигрываете в том случае, если дело, которое вы не делали на самом деле и не надо было делать.
Faulting isn't unique to Core Data. A similar technique is used in many other frameworks, such as Ember and Ruby on Rails. Faulting is a mechanism Core Data employs to reduce your application’s memory usage, only load data when it's needed. A fault is a placeholder object that represents a managed object that has not yet been fully realized, or a collection object that represents a relationship. To make faulting work, Core Data does a bit of magic under the hood by creating custom subclasses at compile time that represent the faults.

<img src="https://github.com/sashakid/ios-guide/blob/master/Images/core_data_faulting.png">

<a name="fetch-result-controller"></a>
## Что такое fetch result controller?
Данные сами по себе может быть и представляют какую-либо ценность, но, обычно их нужно использовать. Одним из элементов представления данных в iOS служат таблицы (объекты класса `UITableView`), которые через объект класса `NSFetchedResultsController` можно привязать к CoreData. После этого при изменении данных в CoreData будет актуализироваться информация в таблице. Так же, с помощью таблицы можно управлять данными в хранилище.
`NSFetchedResultsController` — контроллер результатов выборки. Создается, обычно один экземпляр на `ViewController`, но вполне может работать и без оного, внутрь которого помещается исключительно для того, что бы было проще привязать данные к виду.

<a name="db-migration"></a>
## Как сделать миграцию БД?

You can only open a Core Data store using the managed object model used to create it. Changing a model will therefore make it incompatible with (and so unable to open) the stores it previously created. If you change your model, you therefore need to change the data in existing stores to new version—changing the store format is known as migration. To migrate a store, you need both the version of the model used to create it, and the current version of the model you want to migrate to. You can create a versioned model that contains more than one version of a managed object model. Within the versioned model you mark one version as being the current version. Core Data can then use this model to open persistent stores created using any of the model versions, and migrate the stores to the current version. To help Core Data perform the migration, though, you may have to provide information about how to map from one version of the model to another. This information may be in the form of hints within the versioned model itself, or in a separate mapping model file that you create.

The migration process itself is in three stages. It uses a copy of the source and destination models in which the validation rules are disabled and the class of all entities is changed to `NSManagedObject`.

To perform the migration, Core Data sets up two stacks, one for the source store and one for the destination store. Core Data then processes each entity mapping in the mapping model in turn. It fetches objects of the current entity into the source stack, creates the corresponding objects in the destination stack, then recreates relationships between destination objects in a second stage, before finally applying validation constraints in the final stage.

Before a cycle starts, the entity migration policy responsible for the current entity is sent a `beginEntityMapping:manager:error:` message. You can override this method to perform any initialization the policy requires. The process then proceeds as follows:

1. Create destination instances based on source instances.

At the beginning of this phase, the entity migration policy is sent a `createDestinationInstancesForSourceInstance:entityMapping:manager:error:` message; at the end it is sent a `endInstanceCreationForEntityMapping:manager:error:` message. In this stage, only attributes (not relationships) are set in the destination objects. Instances of the source entity are fetched. For each instance, appropriate instances of the destination entity are created (typically there is only one) and their attributes populated (for trivial cases, `name = $source.name`). A record is kept of the instances per entity mapping since this may be useful in the second stage.

2. Recreate relationships.

At the beginning of this phase, the entity migration policy is sent a `createRelationshipsForDestinationInstance:entityMapping:manager:error:` message; at the end it is sent a `endRelationshipCreationForEntityMapping:manager:error:` message. For each entity mapping (in order), for each destination instance created in the first step any relationships are recreated.

3. Validate and save.

In this phase, the entity migration policy is sent a `performCustomValidationForEntityMapping:manager:error:` message. Validation rules in the destination model are applied to ensure data integrity and consistency, and then the store is saved.

At the end of the cycle, the entity migration policy is sent an endEntityMapping:manager:error: message. You can override this method to perform any clean-up the policy needs to do. Note that Core Data cannot simply fetch objects into the source stack and insert them into the destination stack, the objects must be re-created in the new stack. Core Data maintains “association tables” which tell it which object in the destination store is the migrated version of which object in the source store, and vice-versa. Moreover, because it doesn't have a means to flush the contexts it is working with, you may accumulate many objects in the migration manager as the migration progresses. If this presents a significant memory overhead and hence gives rise to performance problems, you can customize the process as described in Multiple Passes—Dealing With Large Datasets.

__Lightweight Migration__

If you just make simple changes to your model (such as adding a new attribute to an entity), Core Data can perform automatic data migration, referred to as lightweight migration. Lightweight migration is fundamentally the same as ordinary migration, except that instead of you providing a mapping model (as described in Mapping Overview), Core Data infers one from differences between the source and destination managed object models.

Lightweight migration is especially convenient during early stages of application development, when you may be changing your managed object model frequently, but you don’t want to have to keep regenerating test data. You can migrate existing data without having to create a custom mapping model for every model version used to create a store that would need to be migrated.

A further advantage of using lightweight migration—beyond the fact that you don’t need to create the mapping model yourself—is that if you use an inferred model and you use the SQLite store, then Core Data can perform the migration in situ (solely by issuing SQL statements). This can represent a significant performance benefit as Core Data doesn’t have to load any of your data. Because of this, you are encouraged to use inferred migration where possible, even if the mapping model you might create yourself would be trivial.
