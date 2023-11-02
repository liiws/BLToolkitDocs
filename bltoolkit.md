# Business Logic Toolkit

Основная задача статьи дать вводную информацию о библиотеке BLToolkit. Она поможет Вам сделать первый шаг в обуздании этого маленького монстра. Данная статья основана на оригинальной статье [Игоря Ткачева "Пространство имен Rsdn.Framework.Data"](http://rsdn.ru/article/files/libs/RsdnFrameworkData.xml) и, как следствие, содержит части оной.

[Оф. сайт BLToolkit](http://bltoolkit.net/Home.ashx).



## От [liiws](https://github.com/liiws)

Настоящая копипаста создана с целью сохранения оригинального текста, т.к. страница на [RSDN](http://projects.rsdn.ru/RFD/wiki/BLToolkit) умерла.



## Оглавление

- Business Logic Toolkit
    - [Введение](#intro)
        - [Те же и Oracle](#someoracle)
    - [Зачем все это надо](#why)
        - [Немного истории](#history)
        - [За что боролись отцы и деды](#parents)
    - [Класс DbManager](#dbmanager)
        - [Инициализация и создание экземпляра объекта](#init)
        - [Механизм дата провайдеров](#dataprovider)
        - [Параметры](#parameters)
        - [Методы Execute](#executemethods)
            - [ExecuteDataSet](#executedataset)
            - [ExecuteDataTable и ExecuteDataTables](#executedatatable)
            - [ExecuteReader](#executereader)
            - [ExecuteNonQuery](#executenonquery)
            - [ExecuteScalar](#executescalar)
            - [ExecuteScalarList](#executescalarlist)
            - [ExecuteScalarDictionary](#executescalardictionary)
            - [ExecuteObject](#executeobject)
            - [ExecuteList](#executelist)
            - [ExecuteDictionary](#executedictionary)
            - [ExecuteForEach](#executeforeach)
            - [ExecuteResultSets](#executeresultsets)
    - [Отображение данных](#mapping)
        - [Общие сведения](#mappinggeneral)
        - [Map & MappingSchema](#mapandschema)
        - [Семейства функций MappingSchema](#mappingschema)
            - [Семейства Convert](#mappingconvert)
            - [Семейства Map](#mappingmap)
            - [MemberMapper](#mappingmapper)
        - [Метаданные](#metadata)
            - [Атрибуты](#attributes)
            - [XML](#xml)
    - [DataAccess](#dataaccess)
        - [SqlQuery](#sqlquery)
        - [DataAcessor](#dataaccessor)
            - [Возвращаемое значение](#dataaccessorreturntype)
            - [Имя метода](#dataaccessormethodname)
            - [Параметры](#dataaccessorparams)
            - [Атрибуты](#dataaccessorattributes)
        - [Рекомендации по использванию](#dataaccessrecommends)
            - [Реализация методов ручками](#dataaccessrecommendsmanual)
            - [Генерация SQL запросов в абстрактном аксессоре](#dataaccessrecommendsabstract)
    - [При написании использовалось](#materials)





<span id="intro"></span>
## Введение

*BLToolkit* (ранее известна как *Rsdn.Framework.Data*) является библиотекой, содержащей набор классов, представляющих собой высокоуровневую обёртку над ADO.NET (вообще, это не совсем правда, содержит она гораздо больше, но исторически BLT создавалась как раз для этих целей). Казалось бы, ADO.NET сама по себе штука достаточно высокоуровневая и зачем над ней ещё городить какой-то огород? Всё это так, но как это часто бывает, в борьбе добра со злом обычно, увы, побеждает лень.

Рассмотрим в качестве примера функцию, которая возвращает список объектов, содержащих информацию о людях: ID, имя, фамилию, отчество и пол (здесь и далее мы будем использовать базу данных BLToolkit – это тестовая БД к BLT, на данной базе работают блочные тесты, здесь и далее для примеров я постараюсь использовать именно блочные тесты от BLT). Вот как это может выглядеть с использованием ADO.NET:


Таблица Person имеет следующий вид:

```sql
CREATE TABLE Person
(
	PersonID   int          NOT NULL IDENTITY(1,1) CONSTRAINT PK_Person PRIMARY KEY CLUSTERED,
	FirstName  nvarchar(50) NOT NULL,
	LastName   nvarchar(50) NOT NULL,
	MiddleName nvarchar(50)     NULL,
	Gender     char(1)      NOT NULL CONSTRAINT CK_Person_Gender CHECK (Gender in ('M', 'F', 'U', 'O'))
)
ON [PRIMARY]
```

Код:

```csharp
public enum Gender
{
	Female,
	Male,
	Unknown,
	Other
}
public class Person
{
	public int    ID;
	public string FirstName;
	public string MiddleName;
	public string LastName;
	public Gender Gender;
}

Person GetPerson(int personId)
{
    string connectionString =
        "Server=.;Database=BLToolkit;Integrated Security=SSPI";
    string commandText = @"
        SELECT 
            p.PersonId,
            p.FirstName,
            p.SecondName,
            p.MiddleName,
            p.Gender
        FROM Person p
        WHERE p.PersonId = @PersonId";

    using (SqlConnection con = new SqlConnection(connectionString))
    {
        con.Open();

        using (SqlCommand cmd = new SqlCommand(commandText, con))
        {
            cmd.Parameters.Add("@min", min);

            using (SqlDataReader rd = cmd.ExecuteReader())
            {
                Person p = null;

                if (rd.Read())
                {
                    p = new Person();

                    p.ID           = Convert.ToInt32 (rd["PersonId"]);
                    p.FirstName    = Convert.ToString(rd["FirstName"]);
                    p.SecondName   = Convert.ToString(rd["SecondName"]);
                    p.MiddleName   = Convert.ToString(rd["ThirdName"]);
                    string gender  = Convert.ToString(rd["Gender"]);

                    switch(gender)
                    {
                        case "M":
                            p.Gender = Gender.Male;
                            break;
                        case "F":
                            p.Gender = Gender.Female;
                            break;
                        case "U":
                            p.Gender = Gender.Unknown;
                            break;
                        case "0":
                            p.Gender = Gender.Other;
                            break;
                    }
                }
                return p;
            }
        }
    }
}
```

А теперь то же самое, в исполнении BLToolkit:

```csharp
public enum Gender
{
	[MapValue("F")] Female,
	[MapValue("M")] Male,
	[MapValue("U")] Unknown,
	[MapValue("O")] Other
}
public class Person
{
	[MapField("PersonID")]
	public int    ID;
	public string FirstName;
	public string MiddleName;
	public string LastName;
	public Gender Gender;
}

Person GetPerson(int personId)
{
    using (DbManager db = new DbManager())
    {
        return db
            .SetCommand@"
                         SELECT 
                             p.PersonId,
                             p.FirstName,
                             p.SecondName,
                             p.MiddleName,
                             p.Gender
                         FROM Person p
                         WHERE p.PersonId = @PersonId",
                db.Parameter("@PersonId", personId)
            .ExecuteObject(typeof(Person));
    }
}
```



<span id="someoracle"></span>
### Те же и Oracle

Если вы начинаете тренироваться с BLToolkit на базе Oracle, последний фрагмент кода должен иметь вид: 

```csharp
Person GetPerson(int personId)
{
    using (DbManager db = new DbManager())
    {
        return db
            .SetCommand( @"
               SELECT 
                  p.PersonId,
                  p.FirstName,
                  p.LastName,
                  p.MiddleName,
                  p.Gender
               FROM Person p
               WHERE p.PersonId = :PersonId",
            db.Parameter("PersonId", personId))
            .ExecuteObject<Person>();
    }
}
...
var p = new BLToolkit.Data.DataProvider.OracleDataProvider();
p.ParameterPrefix = null;
DbManager.AddDataProvider(p);
DbManager.AddConnectionString("Oracle",
string.Format("Data Source={0};User ID={1};Password={2};",
oracleAlias, oracleUser, oraclePassword));


var person = GetPerson(1);
```


Не трудно заметить, что последний вариант заметно короче. Фактически все, что у нас осталось – это текст самого запроса. Класс DbManager самостоятельно осуществляет всю работу по созданию объекта и отображению (mapping) полей рекордсета на заданную структуру.

Вообще, забегая вперед добиться тех же успехов можно и более лаконичным (и техничным) путем:

```csharp
public abstract class PersonAccessor<Person, PersonAccessor>
{
    [SqlQuery@"
               SELECT 
                   p.PersonId,
                   p.FirstName,
                   p.SecondName,
                   p.MiddleName,
                   p.Gender
               FROM Person p
               WHERE p.PersonId = @PersonId")]
    public abstract Person GetPerson(int personId){}
}

//и уже где-то совсем в другом месте программы
//получим нужного человека:

Person p = PersonAccessor.CreateInstance().GetPerson (10);

//...
```

Но, давайте обо всём по порядку.



<span id="why"></span>
## Зачем всё это надо


<span id="history"></span>
### Немного истории

В конечном итоге все упирается в фундаментальные проблемы: в коде мы имеем дело с классами и объектами, в реляционной БД мы имеем дело с таблицами и их отношениями – с данными.

Две противоположные и одновременно распространенные модели – объектная и реляционная для совместно использования требуют некоторой прослойки позволяющей производить их взаимное отображение.

Подробнее с проблемами, возникающими вокруг ORM можно ознакомиться у [Мартина Фаулера ("Архитектура корпоративных программных приложений")](http://rsdn.ru/res/book/prog/architect.xml).

BLToolkit является маленькой и шустрой системой, позволяющей отображать данные на объекты, а объекты на данные. Библиотека может стать удобным подспорьем для реализации любого паттерна от “Шлюза таблицы данных (Table Data Gateway)” до “Преобразователя данных (Data Mapper)” (термины приведены в соответствии с г-ном Фаулером).

Если посмотреть со внешней стороны, то можно выделить следующих ведущих игроков (именно о них мы будем говорить ниже):

- DbManager – класс, предоставляющий высокоуровневую обертку над ADO .NET.
- Map & MappingSchema – первый класс является статическим, и делегирует свои вызовы к экземпляру MappingSchema. MappingSchema, в свою очередь, обеспечивает отображение ежа на ужа, а ужа на слона.
- DataAccessor<T, D> - ода лени. Позволяет отделить мух от котлет – организовать уровень абстракции объектов от данных и способа их извлечения и сохранения.
- Метаданные – мощный и удобный механизм .Net, предоставляющий возможность описать представление объекта в БД, не внося изменений в открытый интерфейс класса.



<span id="parents"></span>
### За что боролись отцы и деды

А боролись они за высокую производительность, как программиста, так и BLT. Фактически, большая часть потрохов BLT это генераторы абстрактных классов: библиотека «на лету» эмитит небольшие классы, что позволяет избавить программиста от рутинной работы с одной стороны, и повысить производительность с другой.

Таким образом, BLT добавляет некоторые издержки на момент первого вызова, связанные с эмитом и кэшированием. Все последующие вызовы являются максимально оптимальными. Издержки же на отображение настолько низки, что ими можно пренебречь (для примера можно посмотреть [здесь](http://rsdn.ru/forum/info/FAQ.rfd.rfdprojects)).



<span id="dbmanager"></span>
## Класс DbManager


<span id="init"></span>
### Инициализация и создание экземпляра объекта

Класс DbManager является основным в пространстве имен BLToolkit.Data и в единственном лице представляет собой замену всем основным объектам ADO.NET.

Для создания экземпляра объекта служит целый набор конструкторов:

```csharp
public DbManager();

public DbManager(
    string configurationString
    );

public DbManager(
    string providerName, 
    string configuration
    );

public DbManager(
    IDbConnection connection
    );

public DbManager(
    IDbTransaction transaction
    );

public DbManager(
    DataProviderBase dataProvider, 
    string           connectionString
    );

public DbManager(
    DataProviderBase dataProvider, 
    IDbConnection    connection
    );

public DbManager(
    DataProviderBase dataProvider, 
    IDbTransaction   transaction
    );
```

Остановимся подробней на следующих параметрах (прочие параметры не должны вызвать вопросов у тех, кто хотя бы поверхностно знаком с ADO .NET):

- *configurationString* – это не строка соединения (Connecting String), а ключ, по которому строка соединения будет читаться из файла конфигурации.
- *providerName* – так же ключ, является именем дата провайдера, который следует использовать.
- *dataProvider* – экземпляр дата провайдера, который следует использовать. Механизм дата провайдеров достоин отдельного обсуждения, что и будет сделано ниже.

Рассмотрим подробнее правила работы с файлом конфигурации:

```csharp
<appSettings>
<!—- Конфигурация по умолчанию -->
<add
		key   = "ConnectionString"
		value = "Server=.;Database=BLToolkitData;Integrated Security=SSPI"/>
<!—- Конфигурация Development для SQL Server -->
<add
		key   = "ConnectionString.Development"
		value = "Server=.;Database=BLToolkitData;Integrated Security=SSPI"/>
<!-- Конфигурация Production для SQL Server -->
<add
		key   = "ConnectionString.Production"
		value = "Server=.;Database=BLToolkitData;Integrated Security=SSPI"/>
<!-- Конфигурация для SQL Server -->
<add
		key   = "ConnectionString.Sql"
		value = "Server=.;Database=BLToolkitData;Integrated Security=SSPI"/>
<!-- Конфигурация для Oracle -->
<add
		key   = "ConnectionString.Oracle"
		value = "User Id=/;Data Source=BLToolkitData"/>
<!-- Конфигурация OLEDB -->
<add
		key   = "ConnectionString.OleDb"
		value = "Provider=SQLNCLI.1;Data Source=.;Integrated Security=SSPI;Initial Catalog=BLToolkitData"/>
<!-- Конфигурация Development для OLEDB -->
<add
		key   = "ConnectionString.OleDb.Development"
		value = "Provider=SQLNCLI.1;Data Source=.;Integrated Security=SSPI;Initial Catalog=BLToolkitData"/>
<!-- Конфигурация Production для OLEDB -->
<add
		key   = "ConnectionString.OleDb.Production"
		value = "Provider=SQLNCLI.1;Data Source=.;Integrated Security=SSPI;Initial Catalog=BLToolkitData"/>
</appSettings>
```

Как видим, поле key – содержит ключевое значение ConntctionString и разделенные c ним через точку *configurationString* и *providerName*.

Рассмотрим следующие примеры создания DbManager:

```csharp
// Использование конфигурации по умолчанию.
DbManager db = new DbManager();

// Использование конфигурации для Sql Server
// аналогично для Oracle DbManager ("Oracle")
DbManager db = new DbManager("Sql");

// Использование конфигурации Development для Sql Server
// аналогично Production для OLEDB DbManager ("OleDb" & "Production")
DbManager db = new DbManager("Sql", "Development");
```

Дополнительно есть возможность указать конфигурацию по умолчанию. Если в файл конфигурации добавить вот такую секцию:

```csharp
<appSettings>
    <add
        key   = "BLToolkit.DefaultConfiguration"
        value = "Oracle"/>
</appSettings>
```

То вызов конструктора без параметров будет аналогичен вызову DbManager(“Oracle”). 

Таким образом мы можем работать с различными конфигурациями и базами данных. Секция *appSettings* может находиться как в *app.config* или *web.config* так и *machine.config* файле.

Если же вам не хочется возиться с конфигурационными файлами, то для задания строки соединения можно воспользоваться методом AddConnectionString:

```csharp
DbManager.AddConnectionString("MyConfig", connectionString);

using (DbManager db = new DbManager("MyConfig"))
{
    // ...
}
```

или

```csharp
DbManager.AddConnectionString(connectionString);

using (DbManager db = new DbManager())
{
    // ...
}
```

Метод AddConnectionString достаточно вызвать один раз для каждой конфигурации в начале программы.



<span id="dataprovider"></span>
### Механизм дата провайдеров

Отличительной особенностью класса DbManager является то, что он работает исключительно с интерфейсами пространства имён System.Data и вполне может использоваться для работы с различными провайдерами данных. На данный момент поддерживается работа с Data Provider for SQL Server, Data Provider for Oracle, Data Provider for OLE DB и Data Provider for ODBC. Выбор провайдера осуществляется также с помощью строки конфигурации. Для этого достаточно добавить к ней один из следующих постфиксов: “.OleDb”, “.Odbc”, “.Oracle”, “.Sql”. Если постфикс не задан, то по умолчанию выбирается провайдер для SQL Server.

К вопросам о мифичности и реальности поддержки в одном проекте различных баз данных. Исходный код BLToolkit покрыт блочными тестами. При этом есть возможность в качестве тестовой базы данных использовать: MS Sql Server, Access, Oracle (используется OdpDataProvider .\Source\Data\DataProvider\OdpDataProvider.cs), Firebird (используется FdpDataProvider .\Source\Data\DataProvider\FdpDataProvider.cs). Так же примером может служить проект [RSDN@HOME](http://rsdn.ru/projects/janus/article/article.xml), где механизмами BLT осуществлена поддержка нескольких БД.

Таким образом, механизм дата провайдеров позволяет абстрагировать DbManager от специфики конкретного клиента и его реализации. Для примера можно рассмотреть OdpDataProvider.

В дополнение к существующим провайдерам совсем несложно подключить любой другой. Следующий пример демонстрирует подключение Borland Data Providers for .NET (BDP.NET):

```csharp
using System;
using System.Data;
using System.Data.Common;

using Borland.Data.Provider;

using Rsdn.Framework.Data;
using Rsdn.Framework.Data.DataProvider;

namespace Example
{
    public class BdpDataProvider: IDataProvider
    {
        IDbConnection IDataProvider.CreateConnectionObject()
        {
            return new BdpConnection();
        }

        DbDataAdapter IDataProvider.CreateDataAdapterObject()
        {
            return new BdpDataAdapter();
        }

        void IDataProvider.DeriveParameters(IDbCommand command)
        {
            BdpCommandBuilder.DeriveParameters((BdpCommand)command);
        }

        Type IDataProvider.ConnectionType
        {
            get
            {
                return typeof(BdpConnection);
            }
        }

        string IDataProvider.Name
        {
            get
            {
                return "Bdp";
            }
        }
    }

    class Test
    {
        static void Main()
        {
            DbManager.AddDataProvider(new BdpDataProvider());
            DbManager.AddConnectionString(".bdp",
                "assembly=Borland.Data.Mssql,Version=1.1.0.0, " +
                "Culture=neutral,PublicKeyToken=91d62ebb5b0d1b1b;" +
                "vendorclient=sqloledb.dll;osauthentication=True;" +
                "database=Northwind;hostname=localhost;provider=MSSQL");

            using (DbManager db = new DbManager())
            {
                int count = (int)db
                    .SetCommand("SELECT Count(*) FROM Categories")
                    .ExecuteScalar();

                Console.WriteLine(count);
            }
        }
    }
}
```



<span id="parameters"></span>
### Параметры

Большинство используемых запросов требуют тот или иной набор параметров для своего выполнения. В приведённом выше примере таким параметром является @personId – идентификатор человека в базе. Зачастую, среднеленивый программист предпочитает использовать в подобных случаях обычную конкатенацию строк, т.е. что-то наподобие следующего:

```csharp
void Test(int id)
{
    string commandText = @"
        SELECT FirstName
        FROM   Person
        WHERE  PersonId = " + id;

    // ...
}
```

К сожалению, при всей своей простоте, такой стиль плохо читаем, часто ведёт к непредсказуемым ошибкам и долгим мучениям с подбором формата, если в качестве параметра, например, используется дата. Более того, если наш параметр имеет строковый тип, то применение такого подхода в Web-приложениях может сделать их весьма уязвимыми для хакерских атак. Поэтому, отложим шутки в сторону и серьёзно займёмся рассмотрением возможностей, предоставляемых классом DbManager для работы с параметрами.

Для создания параметров служит следующий набор методов:

```csharp
public IDbDataParameter Parameter(
	string parameterName,
	object value
);

public IDbDataParameter InputParameter(
	string parameterName,
	object value
);
```

Создаёт входной (*ParameterDirection.Input*) параметр с именем *parameterName* и значением *value*.

```csharp
public IDbDataParameter NullParameter(
    string parameterName,
    object value
);
```

Делает тоже, что и предыдущие методы и в дополнение проверяет значение *value*. Если оно представляет собой *null*, пустую строку, значение даты *DateTime.MinValue* или 0 для целых типов, то вместо заданного значения подставляется *DBNull.Value*.

```csharp
public IDbDataParameter OutputParameter(
    string parameterName,
    object value
);
```

Создаёт выходной (*ParameterDirection.Output*) параметр.

```csharp
public IDbDataParameter InputOutputParameter(
    string parameterName,
    object value
);
```

Создаёт параметр, работающий как входной и выходной (*ParameterDirection.InputOutput*).

```csharp
public IDbDataParameter ReturnValue(
    string parameterName
);
```

Создаёт параметр-возвращаемое значение (*ParameterDirection.ReturnValue*).

```csharp
public IDbDataParameter Parameter(
    ParameterDirection parameterDirection,
    string parameterName,
    object value
);
```

Создаёт параметр с заданными значениями.

Создание выходных параметров и возвращаемое значение используются для работы с сохранёнными процедурами. Входной параметр можно использовать для построения любых запросов.

Для чтения выходных параметров после выполнения запроса служит следующий метод:

```csharp
public IDbDataParameter Parameter(
    string parameterName
);
```

Каждая версия метода Execute… имеет в своём составе метод, принимающий в качестве последнего аргумента список параметров запроса. Например, для ExecuteNonQuery одна из таких функций имеет следующий вид:

```csharp
public int ExecuteNonQuery(
    string commandText,
    params IDbDataParameter[] commandParameters
);
```

Таким образом, список параметров задаётся простым перечислением через запятую (с таблицей Region и примерами с ней связанными я отойду от правила использовать БД BLToolkit):

```csharp
void InsertRegion(int id, string description)
{
    using (DbManager db = new DbManager())
    {
        db
            .SetCommand(@"
                INSERT INTO Region (
                    RegionID,
                    RegionDescription
                ) VALUES (
                    @id,
                    @desc
                )",
                db.Parameter("@id",   id),
                db.Parameter("@desc", description))
            .ExecuteNonQuery();
    }
}
```

Для создания списка параметров из бизнес объектов существует метод CreateParameters, который принимает в качестве аргумента объект DataRow или любой бизнес-объект. Допустим, у нас имеется класс Region, содержащий информацию о регионе. В этом случае мы могли бы переписать предыдущий пример следующим образом:

```csharp
public class Region
{
    public int    ID;
    public string Description;
}

void InsertRegion(Region region)
{
    using (DbManager db = new DbManager())
    {
        db
            .SetCommand(@"
                INSERT INTO Region (
                    RegionID,
                    RegionDescription
                ) VALUES (
                    @ID,
                    @Description
                )",
                db.CreateParameters(region)).
            .ExecuteNonQuery();
    }
}
```

Более общий вид функции CreateParameters для бизнес объекта (аналогично для DataRow) выглядит следующим образом:

```csharp
public IDbDataParameter[] CreateParameters(
	object                    obj,
	string[]                  outputParameters,
	string[]                  inputOutputParameters,
	string[]                  ignoreParameters,
	params IDbDataParameter[] commandParameters);
```

Подобный вызов позволит явно задать параметрам  по их именам их направления и, при необходимости, указать дополнительные параметры:

```csharp
public class Region
{
    public int    ID;
    public string Description;
}

void InsertRegion(Region region)
{
    using (DbManager db = new DbManager())
    {
        db
            .SetCommand(@"
                INSERT INTO Region (
                    RegionDescription
                ) VALUES (
                    @Description
                )
                SELECT Cast(SCOPE_IDENTITY() as int) ID",
                db.CreateParameters(region, new string[]{"ID"}, null, null)).
            .ExecuteObject(region);
    }
}
```

В результате данного вызова объекту *region* в соответствующее поле будет задано значение *ID* только что вставленной записи (считаем, что поле *ID* в таблице *Region* – автоинкрементное).

Для передачи параметров сохранённой процедуре можно воспользоваться ещё одним способом, не требующим явного указания имён параметров:

```csharp
DataSet SelectByName(string firstName, string lastName)
{
    using (DbManager db = new DbManager())
    {
        return db
            .SetSpCommand("Person_SelectListByName", firstName, lastName)
            .ExecuteDataSet();
    }
}
```

В данном случае важен лишь порядок следования аргументов процедуры. Данная функция самостоятельно строит список параметров исходя из списка параметров сохранённой процедуры.

Для анализа возвращаемого значения и выходных параметров можно воспользоваться следующим методом:

```csharp
public IDbDataParameter Parameter(
    string parameterName
);
```

Например, в приведённом выше примере возвращаемое значение сохранённой процедуры можно (ну тут я слукавил – Person_SelectByName не возвращает такого значения, но если бы возвращала, то было бы можно) проверить следующим образом:

```csharp
DataSet SelectByName(string firstName, string lastName)
{
    using (DbManager db = new DbManager())
    {
        DataSet dataSet = db
            .SetSpCommand("Person_SelectListByName", firstName, lastName)
            .ExecuteDataSet();

        int returnValue = (int)db.Parameter("@RETURN_VALUE").Value;

        if (returnValue != 0)
        {
            throw new Exception(
                string.Format("Return value is '{0}'", returnValue));
        }

        return dataSet;
    }
}
```

Последней возможностью работы с параметрами, которую нам осталось рассмотреть, является использование функции подготовки запроса Prepare, которая может быть полезной при выполнении одного и того же запроса несколько раз. Фактически в данном случае вызов метода Execute… разбивается на две части: первая - вызов Prepare с заданием типа, текста и параметров запроса, вторая - вызов соответствующего метода Execute… для выполнения запроса определённое число раз. Следующий пример демонстрирует данную возможность.

```csharp
void InsertRegionList(Region[] regionList)
{
    using (DbManager db = new DbManager())
    {
        db
            .SetCommand (@"
                INSERT INTO Region (
                    RegionID,
                    RegionDescription
                ) VALUES (
                    @ID,
                    @Description
                )",
                db.Parameter("@ID",          regionList[0].ID),
                db.Parameter("@Description", regionList[0].Description))
            .Prepare();

        foreach (Region r in regionList)
        {
            db.Parameter("@ID").Value          = r.ID;
            db.Parameter("@Description").Value = r.Description;

            db.ExecuteNonQuery();
        }
    }
}
```

Либо мы можем упростить его следующим образом для бизнес объектов...

```csharp
void InsertRegionList(Region[] regionList)
{
    using (DbManager db = new DbManager())
    {
        db
            .SetCommand(@"
                INSERT INTO Region (
                    RegionID,
                    RegionDescription
                ) VALUES (
                    @ID,
                    @Description
                )",
                db.CreateParameters(regionList[0]))
            .Prepare();

        foreach (Region r in regionList)
        {
            db.AssignParameterValues(r);
            db.ExecuteNonQuery();
        }
    }
}
```

и класса DataRow

```csharp
static void InsertRegionTable(DataTable dataTable)
{
    using (DbManager db = new DbManager())
    {
        db
            .SetCommand(@"
                INSERT INTO Region (
                    RegionID,
                    RegionDescription
                ) VALUES (
                    @ID,
                    @Description
                )",
                db.CreateParameters(dataTable.Rows[0]))
            .Prepare();

        foreach (DataRow dr in dataTable.Rows)
            db.AssignParameterValues(dr).ExecuteNonQuery();
    }
}
```

Конечно, для совсем ленивых есть вот такой вариант (метод ExecuteForEach использует именно описанный выше механизм):

```csharp
void InsertRegionList(Region[] regionList)
{
    using (DbManager db = new DbManager())
    {
        db
            .SetCommand(@"
                INSERT INTO Region (
                    RegionID,
                    RegionDescription
                ) VALUES (
                    @ID,
                    @Description
                )")
            .ExecuteForEach(regionList);
    }
}
```



<span id="executemethods"></span>
### Методы Execute

Класс *DbManager* содержит целый набор семейств методов Execute. Каждое семейство отличается типом возвращаемой сущности, это может быть как *DataSet*, бизнес объект, коллекция бизнес объектов и так далее. Ниже мы рассмотрим все семейства Execute.


<span id="executedataset"></span>
#### ExecuteDataSet

```csharp
public DataSet ExecuteDataSet();

public DataSet ExecuteDataSet(DataSet dataSet);

public DataSet ExecuteDataSet(NameOrIndexParameter table);

public DataSet ExecuteDataSet(
    DataSet              dataSet,
    NameOrIndexParameter table);

public DataSet ExecuteDataSet(
    DataSet              dataSet,
    int                  startRecord,
    int                  maxRecords,
    NameOrIndexParameter table);
```

Как видно из названия метода результатом данного выражения является объект класса *DataSet* (подобное семантическое правило сохраняется для всех семейств).

Рассмотрим подробнее параметры методов:

- *dataSet* – результирующий датасет (он будет заполнен и возвращен). Если *null*, то будет создан новый экземпляр датасета. Подобный подход импользуется так же в других методах семейств Execute.
- *table* – имя или номер таблицы для заполнения в результирующем датасете. Отдельный интерес представляет класс *NameOrIndexParameter*, для ознакомления с технологией работы лучше прочитать статью: [Унифицированная система передачи строковых/числовых параметров](http://rsdn.ru/forum/prj.rfd/1940433.1). 
- *startRecord* – номер записи с которой начинать заполнение (считается с нуля).
- *maxRecords* – максимальное число записей для заполнения.

Отдельно отмечу, что библиотека писалась как «самодокументируемая», поэтому в большинстве случаев используются схожие приемы и сохраняются имена для параметров с одинаковым смыслом. Поэтому ниже мы не будем рассматривать повторно то, что уже было рассмотрено ранее.


<span id="executedatatable"></span>
#### ExecuteDataTable и ExecuteDataTables

```csharp
public DataTable ExecuteDataTable();

public DataTable ExecuteDataTable(DataTable dataTable);

public void ExecuteDataTables(
    int    startRecord,
    int    maxRecords,
    params DataTable[] tableList);

public void ExecuteDataTables(params DataTable[] tableList);
```

Как видим, в данном семействе есть два вида методов: *ExecuteDataTable* – заполняет одну таблицу, *ExecuteDataTables* – заполняет массив таблиц, заданный параметром *tableList*.


<span id="executereader"></span>
#### ExecuteReader

```csharp
public IDataReader ExecuteReader();

public IDataReader ExecuteReader(CommandBehavior commandBehavior)
```

Возвращает экземпляр *IDataReader*.

C *commandBehavior* подробней можно ознакомиться в [MSDN](http://msdn.microsoft.com/en-us/library/system.data.commandbehavior(VS.85).aspx).


<span id="executenonquery"></span>
#### ExecuteNonQuery

```csharp
public int ExecuteNonQuery();

public int ExecuteNonQuery(
    string returnValueMember,
    object obj);

public int ExecuteNonQuery(object obj);

public int ExecuteNonQuery(
    string          returnValueMember,
    params object[] objects);

public int ExecuteNonQuery(params object[] objects);
```

Данное семейство используется для выполнения *UPDATE*, *INSERT* и *DELETE* запросов. Все методы возвращают число записей, обработанных запросом.

Рассмотрим подробнее параметры методов:

- *returnValueMember* – грубо говоря, это имя поля в объекте, в которое необходимо записать возвращаемое значение. Если же быть точным, то это имя маппера поля (MemberMapper) в которое следует записать возвращаемое значение. Подробнее о мапинге (отображении) мы поговорим ниже.
- *obj* – объект в который будут отображены (записаны) параметры команды.
- *objects* – коллекция объектов в которые будут отображены (записаны) параметры команды.

Здесь мы впервые столкнулись с проявлениями мапинга (отображения). Ранее мы говорили, что BLT как раз занимается в первую очередь отображением данных из БД на объекте в коде, пришло время рассмотреть первый пример этого отображения.

Первые участники действа это хранимые процедуры:

```sql
-- OutRefTest
CREATE Procedure OutRefTest
	@ID             int,
	@outputID       int output,
	@inputOutputID  int output,
	@str            varchar(50),
	@outputStr      varchar(50) output,
	@inputOutputStr varchar(50) output
AS

SET @outputID       = @ID
SET @inputOutputID  = @ID + @inputOutputID
SET @outputStr      = @str
SET @inputOutputStr = @str + @inputOutputStr

-- Scalar_ReturnParameter
CREATE Function Scalar_ReturnParameter()
RETURNS int
AS
BEGIN
	RETURN 12345
END
```

И собственно тесты:

```csharp
public class ReturnParameter
{
	public int Value;
}

[Test]
public void MapReturnValue()
{
	ReturnParameter e = new ReturnParameter();
	using (DbManager db = new DbManager())
	{
		db
			.SetSpCommand("Scalar_ReturnParameter")
			.ExecuteNonQuery("Value", e);
	}

	Assert.AreEqual(12345, e.Value);
}
```

Как видим в данном тесте BLT успешно отображает возвращаемое функцией *Scalar_ReturnParameter* значение на поле *Value* объекта класса *ReturnParameter*.

Рассмотрим еще два теста:

```csharp
public class OutRefTest
{
	public int    ID             = 5;
	public int    outputID;
	public int    inputOutputID  = 10;
	public string str            = "5";
	public string outputStr;
	public string inputOutputStr = "10";
}

[Test]
public void MapOutput()
{
	OutRefTest o = new OutRefTest();

	using (DbManager db = new DbManager())
	{
		db
			.SetSpCommand("OutRefTest", db.CreateParameters(o,
				new string[] {      "outputID",      Str" },
				new string[] { "inputOutputID", utputStr" },
				null))
			.ExecuteNonQuery(o);
	}

	Assert.AreEqual(5,     o.outputID);
	Assert.AreEqual(15,    o.inputOutputID);
	Assert.AreEqual("5",   o.outputStr);
	Assert.AreEqual("510", o.inputOutputStr);
}

[Test]
public void MapDataRow()
{
	DataTable dataTable = new DataTable();
	dataTable.Columns.Add("ID",             typeof(int));
	dataTable.Columns.Add("outputID",       typeof(int));
	dataTable.Columns.Add("inputOutputID",  typeof(int));
	dataTable.Columns.Add("str",            typeof(string));
	dataTable.Columns.Add("outputStr",      typeof(string));
	dataTable.Columns.Add("inputOutputStr", typeof(string));

	DataRow dataRow = dataTable.Rows.Add(new object[]{5, 0, 10, "5", 10"});

	using (DbManager db = new DbManager())
	{
		db
			.SetSpCommand("OutRefTest", teParameters(dataRow,
				new string[] {      "outputID",      Str" },
				new string[] { "inputOutputID", utputStr" },
				null))
			.ExecuteNonQuery(dataRow);
	}

	Assert.AreEqual(5,     dataRow["outputID"]);
	Assert.AreEqual(15,    dataRow["inputOutputID"]);
	Assert.AreEqual("5",   dataRow["outputStr"]);
	Assert.AreEqual("510", dataRow["inputOutputStr"]);
}
```

Здесь я специально рассмотрел два примера, хотя, по сути, демонстрируют они **одинаковое** использование *ExecuteNonQuery*. Разница заключается в том, что в первом тесте отображение происходит на бизнес объект класса *OutRefTest* а во втором на объект класса *DataRow*. BLT с успехом справляется с задачей отображения «ужа на ежа» а при необходимости и «ужа на слона».

Отличием этих двух тестов от предыдущего, является то, что мы не сообщили **явно** куда и что отображать. Это не есть проявление телепатии, это есть проявление здравого смысла – при мапинге система ориентируется в частности на имена полей и параметров. В рассмотренном примере имена полей класса *OutRefTest* и ячеек объекта *dataRow* совпадают с именами параметров хранимой процедуры *OutRefTest*, именно по этим признакам система поняла, что и куда раскладывать.


```csharp
public IList ExecuteScalarList(
    IList                list,
    Type                 type,
    NameOrIndexParameter nameOrIndex);

public IList ExecuteScalarList(
    IList list, 
    Type  type);

public ArrayList ExecuteScalarList(
    Type                 type, 
    NameOrIndexParameter nameOrIndex);

public ArrayList ExecuteScalarList(Type type);

public List<T> ExecuteScalarList<T>();

public List<T> ExecuteScalarList<T>(NameOrIndexParameter nameOrIndex);

public IList<T> ExecuteScalarList<T>(
    IList<T>             list,
    NameOrIndexParameter nameOrIndex);

public IList<T> ExecuteScalarList<T>(IList<T> list);
```

Семейство предназначено для получения скалярных величин. Функции без параметров возвращают значение в первой колонке первой строки полученного запросом кортежа. 

Подробнее рассмотрим параметры:

- *sourceType* – одно из значений перечисления *ScalarSourceType*, может принимать следующие значения: *DataReader* – будет возвращено значение в первой колонке первой строки кортежа; *OutputParameter* – будет возвращен первый выходной параметр; *ReturnValue* – позволяет получить возвращаемое значение; *AffectedRows* – количество строк, обработанных запросом.
- *nameOrIndex* – позволяет задать имя \ номер колонки (для *ScalarSource.DataReader*) либо параметра (для *ScslsrSource.OutputParameter*) которые следует возвращать.

Generic версии методов позволяют явно задать тип возвращаемого значения.

<blockquote style="background-color: #F5F9FF; color: #506580;">
Мы не будем рассматривать примеры для семейств ExecuteScalar*, вместо этого мы рассмотрим примеры к ExecuteObject и родственным ему семействам. По структуре они практически идентичны, но ExecuteObject гораздо интереснее :).
</blockquote>


<span id="executescalarlist"></span>
#### ExecuteScalarList

```csharp
public IList ExecuteScalarList(
    IList                list,
    Type                 type,
    NameOrIndexParameter nameOrIndex);

public IList ExecuteScalarList(
    IList list, 
    Type  type);

public ArrayList ExecuteScalarList(
    Type                 type, 
    NameOrIndexParameter nameOrIndex);

public ArrayList ExecuteScalarList(Type type);

public List<T> ExecuteScalarList<T>();

public List<T> ExecuteScalarList<T>(NameOrIndexParameter nameOrIndex);

public IList<T> ExecuteScalarList<T>(
    IList<T>             list,
    NameOrIndexParameter nameOrIndex);

public IList<T> ExecuteScalarList<T>(IList<T> list);
```

Семейство предназначено для вычитки списка скалярных величин. Практически все параметры идентичны по семантике параметрам семейства ExecuteScalar. Параметр *type* задает требуемый тип вычитываемой скалярной величины.


<span id="executescalardictionary"></span>
#### ExecuteScalarDictionary

```csharp
public IDictionary ExecuteScalarDictionary(
	IDictionary dic,
	NameOrIndexParameter keyField,   Type keyFieldType,
	NameOrIndexParameter valueField, Type valueFieldType);

public Hashtable ExecuteScalarDictionary(
	NameOrIndexParameter keyField,   Type keyFieldType,
	NameOrIndexParameter valueField, Type valueFieldType);

public IDictionary<K,T> ExecuteScalarDictionary<K,T>(
	IDictionary<K,T>     dic,
	NameOrIndexParameter keyField,
	NameOrIndexParameter valueField);

public Dictionary<K,T> ExecuteScalarDictionary<K,T>(
	NameOrIndexParameter keyField,
	NameOrIndexParameter valueField);
```

Одной из приятностей BLToolkit является возможность возвращать не просто списки, а словари. Итак, данное семейство возвращает словари скалярных величин из кортежа. Параметры по семантике аналогичны семейству ExecuteScalar, прификс *key* – для ключа, прификс *value* для значения.

Но, на этом еще не все. У данного семейства есть еще подсемейство:

```csharp
public IDictionary ExecuteScalarDictionary(
	IDictionary          dic,
	MapIndex             index,
	NameOrIndexParameter valueField,
	Type                 valueFieldType);

public Hashtable ExecuteScalarDictionary(
	MapIndex             index, 
	NameOrIndexParameter valueField, 
	Type                 valueFieldType);

public IDictionary<CompoundValue,T> ExecuteScalarDictionary<T>(
	IDictionary<CompoundValue, T> dic, 
	MapIndex                      index, 
	NameOrIndexParameter          valueField);

public Dictionary<CompoundValue,T> ExecuteScalarDictionary<T>(
	MapIndex             index, 
	NameOrIndexParameter valueField)
```

Отличается оно тем, что вместо параметров с префиксом *key* используется параметр *index*. Параметр *index* позволяет строить индекс не по одному ключевому полю, а по их совокупности. Таким образом, ключом в результирующем словаре будет экземпляр класса *CompaundValue*, представляющий сложный ключ как единый объект. Мы обязательно рассмотрим пример использования «индексированных» словарей, но ниже.


<span id="executeobject"></span>
#### ExecuteObject

```csharp
```

Пожалуй, одно из самых интересных семейств. Предназначено для чтения одной записи возвращаемого кортежа в бизнес объект.

Рассмотрим параметры функций:

- *entity* – объект, куда будет осуществлено чтение.
- *type* – задает требуемый тип возвращаемого объекта.
- *parameters* – дополнительные параметры, которые будут переданы в конструктор. Здесь стоит отметить, что переданы они будут как соответствующее свойство объекта *InitContext*, класс бизнес объекта, в свою очередь, должен иметь конструктор вида *MyObject(InitContext context)*.

Рассмотрим пример:

```csharp
[Test]
public void ExecuteObject()
{
	using (DbManager db = new DbManager())
	{
		Person p = (Person)db
			.SetCommand("SELECT * FROM Person WHERE PersonID = @id",
			db.Parameter("id", 1))
			.ExecuteObject(typeof(Person));

		TypeAccessor.WriteConsole(p);
		Assert.AreEqual(1,           p.ID);
		Assert.AreEqual("John",      p.FirstName);
		Assert.AreEqual("Pupkin",    p.LastName);
		Assert.AreEqual(Gender.Male, p.Gender);
	}
}
```

Кортеж будет иметь следующий вид:

| PersonId | FirstName | LastName | MiddleName | Gender |
|----------|-----------|----------|------------|--------|
| 1        | John      | Pupkin   | NULL       | M      |

Итак, система отобразила выбранную запись на бизнес объект типа Person. Подробней стоит остановиться на полях Person.ID и Person.Gender. Отметим пару интересных моментов:

- В исходном кортеже нет поля ID, а в классе Person поля PersonId. Эта проблема была решена атрибутом MapField(“PersonId”), установленным на поле Person.ID. Так мы сообщили системе, что при мапинге у данного поля будет псевдоним отличный от «родового имени».
- В исходном кортеже поле Gender имеет символьный тип, Person.Gender – является перечислением. Здесь нас выручил атрибут MapValue(“M”) – им мы указали системе, что при отображении данное значение является эквивалентным “M”.


```csharp
public ArrayList ExecuteList(Type type);

public ArrayList ExecuteList(Type type, params object[] parameters);

public IList ExecuteList(IList list, Type type);

public IList ExecuteList(
    IList           list, 
    Type            type, 
    params object[] parameters);

public List<T> ExecuteList<T>();

public List<T> ExecuteList<T>(params object[] parameters);

public IList<T> ExecuteList<T>(IList<T> list);

public IList<T> ExecuteList<T>(IList<T> list, params object[] parameters);

public L ExecuteList<L,T>(L list, params object[] parameters)
    where L : IList<T>;

public L ExecuteList<L,T>(params object[] parameters)
    where L : IList<T>, new();
```

Данное семейство предназначено для чтения списка объектов из выбранного кортежа. Параметры аналогичны семейству ExecuteObject, поэтому на них мы останавливаться не будем.

<blockquote style="background-color: #FFE4E4; color: #FF5555;">
Не используйте данное семейство для вычитки списка скалярных величин, для этого существует семейство <b>ExecuteScalarList</b>.
</blockquote>

Рассмотрим небольшой пример использования данного семейства:

```csharp
[Test]
public void ExecuteList1()
{
	using (DbManager db = new DbManager())
	{
		ArrayList list = db
			.SetCommand("SELECT * FROM Person")
			.ExecuteList(typeof(Person));
		
		Assert.IsNotEmpty(list);
	}
}
```


<span id="executedictionary"></span>
#### ExecuteDictionary

```csharp
public Hashtable ExecuteDictionary(
	NameOrIndexParameter keyField,
	Type                 keyFieldType,
	params object[]      parameters);

public IDictionary ExecuteDictionary(
	IDictionary          dictionary,
	NameOrIndexParameter keyField,
	Type                 type,
	params object[]      parameters);

public Dictionary<TKey, TValue> ExecuteDictionary<TKey, TValue>(
	NameOrIndexParameter keyField,
	params object[]      parameters);

public IDictionary<TKey, TValue> ExecuteDictionary<TKey, TValue>(
	IDictionary<TKey, TValue> dictionary,
	NameOrIndexParameter      keyField,
	params object[]           parameters);

public IDictionary<TKey, TValue> ExecuteDictionary<TKey, TValue>(
	IDictionary<TKey, TValue> dictionary,
	NameOrIndexParameter      keyField,
	Type                      destObjectType,
	params object[]           parameters)
```

Позволяет вычитывать словарь бизнес объектов из кортежа. Семантика параметров аналогична ExecuteScalarDictionary и ExecuteObject. Параметру type и destObjectType – задают требуемый тип бизнес объекта.

<blockquote style="background-color: #FFE4E4; color: #FF5555;">
Не используйте данное семейство для вычитки словарей скалярных величин, для этого есть семейство <b>ExecuteScalarDictionary</b>.
</blockquote>

Как и в случае с ExecuteScalarDictionary ExecuteDictionary имеет «индексированное» подсемейство:

```csharp
public Hashtable ExecuteDictionary(
	MapIndex        index,
	Type            type,
	params object[] parameters);

public IDictionary ExecuteDictionary(
	IDictionary     dictionary,
	MapIndex        index,
	Type            type,
	params object[] parameters);

public Dictionary<CompoundValue, TValue> ExecuteDictionary<TValue>(
	MapIndex        index,
	params object[] parameters);

public IDictionary<CompoundValue, TValue> ExecuteDictionary<TValue>(
	IDictionary<CompoundValue, TValue> dictionary,
	MapIndex                           index,
	params object[]                    parameters);

public IDictionary<CompoundValue, TValue> ExecuteDictionary<TValue>(
	IDictionary<CompoundValue, TValue> dictionary,
	MapIndex                           index,
	Type                               destObjectType,
	params object[]                    parameters)
```

Опять-таки, нам тут все знакомо, поэтому для закрепления понимания сразу перейдем к примеру:

```csharp
private const int     _id = 1;

[Test]
public void DictionaryMapIndexTest3()
{
	using (DbManager db = new DbManager())
	{
		Hashtable table = new Hashtable();
		db
			.SetCommand("SELECT * FROM Person")
			.ExecuteDictionary(table,
			new MapIndex("@PersonID", 2, 3), typeof(Person));

		Assert.IsNotNull(table);
		Assert.IsTrue(table.Count > 0);

		Person actualValue = (Person)table[new CompoundValue(_id, "", "Pupkin")];
		Assert.IsNotNull(actualValue);
		Assert.AreEqual("John", actualValue.FirstName);
	}
}
```

В примере используется сложный ключ, состоящий из полей PersonId, третьего поля в кортеже (все считается с нуля) – SecondName и четвертого поля в кортеже – MiddleName. Ключом в словаре является объект класса CompaundValue.

Ну и как всегда, если нам не нужны такие изыски (сложные ключи) то можно сделать все гораздо проще:

```csharp
[Test]
public void GenericsDictionaryTest()
{
	using (DbManager db = new DbManager())
	{
		Dictionary<int, Person> dic = db
			.SetCommand("SELECT * FROM Person")
			.ExecuteDictionary<int, Person>("ID");

		Assert.IsNotNull(dic);
		Assert.IsTrue(dic.Count > 0);

		Person actualValue = dic[1];
		Assert.IsNotNull(actualValue);
		Assert.AreEqual("John", actualValue.FirstName);
	}
}
```

<blockquote style="background-color: #FFE4E4; color: #FF5555;">
Как видно из примеров, в одном случае используется «@PersonId» а в другом «ID». Разница в следующем: если не указано '@', то значение берётся из поля уже смапленного объекта, если '@' присутствует, то из исходной записи.
<br/><br/>
Зачем это надо. Первый случай может пригодиться, если словарь строится по полю, которое явно не отображается на исходную запись. Например, какое-нибудь составное поле в объекте. Второй случай может понадобиться, когда нужно построить словарь по полю, которое есть в исходном рекордсете, но не отображается на объект. Если ключевое поле один в один отображается на объект, то разницы нет.
<br/><br/>
Оригинал by <a href="http://rsdn.ru/Users/1.aspx">Игорь Ткачев</a> – <a href="http://rsdn.ru/forum/message/3017055.1.aspx">здесь</a>.
</blockquote>


```csharp
public int ExecuteForEach(ICollection collection);

public int ExecuteForEach<T>(ICollection<T> collection);

public int ExecuteForEach(DataTable table);

public int ExecuteForEach(DataSet dataSet);

public int ExecuteForEach(DataSet dataSet, NameOrIndexParameter nameOrIndex);
```

Ранее я уже приводил пример данного семейства. Но не грех и повторить: данное семейство выполняет SQL выражение для заданного множества. Сначала команда готовит выражение, используя метод *Prepare()* после чего выполняет *ExecuteNonQuery()* для каждого элемента коллекции (из элементов коллекции заполняются значения параметров).

Параметры, подробно описывать не буду, замечу только, что для *dataSet* без *nameOrIndex* выражение будет выполнено для первой таблицы (индекс == 0).


```csharp
public MapResultSet[] ExecuteResultSet(params MapResultSet[] resultSets);

public MapResultSet[] ExecuteResultSet(
	Type masterType, 
	params MapNextResult[] nextResults);

public MapResultSet[] ExecuteResultSet<T>(params MapNextResult[] nextResults);
```

Это семейство позволяет выполнять комплексное отображение данных на сложную связанную иерархию объектов.

Можно долго рассказывать, как и что, но проще разобрать все на примере. В примере заданы следующие связи (в рамках объектной модели):

- Parent к Child – ко многим 
- Child к Parent – к одному
- Child к Grandchild – ко многим
- Grandchild к Child – к одному.

Таким образом, имеем иерархию из 3 взаимосвязанных классов.

Особое внимание следует обратить на то, что повязка осуществляется по именам для отображения.

Читайте, наслаждайтесь (примечания в комментариях к тексту):

```csharp
[TestFixture]
public class ComplexMapping
{
	// запрос с 3 связанными таблицами
	const string TestQuery = @"
		-- Parent Data
		SELECT       1 as ParentID
		UNION SELECT 2 as ParentID

		-- Child Data
		SELECT       4 ChildID, 1 as ParentID
		UNION SELECT 5 ChildID, 2 as ParentID
		UNION SELECT 6 ChildID, 2 as ParentID
		UNION SELECT 7 ChildID, 1 as ParentID

		-- Grandchild Data
		SELECT       1 GrandchildID, 4 as ChildID
		UNION SELECT 2 GrandchildID, 4 as ChildID
		UNION SELECT 3 GrandchildID, 5 as ChildID
		UNION SELECT 4 GrandchildID, 5 as ChildID
		UNION SELECT 5 GrandchildID, 6 as ChildID
		UNION SELECT 6 GrandchildID, 6 as ChildID
		UNION SELECT 7 GrandchildID, 7 as ChildID
		UNION SELECT 8 GrandchildID, 7 as ChildID
";
	// верхний класс
	public class Parent
	{
		[MapField("ParentID")]
		public int ID;
		// Список подчиненных объектов
		public List<Child> Children = new List<Child>();
	}
	// класс связанный с Parent
	[MapField("ParentID", "Parent.ID")]
	public class Child
	{
		[MapField("ChildID")]
		public int ID;
		// родительский объект
		public Parent Parent = new Parent();
		//Список подчиненных объектов
		public List<Grandchild> Grandchildren = new List<Grandchild>();
	}
	// Класс связи связанный с Child 
	[MapField("ChildID", "Child.ID")]
	public class Grandchild
	{
		[MapField("GrandchildID")]
		public int ID;
		// родительский объект
		public Child Child = new Child();
	}

	[Test]
	public void Test()
	{
		// список родительских объектов – «корень» который будет заполнен
		List<Parent>   parents = new List<Parent>();
		// массив резалтсетов
		/*[/a]*/MapResultSet/*[/a]*/[] sets    = new MapResultSet[3];

		//создадим резалтсет для корневого списка
		// в качестве параметров переданы тип корневого объекта и 
		// и список объектов, который следует заполнить
		sets[0] = new MapResultSet(typeof(Parent), parents);
		sets[1] = new MapResultSet(typeof(Child));
		sets[2] = new MapResultSet(typeof(Grandchild));

		// зададим связь резалтсету «Parent» устанавливается подчиненная
		// связь к резалтсету «Child»
		// параметры:
		// имя поля отображения по которому осуществляется связь в подчиненном объекте
		// имя поля отображения по которому осуществляется связь в родительском объекте
		// имя поля отображения в родительском объекте для заполнения дочерними
		sets[0].AddRelation(sets[1], "ParentID", "ParentID", "Children");
		// все практически аналогично, но теперь задается обратная связь
		// от Child к Parent
		// таким образом в результате отображения будет заполнено не только
		// поле Parent.Children но и для каждого Child из Children будет задан Parent
		sets[1].AddRelation(sets[0], "ParentID", "ParentID", "Parent");

		// Аналогично, но уже от Child к Grandchild и наоборот
		sets[1].AddRelation(sets[2], "ChildID", "ChildID", "Grandchildren");
		sets[2].AddRelation(sets[1], "ChildID", "ChildID", "Child");

		using (DbManager db = new DbManager())
		{
			db
				.SetCommand      (TestQuery)
				.ExecuteResultSet(sets);
		}
		// здесь проверки правильности заполнения
		Assert.IsNotEmpty(parents);

		foreach (Parent parent in parents)
		{
			Assert.IsNotNull(parent);
			Assert.IsNotEmpty(parent.Children);

			foreach (Child child in parent.Children)
			{
				Assert.AreEqual(parent, child.Parent);
				Assert.IsNotEmpty(child.Grandchildren);

				foreach (Grandchild grandchild in child.Grandchildren)
				{
					Assert.AreEqual(child,  grandchild.Child);
					Assert.AreEqual(parent, grandchild.Child.Parent);
				}
			}
		}
	}
}
```

В приведенном примере для осуществления повязки используются строковые имена, аналогично можно использовать составные индексы, при помощи уже известного нам класса MapIndex (если забыли то см. ExecuteScalarDictionary и ExecuteDictionary и их индексированные подсемейства).



<span id="mapping"></span>
## Отображение данных


<span id="mappinggeneral"></span>
### Общие сведенья

ADO.NET поддерживает два способа чтения данных из источника: прямое чтение из объекта класса *DataReader*, либо с помощью класса *DataAdapter* в экземпляр класса *DataSet*, который по сути представляет собой единственный вариант бизнес сущностей, предлагаемых и культивируемых Microsoft. 

Оставим сегодня в покое достоинства и преимущества класса *DataSet*, и лишь заметим, что часто бывает необходимо уметь читать данные непосредственно в бизнес объекты приложения. При этом иногда нужно выполнять некоторые действия по отображению данных, например, из строковых значений в перечислители (enumerators) или замене значений *NULL* на нечто более удобоваримое. Как вы заметили из нашего самого первого примера, класс *DbManager* великодушно предоставляет нам такие возможности.

Вернемся к примеру использования семейства *ExecuteList*, и разберем подробнее что там происходит.

Метод *ExecuteList* создаёт экземпляр класса *Person* для каждой записи в таблице, затем осуществляет отображение данных на поля объекта и добавляет его в список. Для отображения колонок таблицы на поля и свойства нашего объекта используется механизм *Reflection*, единственным недостатком которого является некоторая нерасторопность. Для решения этой проблемы применён ещё один механизм .NET – генерация исполняемого кода во время выполнения программы (*System.Reflection.Emit* namespace), что позволяет максимально увеличить производительность и свести использование *Reflection* только для начальной инициализации.

В отображении участвуют поля и свойства класса, удовлетворяющие следующим требованиям:

- Модификатор доступа – *public*, либо *internal* (работает в случае с динамически генерируемыми классами).
- Тип является скалярным либо, одним из перечисленных: *Guid*, *SqlBinary*, *SqlBoolean*, *SqlByte*, *SqlDateTime*, *SqlDecimal*, *SqlDouble*, *SqlGuid*, *SqlInt16*, *SqlInt32*, *SqlInt64*, *SqlMoney*, *SqlSingle*, *SqlSting*, *XmlReader*, *XmlDocument*.
- Для поля \ свойства задан *MemberMapper* (подробнее ниже).
- Для поля \ свойства задан атрибут *MapIgnore(false)* (подробнее об атрибутах ниже). В данном случае допускаются к использованию поля типа: *byte[]*, *Stream*, *SqlBytes*, *SqlChars*, *SqlXml*.



<span id="mapandschema"></span>
### Map & MappingSchema

Как и следовало ожидать, *DbManager* вовсе не сам выполняет операции по отображению. Эти действия он делегирует объекту класса *MappingSchema*.  В заголовке упомянут так же класс *Map* – это статический класс, предназначенный для упрощения доступа к функциям *MappingSchema*, ввиду чего мы не будем подробно его рассматривать, и сосредоточимся на *MappingSchema*.

Итак, *MappingSchema* содержит в себе весь необходимый для выполнения отображения контекст, а так же набор семейств функций по отображению ужей на ежей.  Более того, *MappingSchema* содержит так же правила преобразования (конвертации) данных из одного формата в другой (ну, например из *Int32* в *Boolean*, из *String* в *Boolean* и наоборот). Все это превращает *MappingSchema* в мощный инструмент по преобразованию и отображению данных.

<blockquote style="background-color: #F5F9FF; color: #506580;">
Лирическое отступление на тему OdpDataProvider:
<br/><br/>
Орлы из оракла не пользуются SqlString, SqlInt32 и т.п. типами, а напридумывали велосипедов. Поэтому приходится прилагать так много усилий, чтобы привести всякие OracleDecimal хотя бы к System.Decimal.
<br/><br/>
Оригинал by <a href="http://www.rsdn.ru/Users/507.aspx">Павел Блудов</a> – <a href="http://www.rsdn.ru/forum/message/2479957.1.aspx">здесь</a>.
</blockquote>



<span id="mappingschema"></span>
### Семейства функций MappingSchema

Данные семейства можно разделить на два больших класса:

- Convert – преобразование данных.
- Map – отображение данных.


<span id="mappingconvert"></span>
#### Семейства Convert

Тут все достаточно просто и понятно по семантике методов: 

```csharp
// шаблон имени выгляди следующим образом:
// ConvertToDestinatonType(object value) ;
// где DestinatonType – тип в который необходимо преобразовать.
// к Int32
ConvertToInt32(object value);
// к Int32?
ConvertToNullableInt32(object value);
// к SqlInt32
ConvertToSqlInt32(object value);
```

По умолчанию *MappingSchema* делегирует подобные вызовы к классу *Convert* (*BLToolkit.Common.Convert*). Данный класс можно расценивать как замену стандартному классу *System.Convert*, который можно смело назвать "младшим братом", т.к. *BLToolkit.Common.Convert* значительно превосходит его по возможностям.

Отдельно стоит отметить следующие "высокоуровневые" функции:

```csharp
public virtual object ConvertChangeType(
	object value, 
	Type   conversionType);

public virtual object ConvertChangeType(
	object value, 
	Type   conversionType, 
	bool   isNullable);

public virtual T ConvertTo<T, P>(P value);
```

Разберем подробнее параметры

- *value* – значение, которое необходимо преобразовать.
- *conversionType* – результирующий тип к которому необходимо преобразовать.
- *isNullable* – указывает допускает ли результирующий тип значение null.
- *<T, P>* - T – результирующий тип, P – исходный тип.

Ввиду прозрачности функций *Convert** примеров я приводить не буду.


<span id="mappingmap"></span>
#### Семейства Map

Вкратце изложу структуру данного раздела: во-первых, я расскажу про общие принципы именования методов и стандартные виды отображений; во-вторых, я более подробно расскажу о том как это все работает на «низком» уровне.

Семантика методов семейства Map следующая: Map*Source*To*Destination* – все просто: отобразить источник на конечную сущность.

Стандартные участники мапинга (в обе стороны):

- DataReader (IDataReader).
- DataRow.
- DataTable.
- Dictionary (IDictionary, IDictionary<K, T>).
- List (IList, IList<T>).
- Object (бизнес объект).
- ScalarList(IList, IList<T>).
- ResultSet.
- EnumToValue & ValueToEnum.

Как вы уже догадались, в большинстве методов Execute класса *DbManager* прячется обращение к семейству MapDataReaderTo*Destination*, мы достаточно подробно разобрали методы Execute и их параметры, так что, разобраться с параметрами семейств Map для вас не должно составить особого труда.

Самым «низким» уровнем отображения являются следующие методы:

```csharp
public void MapSourceToDestination(
	IMapDataSource      source, object sourceObject, 
	IMapDataDestination dest,   object destObject,
	params object[]     parameters);

public void MapSourceToDestination(
	object          sourceObject,
	object          destObject,
	params object[] parameters);

public virtual void MapSourceListToDestinationList(
	IMapDataSourceList      dataSourceList,
	IMapDataDestinationList dataDestinationList,
	params object[]         parameters)
```

Последний метод отличается от двух первых – он отображает списки объектов.

И как всегда, подробнее о параметрах:

- source – источник, наследник IMapDataSource – предоставляет методы для извлечения данных из источника.
- dest – получатель, наследник IMapDataDestination – предоставляет методы для записи данных.
- sourceObject – собственно объект, из которого происходит отображение.
- destObject – объект в который происходит отображение.
- parameters – набор параметров передаваемый в конструктор объекта получателя через экземпляр InitContext.
- dataSourceList и dataDestinationList – то же, что и source и dest, только для списков.

Разберем пример отображения DataReader на Person:

```csharp
public Person MapDataReaderToPerson(IDataReader reader, Person p)
{
	MappingSchema schema     = new MappingSchema();

	IMapDataSource source    = schema.CreateDataReaderMapper(reader);
	IMapDataDestination dest = schema.GetObjectMapper       (p.GetType());

	Schema.MapDataReaderToObject(source, reader, dest, p);
	
	return p;
}
```

Вот, примерно так оно и происходит.

Теперь давайте подробней рассмотрим IDataSource и IDataDestination. Данные интерфейсы описывают методы, которые предоставляют возможности чтенья из источника и записи в получателя. 

Вкратце рассмотрим данные интерфейсы:

```csharp
public interface IMapDataSource
{
	// общее количество доступных для чтения полей
	int      Count { get; }

	// тип поля по индексу
	Type     GetFieldType (int index);
	// имя поля по индексу
	string   GetName      (int index);
	// получает индекс поля по имени
	int      GetOrdinal   (string name);
	// получить значение из объекта по заданному индексу
	object   GetValue     (object o, int index);
	// получить значение из объекта по заданному имени
	object   GetValue     (object o, string name);

	// поле по заданному индексу IsNull
	bool     IsNull       (object o, int index);

	// поддерживает типизированные значения для поля по индексу
	bool     SupportsTypedValues(int index);

	// получить типизированное значение по заданному индексу
	SByte    GetSByte     (object o, int index);
	Int16    GetInt16     (object o, int index);
	// и так далее
	// XXX GetXXX (object o, int index);
}

public interface IMapDataDestination
{
	// тип поля по индексу
	Type GetFieldType (int index);
	// получает индекс поля по имени
	int  GetOrdinal   (string name);
	// устанавливает значение value в объекте о по индексу
	void SetValue     (object o, int index,   object value);
	// устанавливает значение value в объекте о по имени
	void SetValue     (object o, string name, object value);

	// устанавливает значение null в объекте по индексу
	void SetNull      (object o, int index);

	// поддерживает типизированные значения для поля по индексу
	bool SupportsTypedValues(int index);

	// устанавливают типизированное значение value в объекте о по byltrce
	void SetSByte     (object o, int index, SByte    value);
	void SetInt16     (object o, int index, Int16    value);
	// и так далее
	// SetXXX(object o, int index, XXX value);
}
```

Про поддержку типизированных значений стоит написать отдельно, и не своими словами:

<blockquote style="background-color: #F5F9FF; color: #506580;">
В интерфейсах IMapDataSource и IMapDataDestination есть методы типа 
<br/><br/>
Int32    GetInt32     (object o, int index);
<br/><br/>
Означающие "возьмите у объекта o поле за номером index и верните его как int".
<br/><br/>
Смысл в том, что если у нас есть миллион объектов с двумя полями типа int, то при мапинге через GetValue/SetValue половина работы уходит на boxing/unboxing.
<br/><br/>
Т.е. CLR на полном серьёзе выделяет в куче 2 миллиона маленьких объектов, оборачивает в них наши числа и потом как-нибудь эти 2 миллиона объектов высвобождает.
<br/><br/>
Получаем на ровном месте фрагментацию памяти и лишние вызовы сборщика мусора.
<br/><br/>
При мапинге через TypedValues boxing'а не происходит. GetInt32 вычитывает целое число и сохраняет его в регистр EAX.
<br/><br/>
SetInt32 берёт из EAX и выставляет нашему полю это значение. Если поле имеет тип, например, Int64, то код будет более мудрёным:
<br/><br/>
destMapper.SetInt64(destObj,dstIndex,Converter.ConvertInt32ToInt64(srcMapper.GetInt32(srcObj, srcIndex));
<br/><br/>
Опять-таки никакого выделения/освобождения памяти.
<br/><br/>
Так вот, SupportsTypedValues как раз и сообщает маперу, что источник/получатель умеет работать с числами, датами и т.п. без boxing'а.
<br/><br/>
Оригинал by <a href="http://rsdn.ru/Users/507.aspx">Павел Блудов</a> – <a href="http://rsdn.ru/forum/message/2999766.1.aspx">здесь</a>.
</blockquote>

Дополню, что SupportsValueTypes работает в паре с GetFieldType, который должен сообщить правильный тип поля.

Для отображения списков (коллекции, таблицы и т.п.) существуют еще два дополнительных интерфейса:

```csharp
public interface IMapDataSourceList
{
	void InitMapping      (InitContext initContext);
	bool SetNextDataSource(InitContext initContext);
	void EndMapping       (InitContext initContext);
}

public interface IMapDataDestinationList
{
	void                InitMapping       (InitContext initContext);
	IMapDataDestination GetDataDestination(InitContext initContext);
	object              GetNextObject     (InitContext initContext);
	void                EndMapping        (InitContext initContext);
}
```

InitMapping и EndMapping – инициализация и окончание отображения. В остальном, все сводится к тому, что при маппинге списков производится поочередное отображение каждого их элемента. 

Рассмотренные интерфейсы позволяют описать некий источник или получатель данных, и, при необходимости, вы легко можете расширить систему отображения необходимыми вам источниками и получателями.


<span id="mappingmapper"></span>
#### MemberMapper

Теперь вы получили представление о том, как работает отображение объектов. Но это еще не все. Кроме представлений объектов, есть еще представления полей. Для этого используются *ValueMapper* и *MemberMapper*. Первый используется для отображения скалярных полей, второй – всех прочих. Механизм *ValueMapper* инкапсулирован в BLT, поэтому рассматривать мы его не будем. А вот *MemberMapper* мы рассмотрим более подробно.

*MemberMapper* позволяет вам… ну тут проще показать. Приведу простой пример: допустим, у некоторого объекта есть свойство со словарем строк. При сохранении данного объекта необходимо так же сохранить и словарь. 

```csharp
public class SimpleDictionaryMapper : MemberMapper
{
	public override object GetValue(object o)
	{
		Dictionary<string, string> dic = base.GetValue(o) as Dictionary<string, string>;
		
		if (dic == null) return null;

		StringBuilder sb = new StringBuilder();
		foreach (string key in dic.Keys)
			sb.AppendFormat("{0}={1};", key, dic[key]);

		return sb.ToString();

	}

	public override void SetValue(object o, object value)
	{
		string s = MappingSchema.ConvertToString(value);
		
		if (s == string.Empty) base.SetValue(o, null);
		
		Dictionary<string, string> dic = new Dictionary<string, string>();

		foreach (string pair in s.Split(';'))
		{
			if (pair.Length < 3) continue;

			string[] keyValue = pair.Split('=');

			if (keyValue.Length != 2) continue;

			dic.Add(keyValue[0], keyValue[1]);
		}

		base.SetValue(o, dic);
	}
}
```

Приведенный пример отображает словарь на строку в заданном формате (*GetValue*) при обратном отображении (*SetValue*) данная строка разбирается и из нее заново собирается словарь.

Используется *MemberMapper* следующим образом:

```csharp
public class TestObject
{
	[MemberMapper(typeof(SimpleDictionaryMapper)]
	public Dictionary<string, string> Dictionary;
}
```



<span id="metadata"></span>
### Метаданные

К счастью ли, к печали, но в BLT нет телепатического модуля. Вместо него выступают метаданные, позволяющее декларативно рассказать BLT, как правильно выполнять отображение.

Метаданные задаются двумя способами:

1. Атрибутами.
2. XML расширениями.

Первый механизм является статическим, второй позволяет менять правила игры в динамике.


<span id="attributes"></span>
#### Атрибуты

**MapFieldAttribute** – позволяет изменять алиасы полей, участвующих в маппинге. Мы уже использовали данный атрибут, для изменения алиаса поля ID класса Person. Но у данного атрибута есть еще некоторые применения:

```csharp
// Задает алиас полю Field1.
// Данный подход можно использовать для «переименования» унаследованных полей.
[MapField("MapName", "Field1")]
public class Object1
{
	public int Field1;
	[MapField("intfld")]
	public int Field2;
}

[MapValue(true,  "Y")]
[MapValue(false, "N")]
public class Object2
{
	public bool Field1;
	public int  Field2;
}

public class Object3
{
	public Object2 Object2 = new Object2();
	public Object4 Object4;
}

//При необходимости пожно задать алиасы для полей вложенных объектов.
[MapField("fld1", "Object3.Object2.Field1")]
[MapField("fld2", "Object3.Object4.Str1")]
public class Object4
{
	public Object3 Object3 = new Object3();
	public string  Str1;
	// Простой способ для отображения вложенных объектов и их полей – 
	// задать формат алиаса.
	[MapField(Format="InnerObject_{0}"]
	public Object2 InnerObject = new Object2();
}
```

**MapValueAttribute** – позволяет задать для значений их синонимы. Мы уже сталкивались с данным атрибутом, при отображении перечислений, но этим его возможности не заканчиваются, приведу еще один пример:

```csharp
public class Object1
{
	[MapValue(true,  "Y")]
	[MapValue(false, "N")]
	public bool Bool1;

	[MapValue(true,  "Y", "Yes")]
	[MapValue(false, "N", "No")]
	public bool Bool2;
}
```

Использовать атрибут можно так же и на весь класс, задавая таким образом синонимы по умолчанию для полей данного типа:

```csharp
[MapValue(true,  "Y")]
[MapValue(false, "N")]
public class Object2
{
	public bool Bool1;

	[MapValue(true,  "Y", "Yes")]
	[MapValue(false, "N", "No")]
	public bool Bool2;
}
```

**MapIgnoreAttribute** – поле, помеченное данным атрибутом будет проигнорировано при отображении.

**MemberMapperAttribute** – позволяет задать для поля специфический *MemberMapper*, пример использования был выше.

**NullableAttribute** – позволяет указать системе, что значение данного поля может принимать null. 


<span id="xml"></span>
#### XML



<span id="dataaccess"></span>
## DataAccess

Пространство имен BLToolkit.DataAccess содержит набор классов, позволяющих легко разделить слой модели домена со слоем доступа к данным, с одной стороны, и «автоматизировать» труд программиста с другой. 

Вкратце:

- DataAccessor – используется для динамической генерации классов, осуществляющих доступ к данным и отображение данных на объекты и наоборот.
- SqlQuery – используется так же для доступа к данным и отображения, отличие от первого в том, что в данном случае динамически генерируются SQL запросы к БД.

В обоих случаях в конечном итоге используется класс **DbManager**, ввиду чего я не буду приводить подробные описания методов и их параметров.

Общие правила использования:

- Если в конструкторе (методе CreateInstance) аксесссору в качестве параметра передается экземпляр *DbManager*, то все обращения к базе производятся через данный объект. То же справдедливо и для методов. За DbManager.Dispose() в этом случае отвечает вызывающий. Иными словами передавая аксессору *DbManager* вы можете выполнить несколько действий в одном соединении.
- Если ни в конструкторе, ни в методе в качестве параметра не передан экземпляр *DbManager*, то в методе будет создан и уничтожен свой экзкмпляр. Иными словами - каждый метод будет открывать и закрывать соединение с БД. 



<span id="sqlquery"></span>
### SqlQuery

Как было сказано выше класс **SqlQuery** автоматически (на основе метаданных) генерирует SQL запросы, а именно: вставка (Insert) удаление (Delete, DeleteByKey), обновление (Update), выборка (Select, SelectByKey).

Как было сказано, для генерации запросов используются метаданные. «Расширить» метаданные можно как при помощи XML-расширений, так и атрибутами.

Рассмотрим используемые атрибуты:

**TableNameAttribute** – указывает имя таблицы для бизнес объекта (по умолчанию имя таблицы совпадает с именем класса).

**MapFieldAttribute** – позволяет изменять алиасы полей, участвующих в маппинге.

**PrimaryKeyAttribute** – указывает на то что данное поле используется в качестве первичного ключа в таблице. Т.е. данные поля будут использованы в условии WHERE для выборки, удаления либо обновления. Атрибут может быть задан для нескольких полей.

**NonUpdatableAttribute** – поле не будет обновлено в инструкции Update.

**SqlIgnoreAttribute** - позволяет явно указать использовать ли данное поля при генерации SQL запросов. 

Рассмотрим пример использования (в примере я использую типизированную версию **SqlQuery** – **SqlQuery< T >**, просто лень писать приведение типов):

```csharp
public enum Gender
{
	[MapValue("F")] Female,
	[MapValue("M")] Male,
	[MapValue("U")] Unknown,
	[MapValue("O")] Other
}
public class Person
{
	[MapField("PersonID"), NonUpdatable, PrimaryKey]
	public int    ID;
	public string FirstName;
	public string MiddleName;
	public string LastName;
	public Gender Gender;
}

[Test]
public void SqlQueryTest()
{
    SqlQuery<Person> da = new SqlQuery<Person>();
    
    Person p1 = da.SelectByKey(1);
    
    Assert.IsNotNull(p1);
    Assert.AreEqual(1,           p1.ID);
    Assert.AreEqual("John",      p1.FirstName);
    Assert.AreEqual(Gender.Male, p1.Gender);

    p1.ID        = 101;
    p1.FirstName = "John II";
    
    int r = da.Update(p1);

    Assert.AreEqual(1, r);

    Person p2 = da.SelectByKey(1);

    Assert.IsNotNull(p2);
    Assert.AreEqual(1,           p2.ID);
    Assert.AreEqual("John II",   p2.FirstName);
    Assert.AreEqual(Gender.Male, p2.Gender);

    da.Delete(p1); // da.DeleteByKey(1);
    
    p2 = da.SelectByKey(p1);

    Assert.IsNull(p2);

    List<Person> persons = da.SelectAll();

    Assert.IsNotNull(persons);
}
```

В ходе данного теста были сгенерированы и выполнены следующие запросы:

```sql
-- SelectByKey
SELECT
	[PersonId],
	[FirstName],
	[MiddleName],
	[LastName],
	[Gender]
FROM [Person]
WHERE [PersonId] = @PersonId_W 

-- Update
UPDATE [Person] 
SET
	[FirstName] = @FirstName,
	[MiddleName] = @MiddleName,
	[LastName] = @LastName,
	[Gender] = @Gender
WHERE [PersonId] = @PersonId_W 

-- Delete, DeleteByKey
DELETE FROM [Person]
WHERE [PersonId] = @PersonId_W 

-- SelectAll
SELECT
	[PersonId],
	[FirstName],
	[MiddleName],
	[LastName],
	[Gender]
FROM [Person]
```

Как вы видите в коде запросов используются символы экранирования и имена параметров специфичные для MS SQL Server, поэтому следует оговориться, что в генерации запросов участвует *DataProvider*, а если быть точным то его метод *Convert(...)*, именно через него код запроса «наделяется» спецификой конкретного сервера. Если бы запрос генерировался с использованием *OdpDataProvider* то он бы выглядел примерно так:

```sql
SELECT
	PersonId,
	FirstName,
	MiddleName,
	LastName,
	Gender
FROM Person
WHERE PersonId = :PersonId_W 
```

А для *FdpDataProvider* так:

```sql
SELECT
	"PersonId",
	"FirstName",
	"MiddleName",
	"LastName",
	"Gender"
FROM "Person"
WHERE "PersonId" = @PersonId_W 
```

Так же следует отметить, что генерация запроса происходит только при первом обращении, после чего запрос кэшируется и при следующих обращениях возвращается из кэша.



<span id="dataaccessor"></span>
### DataAcessor

В отличие от *SqlQuery*, используемого для генерации SQL запросов, *DataAccessor* используется для эмита кода, что избавляет программиста от нудных и рутинных операций. Для начала рассмотрим небольшой пример: 

```csharp
namespace DataAccessorTest
{
    public enum Gender
    {
                [MapValue("F")] Female,
                [MapValue("M")] Male,
                [MapValue("U")] Unknown,
                [MapValue("O")] Other
    }
        public class Person
    {
                [MapField("PersonID")]
                public int    ID;
                public string FirstName;
                public string MiddleName;
                public string LastName;
                public Gender Gender;
    } 

    public abstract class PersonAccessor : DataAccessor<Person, PersonAccessor>
    {
         [SqlQuery(@"
               SELECT 
                   p.PersonId,
                   p.FirstName,
                   p.SecondName,
                   p.MiddleName,
                   p.Gender
               FROM Person p
               WHERE p.PersonId = @PersonId")]
        public abstract Person GetPerson(int personId);
    }

    [Test]
    public void Test()
    {
        PersonAccessor pa = PersonAccessor.CreateInstance();

        Person p = pa.GetPerson(1);

        Assert.IsNotNull(p);
        Assert.AreEqual("John", p.FirstName);
    }
}
```

Как видно, класс PersonAccessor – абстрактный, абстрактным так же является его метод GetPerson. При обращении к PersonAccessor.CreateInstance() BLToolkit эмитит сборку, где определяет наследника от PersonAccessor и реализует абстрактные методы, ссылку на экземпляр этого наследника и возвращает CreateInstance(). Если посмотреть на реализацию этого класса (допустим, при помощи  Reflector), то мы увидим примерно следующее:

```csharp
[BLToolkitGenerated]
public class PersonAccessor : DataAccessorTest.PersonAccessor
{
    DataAccessorTest.Person person = null;
    DbManager dbManager = this.GetDbManager();
    try
    {
        Type type = typeof(DataAccessorTest.Person);
        IDbDataParameter[] parameters = new IDbDataParameter[] 
            { 
                dbManager.Parameter(
                    this.GetQueryParameterName(dbManager, "personId"), 
                    personId) };

        person = (DataAccessorTest.Person) dbManager
            .SetCommand("SELECT p.PersonId, p.FirstName, p.SecondName, p.MiddleName, p.Gender FROM Person WHERE p.PersonId = @PersonId", this.PrepareParameters(dbManager, parameters))
            .ExecuteObject(type);
    }
    finally
    {
        this.Dispose(dbManager);
    }
    return person;
}
```

Если не вдаваться в детали, то BLToolkit за нас написал примерно следующее: 

```csharp
using (DbManager db = new DbManager())
{
    return db
        .SetCommand@"
                     SELECT 
                         p.PersonId,
                         p.FirstName,
                         p.SecondName,
                         p.MiddleName,
                         p.Gender
                     FROM Person p
                     WHERE p.PersonId = @PersonId",
            db.Parameter("@PersonId", personId)
        .ExecuteObject(typeof(Person));
}
```

Иными словами, при помощи DataAccessor можно задекларировать какой код нам нужен для работы с данными, и по этой декларации BLToolkit сгенерирует за программиста этот код. Декларация происходит методом объявления наследника от DataAccessor и объявления в нем методов, каждая часть объявления метода имеет свое значение, а именно: 

- Возвращаемое значение.
- Имя метода.
- Параметры.
- Атрибуты, примененные к методу и его параметрам. 



<span id="dataaccessorreturntype"></span>
#### Возвращаемое значение

По типу возвращаемого значения определяется метод DbManager, который следует вызвать: 


| Тип | Метод |
|-|-|
| IDataReader | ExecuteDataReader |
| Наследник от DataSet | ExecuteDataSet |
| Наследник от DataTable | ExecuteDataTable |
| Наследник от IList | ExecuteList или ExecuteScalarList |
| Наследник от IDictionary | ExecuteDictionary или ExecuteScalarDictionary |
| void | ExecuteNonQuery |
| ValueType (int, string, byte[]) | ExecuteScalar |
| Иное | ExecuteObject |



<span id="dataaccessormethodname"></span>
#### Имя метода

По умолчанию DataAccessor использует хранимые процедуры. По имени метода определяется имя хранимой процедуры. Нотация следующая: [Имя типа]_[Имя Процедуры] 

```csharp
public abstract List<Person> SelectAll() // вызовет Person_SelectAll
public abstract void Insert(Person p) // вызовет Person_Insert
```

Переопределить подобное поведение можно написав своего наследника от DataAccessor и перегрузив в нем метод *GetDefaultSpName*.



<span id="dataaccessorparams"></span>
#### Параметры

Параметры используются как параметры запросов. Для параметра запроса используется имя параметра функции. Переопределить данное поведение можно, перегрузив функции GetQueryParameterName и/или GetSpParameterName. 

Если в качестве параметра передан не примитивный тип, то параметры из него будут созданы при помощи метода DbManager.CreateParameters(…). 

Если в качестве параметра будет передан экземпляр DbManager, то он будет использован для выполнения запроса. 



<span id="dataaccessorattributes"></span>
#### Атрибуты

Мощный механизм, позволяющий изменить поведение по умолчанию и сообщить генератору дополнительную информацию, в соответствии с которой будет изменен генерируемый код. 

**ActionNameAttribute** – позволяет задать имя действия для хранимой процедуры.

**ActionSprocNameAttribute** – позволяет ассоциировать метод аксессора с хранимой процедурой.

**SprocNameAttribute** – позволяет явно задать имя хранимой процедуры. 

Рассмотрим несколько примеров (все методы в результате вызывают хранимку Person_SelectAll): 

```csharp
[SprocActionName("GetPersons", "Person_SelectAll")]
public abstract class PersonAccessor : DataAccessor<Person, PersonAccessor>
{
    public abstract List<Person> SelectAll();
    
    [Action("SelectAll")]
    public abstract List<Person> GetAll();

    [SprocName("Person_SelectAll")]
    public abstract List<Person> LoadPersons();

    public abstract List<Person> GetPersons();
}
```

**DiscoverParametersAttribute** – при использовании хранимых процедур через DataAccessor по умолчанию полагается, что имена параметров функции совпадают с именем параметров хранимки, в таком случае порядок параметров в функции не имеет значения. Если же к функции применен данный атрибут BLToolkit получит информацию о параметрах хранимой процедуры и применит ее к параметрам метода по порядку, в данном случае имена параметров будут проигнорированы. 

```csharp
[TestFixture]
public class DiscoverParameters
{
    public abstract class PersonAccessor : DataAccessor
    {
        [DiscoverParameters]
        public abstract Person SelectByName(string anyParameterName, string rParameterName);
    }

    [Test]
    public void Test()
    {
        PersonAccessor pa = DataAccessor.CreateInstance<PersonAccessor>();
        Person         p  = pa.SelectByName("Tester", "Testerson");

        Assert.AreEqual(2, p.ID);
    }
}
```

**SqlQueryAttribute** – говорит методу, что следует выполнить указанный SQL запрос. Выше уже приводился стандартный метод использования данного атрибута. Кроме того, у атрибута есть свойство IsDynamic?. Если данное свойство выставлено в true то для получения кода запроса используется метод атрибута GetSqlText?(…). 

Рассмотрим пример: 

```csharp
public class TestQueryAttribute : SqlQueryAttribute
{
        public TestQueryAttribute()
        {
                IsDynamic = true;
        }

        private string _oracleText;
        public  string  OracleText
        {
                get { return _oracleText;  }
                set { _oracleText = value; }
        }

        public override string GetSqlText(DataAccessor accessor, DbManager dbManager)
        {
                switch (dbManager.DataProvider.Name)
                {
                        case "Sql"   :
                        case "SqlCe" : return SqlText;

                        case "Oracle":
                        case "ODP"   : return OracleText ?? SqlText;
                }

                throw new ApplicationException(string.Format("Unknown data ider '{0}'", dbManager.DataProvider.Name));
        }
}

public abstract class PersonAccessor : DataAccessor<Person, Person>
{
        [TestQuery(
                SqlText    = "SELECT * FROM Person WHERE LastName = @lastName",
                OracleText = "SELECT * FROM Person WHERE LastName = :lastName")]
        public abstract List<Person> SelectByLastName(string lastName);
}
```

В данном примере, в зависимости от имени используемого DataProvider-а выполняются разные запросы. 

**FormatAttribute** – применяется к параметру, указывает, что данный параметр используется для генерации текста SQL запроса или имени хранимой процедуры.

```csharp
public abstract class PersonAccessor : DataAccessor
{
    [SqlQuery("SELECT TOP {0} * FROM Person")]
    public abstract List<Person> GetPersonList([Format] int top);
}

[Test]
public void Test()
{
    PersonAccessor pa   = DataAccessor.CreateInstance<PersonAccessor>();
    List<Person>   list = pa.GetPersonList(2); // SELECT TOP 2 * FROM Person

    Assert.That(list,       Is.Not.Null);
    Assert.That(list.Count, Is.LessThanOrEqualTo(2));
}
```

**ParamNameAttribute** – позволяет явно задать имя параметра.

**ParamDbTypeAttribute** – позволяет явно задать тип параметра.

**ParamSizeAttribute** – позволяет явно задать размер параметра.

**ParamNullValueAttribute** – позволяет указать какое значение параметра считать за NULL. 

```csharp
public abstract class TestAccessor : DataAccessor
{

    [SqlQuery("SELECT * FROM Person WHERE PersonID = @personId")]
    public abstract Person SelectByKey([ParamName("personId")]int id);

    // при id == 1 значение параметра будет заменено на NULL
    public abstract Person SelectByKey([ParamNullValue(1)] int id);

    [SqlQuery("SELECT {0} = {1} FROM Person WHERE PersonID = 1")]
    public abstract void SelectJohn(
        [ParamSize(50), ParamDbType(DbType.String)] out string name,
        [Format] string paramName,
        [Format] string fieldName); 
}

[Test]
public void AccessorTest()
{
    using (DbManager db = new DbManager())
    {
        TestAccessor ta = DataAccessor.CreateInstance<TestAccessor>(db);

        string actualName;

        // SELECT @name  = FirstName FROM Person WHERE PersonID = 1
        // полученое значение будет возвращено в параметр name
        ta.SelectJohn(out actualName, "@name", "FirstName");

        Assert.AreEqual("John", actualName);
    }
}
```

**DirectionAttribute** – позволяет для параметра явно задать его направление.

**DestinationAttribute** – указывает, что в данный параметр следует отмапить выбранные значения. 

```csharp
public abstract class PersonAccessor : DataAccessor<Person, PersonAccessor>
{
    [SqlQuery("SELECT * FROM Person")]
    public abstract void SelectAll([Destination]List<Person> list);
    
    // CREATE Procedure Person_Insert_OutputParameter
    //   @FirstName  nvarchar(50),
    //   @LastName   nvarchar(50),
    //   @MiddleName nvarchar(50),
    //   @Gender     char(1),
    //   @PersonID   int output
    //   AS 
    //
    //     INSERT INTO Person
    //       ( LastName,  FirstName,  MiddleName,  Gender)
    //     VALUES
    //       (@LastName, @FirstName, @MiddleName, @Gender)
    //
    //     SET @PersonID = Cast(SCOPE_IDENTITY() as int)
    [SprocName("Person_Insert_OutputParameter")]
    public abstract void Insert([Direction.Output("PersonId")] Person p);
}
```

**IndexAttribute** – позволяет указать индекс для словаря (по умолчанию используется первичный ключ): 

```csharp
public abstract class PersonAccessor : DataAccessor<Person>
{
    // Для ключа словаря будет использован первичный ключ объекта Person,
    // т.е. поля, помеченные атрибутом PrimaryKey
    [ActionName("SelectAll")]
    public abstract Dictionary<int,Person> GetPersonDictionary1();

    // Явно задаем индекс. ID – поле класса Person.
    //
    [ActionName("SelectAll")]
    [Index("ID")]
    public abstract Dictionary<int,Person> GetPersonDictionary2();

    // Явно задаем индекс. 
    // "@PersonID"- поле полученного в результате выборки кортежа!.
    // Важно: собачка - '@' заставляет BLToolkit для индекса 
    // брать значения из полей кортежа!
    //
    [ActionName("SelectAll")]
    [Index("@PersonID")]
    public abstract Dictionary<int,Person> GetPersonDictionary3();

    // Будет вычитан словарь со скалярнми величинами.
    //
    [SqlQuery("SELECT PersonID, FirstName FROM Person")]
    [Index("PersonID")]
    [ScalarFieldName("FirstName")]
    public abstract Dictionary<int,string> GetPersonNameDictionary();
}
```

**ObjectTypeAttribute** – явно задает тип возвращаемого абстрактным методом объекта.

**ActualTypeAttribute** – явно задает тип возвращаемого абстрактным методом объекта, имеет более низкий приоритет чем ObjectType.

```csharp
public interface IName
{
    string Name { get; }
}

public class NameBase : IName
{
    private string _name;
    public  string  Name { get { return _name; } set { _name = value; } }
}

public class Name1 : NameBase {}
public class Name2 : NameBase {}

[ActualType(typeof(IName), typeof(Name1))]
public abstract class TestAccessor : DataAccessor
{
    // Вернет объект класса Name1
    [SqlQuery("SELECT 'John' as Name")]
    public abstract IName GetName1();

    // Вернет объект класса Name2
    [SqlQuery("SELECT 'John' as Name"), ObjectType(typeof(Name2))]
    public abstract IName GetName2();

    [SqlQuery("SELECT 'John' as Name")]
    public abstract IList<IName> GetName1List();

    [SqlQuery("SELECT 'John' as Name"), ObjectType(typeof(Name2))]
    public abstract IList<IName> GetName2List();

    [SqlQuery("SELECT 1 as ID, 'John' as Name"), Index("@ID")]
    public abstract IDictionary<int, IName> GetName1Dictionary();

    [SqlQuery("SELECT 1 as ID, 'John' as Name"), Index("@ID"),
     ObjectType(typeof(Name2))]
    public abstract IDictionary<int, IName> GetName2Dictionary();
}


[ObjectType(typeof(Person)]
public abstract class PersonAccessor : DataAccessor
{
    public abstract ArrayList SelectAll();

    public abstract object SelectByKey(int personId);
}
```



<span id="dataaccessrecommends"></span>
### Рекомендации по использванию


<span id="dataaccessrecommendsmanual"></span>
#### Реализация методов ручками

Эмит кода, это конечно хорошо, но переодически возникает необходимость сделать метод руками, в таком случае рекомендуется делать это так: 

```csharp
public class MyAccessor : DataAccessor
{
    public List<Person> SelectAll()
    {
        DbManager db = GetDbManager();
        try
        {
            return SelectAll(db);
        }
        finally
        {
            Dispose(db);
        }
    }

    public SelectAll(DbManager db)
    {
        return db.SetCommand("SELECT * FROM Person").ExecuteList<Person>();
    }
}
```

Это типовой шаблон реализации методов как для всех наследников DataAccessorBase, коими являются как DataAccessor так и SqlQuery. Ключевыми являются использование функций GetDbManager() и Dispose(DbManager db). Первый возвращает эеземпляр DbManager, переданный в конструктор, если передавался, иначе новый экзкмпляр. Второй освобождает экземпляр DbManager, в случае если оный не был передан через конструктор. 



<span id="dataaccessrecommendsabstract"></span>
#### Генерация SQL запросов в абстрактном аксессоре

Абстрактные аксессоры не поддерживают генерацию SQL запросов, однако, при необходимости можно реализовать своего наследника, допустим, следующего вида: 

```csharp
public abstract class MyAccessorBase<T, TA> : DataAccessor<T, TA> where TA : DataAccessor<T>
{
        SqlQuery<T> _query = new SqlQuery<T>();

        private delegate R Func<P1, P2, R>(P1 par1, P2 par2);
        private delegate R Func<P1, R>(P1 par1);

        private R Exec<R, P>(Func<DbManager, P, R> op, P obj)
        {
                DbManager db = GetDbManager();
                try
                {
                        return op(db, obj);
                }
                finally
                {
                        Dispose(db);
                }
        }

        private R Exec<R>(Func<DbManager, R> op)
        {
                DbManager db = GetDbManager();
                try
                {
                        return op(db);
                }
                finally
                {
                        Dispose(db);
                }
        }

        public virtual int  Insert(T obj)
        {
                return Exec<int, T>(Insert, obj);
        }
                        
        public virtual int Insert(DbManager db, T obj)
        {
                return _query.Insert(db, obj);
        }

        public virtual int Update(T obj)
        {
                return Exec<int, T>(Update, obj);
        }

        public virtual int Update(DbManager db, T obj)
        {
                return _query.Update(db, obj);
        }

        public virtual int Delete(T obj)
        {
                return Exec<int, T>(Delete, obj);
        }
                        
        public virtual int Delete(DbManager db, T obj)
        {
                return _query.Delete(db, obj);
        }

        public virtual int DeleteByKey(object[] keys)
        {
                return Exec<int, object[]>(DeleteByKey, keys);
        }

        public virtual int DeleteByKey(DbManager db, object[] keys)
        {
                return _query.DeleteByKey(db, keys);
        }

        public virtual List<T> SelectAll()
        {
                return Exec<List<T>>(SelectAll);
        }

        public virtual List<T> SelectAll(DbManager db)
        {
                return _query.SelectAll(db);
        }

        public virtual T SelectByKey(object[] keys)
        {
                return Exec<T, object[]>(SelectByKey, keys);
        }

        public virtual T SelectByKey(DbManager db, object[] keys)
        {
                return _query.SelectByKey(db, keys);
        }
}
```




## Исходники

Тут описание структуры исходников, что где и зачем.



<span id="materials"></span>
## При написании использовалось

- Редактор markdown [MarkdownPad](http://markdownpad.com/).
- Подсветка синтаксиса [http://hilite.me/](http://hilite.me/).
- Кэш из вебархива ([http://web.archive.org/web/20131106072643/http://projects.rsdn.ru/RFD/wiki/BLToolkit](http://web.archive.org/web/20131106072643/http://projects.rsdn.ru/RFD/wiki/BLToolkit)) оригинальной страницы [http://projects.rsdn.ru/RFD/wiki/BLToolkit](http://projects.rsdn.ru/RFD/wiki/BLToolkit).
- Оригинальная документация [http://files.rsdn.ru/49168/BLToolkit.doc](http://files.rsdn.ru/49168/BLToolkit.doc).
- Исходник этой статьи на markdown - [https://raw.githubusercontent.com/liiws/BLToolkitDocs/master/bltoolkit.md](https://raw.githubusercontent.com/liiws/BLToolkitDocs/master/bltoolkit.md).

