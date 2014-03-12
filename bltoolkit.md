# Business Logic Toolkit

Основная задача статьи дать вводную информацию о библиотеке BLToolkit. Она поможет Вам сделать первый шаг в обуздании этого маленького монстра. Данная статья основана на оригинальной статье [Игоря Ткачева "Пространство имен Rsdn.Framework.Data"](http://rsdn.ru/article/files/libs/RsdnFrameworkData.xml) и, как следствие, содержит части оной.



## От [liiws](https://bitbucket.org/liiws)

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

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">CREATE</span> <span style="color: #0000ff">TABLE</span> Person
(
	PersonID   int          <span style="color: #0000ff">NOT</span> <span style="color: #0000ff">NULL</span> <span style="color: #0000ff">IDENTITY</span>(1,1) <span style="color: #0000ff">CONSTRAINT</span> PK_Person <span style="color: #0000ff">PRIMARY</span> <span style="color: #0000ff">KEY</span> CLUSTERED,
	FirstName  nvarchar(50) <span style="color: #0000ff">NOT</span> <span style="color: #0000ff">NULL</span>,
	LastName   nvarchar(50) <span style="color: #0000ff">NOT</span> <span style="color: #0000ff">NULL</span>,
	MiddleName nvarchar(50)     <span style="color: #0000ff">NULL</span>,
	Gender     char(1)      <span style="color: #0000ff">NOT</span> <span style="color: #0000ff">NULL</span> <span style="color: #0000ff">CONSTRAINT</span> CK_Person_Gender <span style="color: #0000ff">CHECK</span> (Gender <span style="color: #0000ff">in</span> (<span style="color: #a31515">&#39;M&#39;</span>, <span style="color: #a31515">&#39;F&#39;</span>, <span style="color: #a31515">&#39;U&#39;</span>, <span style="color: #a31515">&#39;O&#39;</span>))
)
<span style="color: #0000ff">ON</span> [<span style="color: #0000ff">PRIMARY</span>]
</pre></div>

Код:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> <span style="color: #0000ff">enum</span> Gender
{
	Female,
	Male,
	Unknown,
	Other
}
<span style="color: #0000ff">public</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">Person</span>
{
	<span style="color: #0000ff">public</span> <span style="color: #2b91af">int</span>    ID;
	<span style="color: #0000ff">public</span> <span style="color: #2b91af">string</span> FirstName;
	<span style="color: #0000ff">public</span> <span style="color: #2b91af">string</span> MiddleName;
	<span style="color: #0000ff">public</span> <span style="color: #2b91af">string</span> LastName;
	<span style="color: #0000ff">public</span> Gender Gender;
}

Person GetPerson(<span style="color: #2b91af">int</span> personId)
{
    <span style="color: #2b91af">string</span> connectionString =
        <span style="color: #a31515">&quot;Server=.;Database=BLToolkit;Integrated Security=SSPI&quot;</span>;
    <span style="color: #2b91af">string</span> commandText = <span style="color: #a31515">@&quot;</span>
<span style="color: #a31515">        SELECT </span>
<span style="color: #a31515">            p.PersonId,</span>
<span style="color: #a31515">            p.FirstName,</span>
<span style="color: #a31515">            p.SecondName,</span>
<span style="color: #a31515">            p.MiddleName,</span>
<span style="color: #a31515">            p.Gender</span>
<span style="color: #a31515">        FROM Person p</span>
<span style="color: #a31515">        WHERE p.PersonId = @PersonId&quot;</span>;

    <span style="color: #0000ff">using</span> (SqlConnection con = <span style="color: #0000ff">new</span> SqlConnection(connectionString))
    {
        con.Open();

        <span style="color: #0000ff">using</span> (SqlCommand cmd = <span style="color: #0000ff">new</span> SqlCommand(commandText, con))
        {
            cmd.Parameters.Add(<span style="color: #a31515">&quot;@min&quot;</span>, min);

            <span style="color: #0000ff">using</span> (SqlDataReader rd = cmd.ExecuteReader())
            {
                Person p = <span style="color: #0000ff">null</span>;

                <span style="color: #0000ff">if</span> (rd.Read())
                {
                    p = <span style="color: #0000ff">new</span> Person();

                    p.ID           = Convert.ToInt32 (rd[<span style="color: #a31515">&quot;PersonId&quot;</span>]);
                    p.FirstName    = Convert.ToString(rd[<span style="color: #a31515">&quot;FirstName&quot;</span>]);
                    p.SecondName   = Convert.ToString(rd[<span style="color: #a31515">&quot;SecondName&quot;</span>]);
                    p.MiddleName   = Convert.ToString(rd[<span style="color: #a31515">&quot;ThirdName&quot;</span>]);
                    <span style="color: #2b91af">string</span> gender  = Convert.ToString(rd[<span style="color: #a31515">&quot;Gender&quot;</span>]);

                    <span style="color: #0000ff">switch</span>(gender)
                    {
                        <span style="color: #0000ff">case</span> <span style="color: #a31515">&quot;M&quot;</span>:
                            p.Gender = Gender.Male;
                            <span style="color: #0000ff">break</span>;
                        <span style="color: #0000ff">case</span> <span style="color: #a31515">&quot;F&quot;</span>:
                            p.Gender = Gender.Female;
                            <span style="color: #0000ff">break</span>;
                        <span style="color: #0000ff">case</span> <span style="color: #a31515">&quot;U&quot;</span>:
                            p.Gender = Gender.Unknown;
                            <span style="color: #0000ff">break</span>;
                        <span style="color: #0000ff">case</span> <span style="color: #a31515">&quot;0&quot;</span>:
                            p.Gender = Gender.Other;
                            <span style="color: #0000ff">break</span>;
                    }
                }
                <span style="color: #0000ff">return</span> p;
            }
        }
    }
}
</pre></div>

А теперь то же самое, в исполнении BLToolkit:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> <span style="color: #0000ff">enum</span> Gender
{
	[MapValue(&quot;F&quot;)] Female,
	[MapValue(&quot;M&quot;)] Male,
	[MapValue(&quot;U&quot;)] Unknown,
	[MapValue(&quot;O&quot;)] Other
}
<span style="color: #0000ff">public</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">Person</span>
{
	[MapField(&quot;PersonID&quot;)]
	<span style="color: #0000ff">public</span> <span style="color: #2b91af">int</span>    ID;
	<span style="color: #0000ff">public</span> <span style="color: #2b91af">string</span> FirstName;
	<span style="color: #0000ff">public</span> <span style="color: #2b91af">string</span> MiddleName;
	<span style="color: #0000ff">public</span> <span style="color: #2b91af">string</span> LastName;
	<span style="color: #0000ff">public</span> Gender Gender;
}

Person GetPerson(<span style="color: #2b91af">int</span> personId)
{
    <span style="color: #0000ff">using</span> (DbManager db = <span style="color: #0000ff">new</span> DbManager())
    {
        <span style="color: #0000ff">return</span> db
            .SetCommand<span style="color: #a31515">@&quot;</span>
<span style="color: #a31515">                         SELECT </span>
<span style="color: #a31515">                             p.PersonId,</span>
<span style="color: #a31515">                             p.FirstName,</span>
<span style="color: #a31515">                             p.SecondName,</span>
<span style="color: #a31515">                             p.MiddleName,</span>
<span style="color: #a31515">                             p.Gender</span>
<span style="color: #a31515">                         FROM Person p</span>
<span style="color: #a31515">                         WHERE p.PersonId = @PersonId&quot;</span>,
                db.Parameter(<span style="color: #a31515">&quot;@PersonId&quot;</span>, personId)
            .ExecuteObject(<span style="color: #0000ff">typeof</span>(Person));
    }
}
</pre></div>



<span id="someoracle"></span>
### Те же и Oracle

Если вы начинаете тренироваться с BLToolkit на базе Oracle, последний фрагмент кода должен иметь вид: 

<!-- HTML generated using hilite.me --><div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%">Person GetPerson(<span style="color: #2b91af">int</span> personId)
{
   <span style="color: #0000ff">using</span> (DbManager db = <span style="color: #0000ff">new</span> DbManager())
   {
      <span style="color: #0000ff">return</span> db
        .SetCommand( <span style="color: #a31515">@&quot;</span>
<span style="color: #a31515">               SELECT </span>
<span style="color: #a31515">                  p.PersonId,</span>
<span style="color: #a31515">                  p.FirstName,</span>
<span style="color: #a31515">                  p.LastName,</span>
<span style="color: #a31515">                  p.MiddleName,</span>
<span style="color: #a31515">                  p.Gender</span>
<span style="color: #a31515">               FROM Person p</span>
<span style="color: #a31515">               WHERE p.PersonId = :PersonId&quot;</span>,
           db.Parameter(<span style="color: #a31515">&quot;PersonId&quot;</span>, personId))
       .ExecuteObject&lt;Person&gt;();
   }
}
...
   <span style="color: #2b91af">var</span> p = <span style="color: #0000ff">new</span> BLToolkit.Data.DataProvider.OracleDataProvider();
   p.ParameterPrefix = <span style="color: #0000ff">null</span>;
   DbManager.AddDataProvider(p);
   DbManager.AddConnectionString(<span style="color: #a31515">&quot;Oracle&quot;</span>,
      <span style="color: #2b91af">string</span>.Format(<span style="color: #a31515">&quot;Data Source={0};User ID={1};Password={2};&quot;</span>,
      oracleAlias, oracleUser, oraclePassword));
   
   <span style="color: #2b91af">var</span> person = GetPerson(1);
</pre></div>


Не трудно заметить, что последний вариант заметно короче. Фактически все, что у нас осталось – это текст самого запроса. Класс DbManager самостоятельно осуществляет всю работу по созданию объекта и отображению (mapping) полей рекордсета на заданную структуру.

Вообще, забегая вперед добиться тех же успехов можно и более лаконичным (и техничным) путем:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">PersonAccessor</span>&lt;Person, PersonAccessor&gt;
{
    [SqlQuery@&quot;
               SELECT 
                   p.PersonId,
                   p.FirstName,
                   p.SecondName,
                   p.MiddleName,
                   p.Gender
               FROM Person p
               WHERE p.PersonId = @PersonId&quot;)]
    <span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> Person GetPerson(<span style="color: #2b91af">int</span> personId){}
}

<span style="color: #008000">//и уже где-то совсем в другом месте программы</span>
<span style="color: #008000">//получим нужного человека:</span>

Person p = PersonAccessor.CreateInstance().GetPerson (10);

<span style="color: #008000">//...</span>
</pre></div>

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

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> DbManager();

<span style="color: #0000ff">public</span> DbManager(
    <span style="color: #2b91af">string</span> configurationString
    );

<span style="color: #0000ff">public</span> DbManager(
    <span style="color: #2b91af">string</span> providerName, 
    <span style="color: #2b91af">string</span> configuration
    );

<span style="color: #0000ff">public</span> DbManager(
    IDbConnection connection
    );

<span style="color: #0000ff">public</span> DbManager(
    IDbTransaction transaction
    );

<span style="color: #0000ff">public</span> DbManager(
    DataProviderBase dataProvider, 
    <span style="color: #2b91af">string</span>           connectionString
    );

<span style="color: #0000ff">public</span> DbManager(
    DataProviderBase dataProvider, 
    IDbConnection    connection
    );

<span style="color: #0000ff">public</span> DbManager(
    DataProviderBase dataProvider, 
    IDbTransaction   transaction
    );
</pre></div>

Остановимся подробней на следующих параметрах (прочие параметры не должны вызвать вопросов у тех, кто хотя бы поверхностно знаком с ADO .NET):

- *configurationString* – это не строка соединения (Connecting String), а ключ, по которому строка соединения будет читаться из файла конфигурации.
- *providerName* – так же ключ, является именем дата провайдера, который следует использовать.
- *dataProvider* – экземпляр дата провайдера, который следует использовать. Механизм дата провайдеров достоин отдельного обсуждения, что и будет сделано ниже.

Рассмотрим подробнее правила работы с файлом конфигурации:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%">&lt;appSettings&gt;
<span style="color: #0000ff">&lt;!—- Конфигурация по умолчанию --&gt;</span>
&lt;add
		key   = <span style="color: #a31515">&quot;ConnectionString&quot;</span>
		value = <span style="color: #a31515">&quot;Server=.;Database=BLToolkitData;Integrated Security=SSPI&quot;</span>/&gt;
<span style="color: #0000ff">&lt;!—- Конфигурация Development для SQL Server --&gt;</span>
&lt;add
		key   = <span style="color: #a31515">&quot;ConnectionString.Development&quot;</span>
		value = <span style="color: #a31515">&quot;Server=.;Database=BLToolkitData;Integrated Security=SSPI&quot;</span>/&gt;
<span style="color: #008000">&lt;!-- Конфигурация Production для SQL Server --&gt;</span>
&lt;add
		key   = <span style="color: #a31515">&quot;ConnectionString.Production&quot;</span>
		value = <span style="color: #a31515">&quot;Server=.;Database=BLToolkitData;Integrated Security=SSPI&quot;</span>/&gt;
<span style="color: #008000">&lt;!-- Конфигурация для SQL Server --&gt;</span>
&lt;add
		key   = <span style="color: #a31515">&quot;ConnectionString.Sql&quot;</span>
		value = <span style="color: #a31515">&quot;Server=.;Database=BLToolkitData;Integrated Security=SSPI&quot;</span>/&gt;
<span style="color: #008000">&lt;!-- Конфигурация для Oracle --&gt;</span>
&lt;add
		key   = <span style="color: #a31515">&quot;ConnectionString.Oracle&quot;</span>
		value = <span style="color: #a31515">&quot;User Id=/;Data Source=BLToolkitData&quot;</span>/&gt;
<span style="color: #008000">&lt;!-- Конфигурация OLEDB --&gt;</span>
&lt;add
		key   = <span style="color: #a31515">&quot;ConnectionString.OleDb&quot;</span>
		value = <span style="color: #a31515">&quot;Provider=SQLNCLI.1;Data Source=.;Integrated Security=SSPI;Initial Catalog=BLToolkitData&quot;</span>/&gt;
<span style="color: #008000">&lt;!-- Конфигурация Development для OLEDB --&gt;</span>
&lt;add
		key   = <span style="color: #a31515">&quot;ConnectionString.OleDb.Development&quot;</span>
		value = <span style="color: #a31515">&quot;Provider=SQLNCLI.1;Data Source=.;Integrated Security=SSPI;Initial Catalog=BLToolkitData&quot;</span>/&gt;
<span style="color: #008000">&lt;!-- Конфигурация Production для OLEDB --&gt;</span>
&lt;add
		key   = <span style="color: #a31515">&quot;ConnectionString.OleDb.Production&quot;</span>
		value = <span style="color: #a31515">&quot;Provider=SQLNCLI.1;Data Source=.;Integrated Security=SSPI;Initial Catalog=BLToolkitData&quot;</span>/&gt;
&lt;/appSettings&gt;
</pre></div>

Как видим, поле key – содержит ключевое значение ConntctionString и разделенные c ним через точку *configurationString* и *providerName*.

Рассмотрим следующие примеры создания DbManager:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #008000">// Использование конфигурации по умолчанию.</span>
DbManager db = <span style="color: #0000ff">new</span> DbManager();

<span style="color: #008000">// Использование конфигурации для Sql Server</span>
<span style="color: #008000">// аналогично для Oracle DbManager (&quot;Oracle&quot;)</span>
DbManager db = <span style="color: #0000ff">new</span> DbManager(<span style="color: #a31515">&quot;Sql&quot;</span>);

<span style="color: #008000">// Использование конфигурации Development для Sql Server</span>
<span style="color: #008000">// аналогично Production для OLEDB DbManager (&quot;OleDb&quot; &amp; &quot;Production&quot;)</span>
DbManager db = <span style="color: #0000ff">new</span> DbManager(<span style="color: #a31515">&quot;Sql&quot;</span>, <span style="color: #a31515">&quot;Development&quot;</span>);
</pre></div>

Дополнительно есть возможность указать конфигурацию по умолчанию. Если в файл конфигурации добавить вот такую секцию:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%">&lt;appSettings&gt;
    &lt;add
        key   = <span style="color: #a31515">&quot;BLToolkit.DefaultConfiguration&quot;</span>
        value = <span style="color: #a31515">&quot;Oracle&quot;</span>/&gt;
&lt;/appSettings&gt;
</pre></div>

То вызов конструктора без параметров будет аналогичен вызову DbManager(“Oracle”). 

Таким образом мы можем работать с различными конфигурациями и базами данных. Секция *appSettings* может находиться как в *app.config* или *web.config* так и *machine.config* файле.

Если же вам не хочется возиться с конфигурационными файлами, то для задания строки соединения можно воспользоваться методом AddConnectionString:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%">DbManager.AddConnectionString(<span style="color: #a31515">&quot;MyConfig&quot;</span>, connectionString);

<span style="color: #0000ff">using</span> (DbManager db = <span style="color: #0000ff">new</span> DbManager(<span style="color: #a31515">&quot;MyConfig&quot;</span>))
{
    <span style="color: #008000">// ...</span>
}
</pre></div>

или

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%">DbManager.AddConnectionString(connectionString);

<span style="color: #0000ff">using</span> (DbManager db = <span style="color: #0000ff">new</span> DbManager())
{
    <span style="color: #008000">// ...</span>
}
</pre></div>

Метод AddConnectionString достаточно вызвать один раз для каждой конфигурации в начале программы.



<span id="dataprovider"></span>
### Механизм дата провайдеров

Отличительной особенностью класса DbManager является то, что он работает исключительно с интерфейсами пространства имён System.Data и вполне может использоваться для работы с различными провайдерами данных. На данный момент поддерживается работа с Data Provider for SQL Server, Data Provider for Oracle, Data Provider for OLE DB и Data Provider for ODBC. Выбор провайдера осуществляется также с помощью строки конфигурации. Для этого достаточно добавить к ней один из следующих постфиксов: “.OleDb”, “.Odbc”, “.Oracle”, “.Sql”. Если постфикс не задан, то по умолчанию выбирается провайдер для SQL Server.

К вопросам о мифичности и реальности поддержки в одном проекте различных баз данных. Исходный код BLToolkit покрыт блочными тестами. При этом есть возможность в качестве тестовой базы данных использовать: MS Sql Server, Access, Oracle (используется OdpDataProvider .\Source\Data\DataProvider\OdpDataProvider.cs), Firebird (используется FdpDataProvider .\Source\Data\DataProvider\FdpDataProvider.cs). Так же примером может служить проект [RSDN@HOME](http://rsdn.ru/projects/janus/article/article.xml), где механизмами BLT осуществлена поддержка нескольких БД.

Таким образом, механизм дата провайдеров позволяет абстрагировать DbManager от специфики конкретного клиента и его реализации. Для примера можно рассмотреть OdpDataProvider.

В дополнение к существующим провайдерам совсем несложно подключить любой другой. Следующий пример демонстрирует подключение Borland Data Providers for .NET (BDP.NET):

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">using</span> System;
<span style="color: #0000ff">using</span> System.Data;
<span style="color: #0000ff">using</span> System.Data.Common;

<span style="color: #0000ff">using</span> Borland.Data.Provider;

<span style="color: #0000ff">using</span> Rsdn.Framework.Data;
<span style="color: #0000ff">using</span> Rsdn.Framework.Data.DataProvider;

<span style="color: #0000ff">namespace</span> Example
{
    <span style="color: #0000ff">public</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">BdpDataProvider</span>: IDataProvider
    {
        IDbConnection IDataProvider.CreateConnectionObject()
        {
            <span style="color: #0000ff">return</span> <span style="color: #0000ff">new</span> BdpConnection();
        }

        DbDataAdapter IDataProvider.CreateDataAdapterObject()
        {
            <span style="color: #0000ff">return</span> <span style="color: #0000ff">new</span> BdpDataAdapter();
        }

        <span style="color: #0000ff">void</span> IDataProvider.DeriveParameters(IDbCommand command)
        {
            BdpCommandBuilder.DeriveParameters((BdpCommand)command);
        }

        Type IDataProvider.ConnectionType
        {
            <span style="color: #0000ff">get</span>
            {
                <span style="color: #0000ff">return</span> typeof(BdpConnection);
            }
        }

        <span style="color: #2b91af">string</span> IDataProvider.Name
        {
            <span style="color: #0000ff">get</span>
            {
                <span style="color: #0000ff">return</span> <span style="color: #a31515">&quot;Bdp&quot;</span>;
            }
        }
    }

    <span style="color: #0000ff">class</span> <span style="color: #2b91af">Test</span>
    {
        <span style="color: #0000ff">static</span> <span style="color: #0000ff">void</span> Main()
        {
            DbManager.AddDataProvider(<span style="color: #0000ff">new</span> BdpDataProvider());
            DbManager.AddConnectionString(<span style="color: #a31515">&quot;.bdp&quot;</span>,
                <span style="color: #a31515">&quot;assembly=Borland.Data.Mssql,Version=1.1.0.0, &quot;</span> +
                <span style="color: #a31515">&quot;Culture=neutral,PublicKeyToken=91d62ebb5b0d1b1b;&quot;</span> +
                <span style="color: #a31515">&quot;vendorclient=sqloledb.dll;osauthentication=True;&quot;</span> +
                <span style="color: #a31515">&quot;database=Northwind;hostname=localhost;provider=MSSQL&quot;</span>);

            <span style="color: #0000ff">using</span> (DbManager db = <span style="color: #0000ff">new</span> DbManager())
            {
                <span style="color: #2b91af">int</span> count = (<span style="color: #2b91af">int</span>)db
                    .SetCommand(<span style="color: #a31515">&quot;SELECT Count(*) FROM Categories&quot;</span>)
                    .ExecuteScalar();

                Console.WriteLine(count);
            }
        }
    }
}
</pre></div>



<span id="parameters"></span>
### Параметры

Большинство используемых запросов требуют тот или иной набор параметров для своего выполнения. В приведённом выше примере таким параметром является @personId – идентификатор человека в базе. Зачастую, среднеленивый программист предпочитает использовать в подобных случаях обычную конкатенацию строк, т.е. что-то наподобие следующего:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">void</span> Test(<span style="color: #2b91af">int</span> id)
{
    <span style="color: #2b91af">string</span> commandText = <span style="color: #a31515">@&quot;</span>
<span style="color: #a31515">        SELECT FirstName</span>
<span style="color: #a31515">        FROM   Person</span>
<span style="color: #a31515">        WHERE  PersonId = &quot;</span> + id;

    <span style="color: #008000">// ...</span>
}
</pre></div>

К сожалению, при всей своей простоте, такой стиль плохо читаем, часто ведёт к непредсказуемым ошибкам и долгим мучениям с подбором формата, если в качестве параметра, например, используется дата. Более того, если наш параметр имеет строковый тип, то применение такого подхода в Web-приложениях может сделать их весьма уязвимыми для хакерских атак. Поэтому, отложим шутки в сторону и серьёзно займёмся рассмотрением возможностей, предоставляемых классом DbManager для работы с параметрами.

Для создания параметров служит следующий набор методов:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> IDbDataParameter Parameter(
	<span style="color: #2b91af">string</span> parameterName,
	<span style="color: #2b91af">object</span> <span style="color: #0000ff">value</span>
);

<span style="color: #0000ff">public</span> IDbDataParameter InputParameter(
	<span style="color: #2b91af">string</span> parameterName,
	<span style="color: #2b91af">object</span> <span style="color: #0000ff">value</span>
);
</pre></div>

Создаёт входной (*ParameterDirection.Input*) параметр с именем *parameterName* и значением *value*.

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> IDbDataParameter NullParameter(
    <span style="color: #2b91af">string</span> parameterName,
    <span style="color: #2b91af">object</span> <span style="color: #0000ff">value</span>
);
</pre></div>

Делает тоже, что и предыдущие методы и в дополнение проверяет значение *value*. Если оно представляет собой *null*, пустую строку, значение даты *DateTime.MinValue* или 0 для целых типов, то вместо заданного значения подставляется *DBNull.Value*.

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> IDbDataParameter OutputParameter(
    <span style="color: #2b91af">string</span> parameterName,
    <span style="color: #2b91af">object</span> <span style="color: #0000ff">value</span>
);
</pre></div>

Создаёт выходной (*ParameterDirection.Output*) параметр.

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> IDbDataParameter InputOutputParameter(
    <span style="color: #2b91af">string</span> parameterName,
    <span style="color: #2b91af">object</span> <span style="color: #0000ff">value</span>
);
</pre></div>

Создаёт параметр, работающий как входной и выходной (*ParameterDirection.InputOutput*).

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> IDbDataParameter ReturnValue(
    <span style="color: #2b91af">string</span> parameterName
);
</pre></div>

Создаёт параметр-возвращаемое значение (*ParameterDirection.ReturnValue*).

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> IDbDataParameter Parameter(
    ParameterDirection parameterDirection,
    <span style="color: #2b91af">string</span> parameterName,
    <span style="color: #2b91af">object</span> <span style="color: #0000ff">value</span>
);
</pre></div>

Создаёт параметр с заданными значениями.

Создание выходных параметров и возвращаемое значение используются для работы с сохранёнными процедурами. Входной параметр можно использовать для построения любых запросов.

Для чтения выходных параметров после выполнения запроса служит следующий метод:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> IDbDataParameter Parameter(
    <span style="color: #2b91af">string</span> parameterName
);
</pre></div>

Каждая версия метода Execute… имеет в своём составе метод, принимающий в качестве последнего аргумента список параметров запроса. Например, для ExecuteNonQuery одна из таких функций имеет следующий вид:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> <span style="color: #2b91af">int</span> ExecuteNonQuery(
    <span style="color: #2b91af">string</span> commandText,
    <span style="color: #0000ff">params</span> IDbDataParameter[] commandParameters
);
</pre></div>

Таким образом, список параметров задаётся простым перечислением через запятую (с таблицей Region и примерами с ней связанными я отойду от правила использовать БД BLToolkit):

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">void</span> InsertRegion(<span style="color: #2b91af">int</span> id, <span style="color: #2b91af">string</span> description)
{
    <span style="color: #0000ff">using</span> (DbManager db = <span style="color: #0000ff">new</span> DbManager())
    {
        db
            .SetCommand(<span style="color: #a31515">@&quot;</span>
<span style="color: #a31515">                INSERT INTO Region (</span>
<span style="color: #a31515">                    RegionID,</span>
<span style="color: #a31515">                    RegionDescription</span>
<span style="color: #a31515">                ) VALUES (</span>
<span style="color: #a31515">                    @id,</span>
<span style="color: #a31515">                    @desc</span>
<span style="color: #a31515">                )&quot;</span>,
                db.Parameter(<span style="color: #a31515">&quot;@id&quot;</span>,   id),
                db.Parameter(<span style="color: #a31515">&quot;@desc&quot;</span>, description))
            .ExecuteNonQuery();
    }
}
</pre></div>

Для создания списка параметров из бизнес объектов существует метод CreateParameters, который принимает в качестве аргумента объект DataRow или любой бизнес-объект. Допустим, у нас имеется класс Region, содержащий информацию о регионе. В этом случае мы могли бы переписать предыдущий пример следующим образом:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">Region</span>
{
    <span style="color: #0000ff">public</span> <span style="color: #2b91af">int</span>    ID;
    <span style="color: #0000ff">public</span> <span style="color: #2b91af">string</span> Description;
}

<span style="color: #0000ff">void</span> InsertRegion(Region region)
{
    <span style="color: #0000ff">using</span> (DbManager db = <span style="color: #0000ff">new</span> DbManager())
    {
        db
            .SetCommand(<span style="color: #a31515">@&quot;</span>
<span style="color: #a31515">                INSERT INTO Region (</span>
<span style="color: #a31515">                    RegionID,</span>
<span style="color: #a31515">                    RegionDescription</span>
<span style="color: #a31515">                ) VALUES (</span>
<span style="color: #a31515">                    @ID,</span>
<span style="color: #a31515">                    @Description</span>
<span style="color: #a31515">                )&quot;</span>,
                db.CreateParameters(region)).
            .ExecuteNonQuery();
    }
}
</pre></div>

Более общий вид функции CreateParameters для бизнес объекта (аналогично для DataRow) выглядит следующим образом:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> IDbDataParameter[] CreateParameters(
	<span style="color: #2b91af">object</span>                    obj,
	<span style="color: #2b91af">string</span>[]                  outputParameters,
	<span style="color: #2b91af">string</span>[]                  inputOutputParameters,
	<span style="color: #2b91af">string</span>[]                  ignoreParameters,
	<span style="color: #0000ff">params</span> IDbDataParameter[] commandParameters);
</pre></div>

Подобный вызов позволит явно задать параметрам  по их именам их направления и, при необходимости, указать дополнительные параметры:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">Region</span>
{
    <span style="color: #0000ff">public</span> <span style="color: #2b91af">int</span>    ID;
    <span style="color: #0000ff">public</span> <span style="color: #2b91af">string</span> Description;
}

<span style="color: #0000ff">void</span> InsertRegion(Region region)
{
    <span style="color: #0000ff">using</span> (DbManager db = <span style="color: #0000ff">new</span> DbManager())
    {
        db
            .SetCommand(<span style="color: #a31515">@&quot;</span>
<span style="color: #a31515">                INSERT INTO Region (</span>
<span style="color: #a31515">                    RegionDescription</span>
<span style="color: #a31515">                ) VALUES (</span>
<span style="color: #a31515">                    @Description</span>
<span style="color: #a31515">                )</span>
<span style="color: #a31515">                SELECT Cast(SCOPE_IDENTITY() as int) ID&quot;</span>,
                db.CreateParameters(region, <span style="color: #0000ff">new</span> <span style="color: #2b91af">string</span>[]{<span style="color: #a31515">&quot;ID&quot;</span>}, <span style="color: #0000ff">null</span>, <span style="color: #0000ff">null</span>)).
            .ExecuteObject(region);
    }
}
</pre></div>

В результате данного вызова объекту *region* в соответствующее поле будет задано значение *ID* только что вставленной записи (считаем, что поле *ID* в таблице *Region* – автоинкрементное).

Для передачи параметров сохранённой процедуре можно воспользоваться ещё одним способом, не требующим явного указания имён параметров:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%">DataSet SelectByName(<span style="color: #2b91af">string</span> firstName, <span style="color: #2b91af">string</span> lastName)
{
    <span style="color: #0000ff">using</span> (DbManager db = <span style="color: #0000ff">new</span> DbManager())
    {
        <span style="color: #0000ff">return</span> db
            .SetSpCommand(<span style="color: #a31515">&quot;Person_SelectListByName&quot;</span>, firstName, lastName)
            .ExecuteDataSet();
    }
}
</pre></div>

В данном случае важен лишь порядок следования аргументов процедуры. Данная функция самостоятельно строит список параметров исходя из списка параметров сохранённой процедуры.

Для анализа возвращаемого значения и выходных параметров можно воспользоваться следующим методом:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> IDbDataParameter Parameter(
    <span style="color: #2b91af">string</span> parameterName
);
</pre></div>

Например, в приведённом выше примере возвращаемое значение сохранённой процедуры можно (ну тут я слукавил – Person_SelectByName не возвращает такого значения, но если бы возвращала, то было бы можно) проверить следующим образом:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%">DataSet SelectByName(<span style="color: #2b91af">string</span> firstName, <span style="color: #2b91af">string</span> lastName)
{
    <span style="color: #0000ff">using</span> (DbManager db = <span style="color: #0000ff">new</span> DbManager())
    {
        DataSet dataSet = db
            .SetSpCommand(<span style="color: #a31515">&quot;Person_SelectListByName&quot;</span>, firstName, lastName)
            .ExecuteDataSet();

        <span style="color: #2b91af">int</span> returnValue = (<span style="color: #2b91af">int</span>)db.Parameter(<span style="color: #a31515">&quot;@RETURN_VALUE&quot;</span>).Value;

        <span style="color: #0000ff">if</span> (returnValue != 0)
        {
            <span style="color: #0000ff">throw</span> <span style="color: #0000ff">new</span> Exception(
                <span style="color: #2b91af">string</span>.Format(<span style="color: #a31515">&quot;Return value is &#39;{0}&#39;&quot;</span>, returnValue));
        }

        <span style="color: #0000ff">return</span> dataSet;
    }
}
</pre></div>

Последней возможностью работы с параметрами, которую нам осталось рассмотреть, является использование функции подготовки запроса Prepare, которая может быть полезной при выполнении одного и того же запроса несколько раз. Фактически в данном случае вызов метода Execute… разбивается на две части: первая - вызов Prepare с заданием типа, текста и параметров запроса, вторая - вызов соответствующего метода Execute… для выполнения запроса определённое число раз. Следующий пример демонстрирует данную возможность.

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">void</span> InsertRegionList(Region[] regionList)
{
    <span style="color: #0000ff">using</span> (DbManager db = <span style="color: #0000ff">new</span> DbManager())
    {
        db
            .SetCommand (<span style="color: #a31515">@&quot;</span>
<span style="color: #a31515">                INSERT INTO Region (</span>
<span style="color: #a31515">                    RegionID,</span>
<span style="color: #a31515">                    RegionDescription</span>
<span style="color: #a31515">                ) VALUES (</span>
<span style="color: #a31515">                    @ID,</span>
<span style="color: #a31515">                    @Description</span>
<span style="color: #a31515">                )&quot;</span>,
                db.Parameter(<span style="color: #a31515">&quot;@ID&quot;</span>,          regionList[0].ID),
                db.Parameter(<span style="color: #a31515">&quot;@Description&quot;</span>, regionList[0].Description))
            .Prepare();

        <span style="color: #0000ff">foreach</span> (Region r <span style="color: #0000ff">in</span> regionList)
        {
            db.Parameter(<span style="color: #a31515">&quot;@ID&quot;</span>).Value          = r.ID;
            db.Parameter(<span style="color: #a31515">&quot;@Description&quot;</span>).Value = r.Description;

            db.ExecuteNonQuery();
        }
    }
}
</pre></div>

Либо мы можем упростить его следующим образом для бизнес объектов...

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">void</span> InsertRegionList(Region[] regionList)
{
    <span style="color: #0000ff">using</span> (DbManager db = <span style="color: #0000ff">new</span> DbManager())
    {
        db
            .SetCommand(<span style="color: #a31515">@&quot;</span>
<span style="color: #a31515">                INSERT INTO Region (</span>
<span style="color: #a31515">                    RegionID,</span>
<span style="color: #a31515">                    RegionDescription</span>
<span style="color: #a31515">                ) VALUES (</span>
<span style="color: #a31515">                    @ID,</span>
<span style="color: #a31515">                    @Description</span>
<span style="color: #a31515">                )&quot;</span>,
                db.CreateParameters(regionList[0]))
            .Prepare();

        <span style="color: #0000ff">foreach</span> (Region r <span style="color: #0000ff">in</span> regionList)
        {
            db.AssignParameterValues(r);
            db.ExecuteNonQuery();
        }
    }
}
</pre></div>

и класса DataRow

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">static</span> <span style="color: #0000ff">void</span> InsertRegionTable(DataTable dataTable)
{
    <span style="color: #0000ff">using</span> (DbManager db = <span style="color: #0000ff">new</span> DbManager())
    {
        db
            .SetCommand(<span style="color: #a31515">@&quot;</span>
<span style="color: #a31515">                INSERT INTO Region (</span>
<span style="color: #a31515">                    RegionID,</span>
<span style="color: #a31515">                    RegionDescription</span>
<span style="color: #a31515">                ) VALUES (</span>
<span style="color: #a31515">                    @ID,</span>
<span style="color: #a31515">                    @Description</span>
<span style="color: #a31515">                )&quot;</span>,
                db.CreateParameters(dataTable.Rows[0]))
            .Prepare();

        <span style="color: #0000ff">foreach</span> (DataRow dr <span style="color: #0000ff">in</span> dataTable.Rows)
            db.AssignParameterValues(dr).ExecuteNonQuery();
    }
}
</pre></div>

Конечно, для совсем ленивых есть вот такой вариант (метод ExecuteForEach использует именно описанный выше механизм):

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">void</span> InsertRegionList(Region[] regionList)
{
    <span style="color: #0000ff">using</span> (DbManager db = <span style="color: #0000ff">new</span> DbManager())
    {
        db
            .SetCommand(<span style="color: #a31515">@&quot;</span>
<span style="color: #a31515">                INSERT INTO Region (</span>
<span style="color: #a31515">                    RegionID,</span>
<span style="color: #a31515">                    RegionDescription</span>
<span style="color: #a31515">                ) VALUES (</span>
<span style="color: #a31515">                    @ID,</span>
<span style="color: #a31515">                    @Description</span>
<span style="color: #a31515">                )&quot;</span>)
            .ExecuteForEach(regionList);
    }
}
</pre></div>



<span id="executemethods"></span>
### Методы Execute

Класс *DbManager* содержит целый набор семейств методов Execute. Каждое семейство отличается типом возвращаемой сущности, это может быть как *DataSet*, бизнес объект, коллекция бизнес объектов и так далее. Ниже мы рассмотрим все семейства Execute.


<span id="executedataset"></span>
#### ExecuteDataSet

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> DataSet ExecuteDataSet();

<span style="color: #0000ff">public</span> DataSet ExecuteDataSet(DataSet dataSet);

<span style="color: #0000ff">public</span> DataSet ExecuteDataSet(NameOrIndexParameter table);

<span style="color: #0000ff">public</span> DataSet ExecuteDataSet(
    DataSet              dataSet,
    NameOrIndexParameter table);

<span style="color: #0000ff">public</span> DataSet ExecuteDataSet(
    DataSet              dataSet,
    <span style="color: #2b91af">int</span>                  startRecord,
    <span style="color: #2b91af">int</span>                  maxRecords,
    NameOrIndexParameter table);
</pre></div>

Как видно из названия метода результатом данного выражения является объект класса *DataSet* (подобное семантическое правило сохраняется для всех семейств).

Рассмотрим подробнее параметры методов:

- *dataSet* – результирующий датасет (он будет заполнен и возвращен). Если *null*, то будет создан новый экземпляр датасета. Подобный подход импользуется так же в других методах семейств Execute.
- *table* – имя или номер таблицы для заполнения в результирующем датасете. Отдельный интерес представляет класс *NameOrIndexParameter*, для ознакомления с технологией работы лучше прочитать статью: [Унифицированная система передачи строковых/числовых параметров](http://rsdn.ru/forum/prj.rfd/1940433.1). 
- *startRecord* – номер записи с которой начинать заполнение (считается с нуля).
- *maxRecords* – максимальное число записей для заполнения.

Отдельно отмечу, что библиотека писалась как «самодокументируемая», поэтому в большинстве случаев используются схожие приемы и сохраняются имена для параметров с одинаковым смыслом. Поэтому ниже мы не будем рассматривать повторно то, что уже было рассмотрено ранее.


<span id="executedatatable"></span>
#### ExecuteDataTable и ExecuteDataTables

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> DataTable ExecuteDataTable();

<span style="color: #0000ff">public</span> DataTable ExecuteDataTable(DataTable dataTable);

<span style="color: #0000ff">public</span> <span style="color: #0000ff">void</span> ExecuteDataTables(
    <span style="color: #2b91af">int</span>    startRecord,
    <span style="color: #2b91af">int</span>    maxRecords,
    <span style="color: #0000ff">params</span> DataTable[] tableList);

<span style="color: #0000ff">public</span> <span style="color: #0000ff">void</span> ExecuteDataTables(<span style="color: #0000ff">params</span> DataTable[] tableList);
</pre></div>

Как видим, в данном семействе есть два вида методов: *ExecuteDataTable* – заполняет одну таблицу, *ExecuteDataTables* – заполняет массив таблиц, заданный параметром *tableList*.


<span id="executereader"></span>
#### ExecuteReader

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> IDataReader ExecuteReader();

<span style="color: #0000ff">public</span> IDataReader ExecuteReader(CommandBehavior commandBehavior)
</pre></div>

Возвращает экземпляр *IDataReader*.

C *commandBehavior* подробней можно ознакомиться в [MSDN](http://msdn.microsoft.com/en-us/library/system.data.commandbehavior(VS.85).aspx).


<span id="executenonquery"></span>
#### ExecuteNonQuery

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> <span style="color: #2b91af">int</span> ExecuteNonQuery();

<span style="color: #0000ff">public</span> <span style="color: #2b91af">int</span> ExecuteNonQuery(
    <span style="color: #2b91af">string</span> returnValueMember,
    <span style="color: #2b91af">object</span> obj);

<span style="color: #0000ff">public</span> <span style="color: #2b91af">int</span> ExecuteNonQuery(<span style="color: #2b91af">object</span> obj);

<span style="color: #0000ff">public</span> <span style="color: #2b91af">int</span> ExecuteNonQuery(
    <span style="color: #2b91af">string</span>          returnValueMember,
    <span style="color: #0000ff">params</span> <span style="color: #2b91af">object</span>[] objects);

<span style="color: #0000ff">public</span> <span style="color: #2b91af">int</span> ExecuteNonQuery(<span style="color: #0000ff">params</span> <span style="color: #2b91af">object</span>[] objects);
</pre></div>

Данное семейство используется для выполнения *UPDATE*, *INSERT* и *DELETE* запросов. Все методы возвращают число записей, обработанных запросом.

Рассмотрим подробнее параметры методов:

- *returnValueMember* – грубо говоря, это имя поля в объекте, в которое необходимо записать возвращаемое значение. Если же быть точным, то это имя маппера поля (MemberMapper) в которое следует записать возвращаемое значение. Подробнее о мапинге (отображении) мы поговорим ниже.
- *obj* – объект в который будут отображены (записаны) параметры команды.
- *objects* – коллекция объектов в которые будут отображены (записаны) параметры команды.

Здесь мы впервые столкнулись с проявлениями мапинга (отображения). Ранее мы говорили, что BLT как раз занимается в первую очередь отображением данных из БД на объекте в коде, пришло время рассмотреть первый пример этого отображения.

Первые участники действа это хранимые процедуры:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #008000">-- OutRefTest</span>
<span style="color: #0000ff">CREATE</span> <span style="color: #0000ff">Procedure</span> OutRefTest
	@ID             int,
	@outputID       int <span style="color: #0000ff">output</span>,
	@inputOutputID  int <span style="color: #0000ff">output</span>,
	@str            varchar(50),
	@outputStr      varchar(50) <span style="color: #0000ff">output</span>,
	@inputOutputStr varchar(50) <span style="color: #0000ff">output</span>
<span style="color: #0000ff">AS</span>

<span style="color: #0000ff">SET</span> @outputID       = @ID
<span style="color: #0000ff">SET</span> @inputOutputID  = @ID + @inputOutputID
<span style="color: #0000ff">SET</span> @outputStr      = @str
<span style="color: #0000ff">SET</span> @inputOutputStr = @str + @inputOutputStr

<span style="color: #008000">-- Scalar_ReturnParameter</span>
<span style="color: #0000ff">CREATE</span> <span style="color: #0000ff">Function</span> Scalar_ReturnParameter()
<span style="color: #0000ff">RETURNS</span> int
<span style="color: #0000ff">AS</span>
<span style="color: #0000ff">BEGIN</span>
	<span style="color: #0000ff">RETURN</span> 12345
<span style="color: #0000ff">END</span>
</pre></div>

И собственно тесты:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">ReturnParameter</span>
{
	<span style="color: #0000ff">public</span> <span style="color: #2b91af">int</span> Value;
}

[Test]
<span style="color: #0000ff">public</span> <span style="color: #0000ff">void</span> MapReturnValue()
{
	ReturnParameter e = <span style="color: #0000ff">new</span> ReturnParameter();
	<span style="color: #0000ff">using</span> (DbManager db = <span style="color: #0000ff">new</span> DbManager())
	{
		db
			.SetSpCommand(<span style="color: #a31515">&quot;Scalar_ReturnParameter&quot;</span>)
			.ExecuteNonQuery(<span style="color: #a31515">&quot;Value&quot;</span>, e);
	}

	Assert.AreEqual(12345, e.Value);
}
</pre></div>

Как видим в данном тесте BLT успешно отображает возвращаемое функцией *Scalar_ReturnParameter* значение на поле *Value* объекта класса *ReturnParameter*.

Рассмотрим еще два теста:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">OutRefTest</span>
{
	<span style="color: #0000ff">public</span> <span style="color: #2b91af">int</span>    ID             = 5;
	<span style="color: #0000ff">public</span> <span style="color: #2b91af">int</span>    outputID;
	<span style="color: #0000ff">public</span> <span style="color: #2b91af">int</span>    inputOutputID  = 10;
	<span style="color: #0000ff">public</span> <span style="color: #2b91af">string</span> str            = <span style="color: #a31515">&quot;5&quot;</span>;
	<span style="color: #0000ff">public</span> <span style="color: #2b91af">string</span> outputStr;
	<span style="color: #0000ff">public</span> <span style="color: #2b91af">string</span> inputOutputStr = <span style="color: #a31515">&quot;10&quot;</span>;
}

[Test]
<span style="color: #0000ff">public</span> <span style="color: #0000ff">void</span> MapOutput()
{
	OutRefTest o = <span style="color: #0000ff">new</span> OutRefTest();

	<span style="color: #0000ff">using</span> (DbManager db = <span style="color: #0000ff">new</span> DbManager())
	{
		db
			.SetSpCommand(<span style="color: #a31515">&quot;OutRefTest&quot;</span>, db.CreateParameters(o,
				<span style="color: #0000ff">new</span> <span style="color: #2b91af">string</span>[] {      <span style="color: #a31515">&quot;outputID&quot;</span>,      Str<span style="color: #a31515">&quot; },</span>
				<span style="color: #0000ff">new</span> <span style="color: #2b91af">string</span>[] { <span style="color: #a31515">&quot;inputOutputID&quot;</span>, utputStr<span style="color: #a31515">&quot; },</span>
				<span style="color: #0000ff">null</span>))
			.ExecuteNonQuery(o);
	}

	Assert.AreEqual(5,     o.outputID);
	Assert.AreEqual(15,    o.inputOutputID);
	Assert.AreEqual(<span style="color: #a31515">&quot;5&quot;</span>,   o.outputStr);
	Assert.AreEqual(<span style="color: #a31515">&quot;510&quot;</span>, o.inputOutputStr);
}

[Test]
<span style="color: #0000ff">public</span> <span style="color: #0000ff">void</span> MapDataRow()
{
	DataTable dataTable = <span style="color: #0000ff">new</span> DataTable();
	dataTable.Columns.Add(<span style="color: #a31515">&quot;ID&quot;</span>,             <span style="color: #0000ff">typeof</span>(<span style="color: #2b91af">int</span>));
	dataTable.Columns.Add(<span style="color: #a31515">&quot;outputID&quot;</span>,       <span style="color: #0000ff">typeof</span>(<span style="color: #2b91af">int</span>));
	dataTable.Columns.Add(<span style="color: #a31515">&quot;inputOutputID&quot;</span>,  <span style="color: #0000ff">typeof</span>(<span style="color: #2b91af">int</span>));
	dataTable.Columns.Add(<span style="color: #a31515">&quot;str&quot;</span>,            <span style="color: #0000ff">typeof</span>(<span style="color: #2b91af">string</span>));
	dataTable.Columns.Add(<span style="color: #a31515">&quot;outputStr&quot;</span>,      <span style="color: #0000ff">typeof</span>(<span style="color: #2b91af">string</span>));
	dataTable.Columns.Add(<span style="color: #a31515">&quot;inputOutputStr&quot;</span>, <span style="color: #0000ff">typeof</span>(<span style="color: #2b91af">string</span>));

	DataRow dataRow = dataTable.Rows.Add(<span style="color: #0000ff">new</span> <span style="color: #2b91af">object</span>[]{5, 0, 10, <span style="color: #a31515">&quot;5&quot;</span>, 10<span style="color: #a31515">&quot;});</span>

	<span style="color: #0000ff">using</span> (DbManager db = <span style="color: #0000ff">new</span> DbManager())
	{
		db
			.SetSpCommand(<span style="color: #a31515">&quot;OutRefTest&quot;</span>, teParameters(dataRow,
				<span style="color: #0000ff">new</span> <span style="color: #2b91af">string</span>[] {      <span style="color: #a31515">&quot;outputID&quot;</span>,      Str<span style="color: #a31515">&quot; },</span>
				<span style="color: #0000ff">new</span> <span style="color: #2b91af">string</span>[] { <span style="color: #a31515">&quot;inputOutputID&quot;</span>, utputStr<span style="color: #a31515">&quot; },</span>
				<span style="color: #0000ff">null</span>))
			.ExecuteNonQuery(dataRow);
	}

	Assert.AreEqual(5,     dataRow[<span style="color: #a31515">&quot;outputID&quot;</span>]);
	Assert.AreEqual(15,    dataRow[<span style="color: #a31515">&quot;inputOutputID&quot;</span>]);
	Assert.AreEqual(<span style="color: #a31515">&quot;5&quot;</span>,   dataRow[<span style="color: #a31515">&quot;outputStr&quot;</span>]);
	Assert.AreEqual(<span style="color: #a31515">&quot;510&quot;</span>, dataRow[<span style="color: #a31515">&quot;inputOutputStr&quot;</span>]);
}
</pre></div>

Здесь я специально рассмотрел два примера, хотя, по сути, демонстрируют они **одинаковое** использование *ExecuteNonQuery*. Разница заключается в том, что в первом тесте отображение происходит на бизнес объект класса *OutRefTest* а во втором на объект класса *DataRow*. BLT с успехом справляется с задачей отображения «ужа на ежа» а при необходимости и «ужа на слона».

Отличием этих двух тестов от предыдущего, является то, что мы не сообщили **явно** куда и что отображать. Это не есть проявление телепатии, это есть проявление здравого смысла – при мапинге система ориентируется в частности на имена полей и параметров. В рассмотренном примере имена полей класса *OutRefTest* и ячеек объекта *dataRow* совпадают с именами параметров хранимой процедуры *OutRefTest*, именно по этим признакам система поняла, что и куда раскладывать.


<span id="executescalar"></span>
#### ExecuteScalar

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> <span style="color: #2b91af">object</span> ExecuteScalar();

<span style="color: #0000ff">public</span> <span style="color: #2b91af">object</span> ExecuteScalar(ScalarSourceType sourceType);

<span style="color: #0000ff">public</span> <span style="color: #2b91af">object</span> ExecuteScalar(
    ScalarSourceType     sourceType, 
    NameOrIndexParameter nameOrIndex);

<span style="color: #0000ff">public</span> T ExecuteScalar&lt;T&gt;();

<span style="color: #0000ff">public</span> T ExecuteScalar&lt;T&gt;(ScalarSourceType sourceType);

<span style="color: #0000ff">public</span> T ExecuteScalar&lt;T&gt;(
    ScalarSourceType     sourceType, 
    NameOrIndexParameter nameOrIndex);
</pre></div>

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

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> IList ExecuteScalarList(
    IList                list,
    Type                 type,
    NameOrIndexParameter nameOrIndex);

<span style="color: #0000ff">public</span> IList ExecuteScalarList(
    IList list, 
    Type  type);

<span style="color: #0000ff">public</span> ArrayList ExecuteScalarList(
    Type                 type, 
    NameOrIndexParameter nameOrIndex);

<span style="color: #0000ff">public</span> ArrayList ExecuteScalarList(Type type);

<span style="color: #0000ff">public</span> List&lt;T&gt; ExecuteScalarList&lt;T&gt;();

<span style="color: #0000ff">public</span> List&lt;T&gt; ExecuteScalarList&lt;T&gt;(NameOrIndexParameter nameOrIndex);

<span style="color: #0000ff">public</span> IList&lt;T&gt; ExecuteScalarList&lt;T&gt;(
    IList&lt;T&gt;             list,
    NameOrIndexParameter nameOrIndex);

<span style="color: #0000ff">public</span> IList&lt;T&gt; ExecuteScalarList&lt;T&gt;(IList&lt;T&gt; list);
</pre></div>

Семейство предназначено для вычитки списка скалярных величин. Практически все параметры идентичны по семантике параметрам семейства ExecuteScalar. Параметр *type* задает требуемый тип вычитываемой скалярной величины.


<span id="executescalardictionary"></span>
#### ExecuteScalarDictionary

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> IDictionary ExecuteScalarDictionary(
	IDictionary dic,
	NameOrIndexParameter keyField,   Type keyFieldType,
	NameOrIndexParameter valueField, Type valueFieldType);

<span style="color: #0000ff">public</span> Hashtable ExecuteScalarDictionary(
	NameOrIndexParameter keyField,   Type keyFieldType,
	NameOrIndexParameter valueField, Type valueFieldType);

<span style="color: #0000ff">public</span> IDictionary&lt;K,T&gt; ExecuteScalarDictionary&lt;K,T&gt;(
	IDictionary&lt;K,T&gt;     dic,
	NameOrIndexParameter keyField,
	NameOrIndexParameter valueField);

<span style="color: #0000ff">public</span> Dictionary&lt;K,T&gt; ExecuteScalarDictionary&lt;K,T&gt;(
	NameOrIndexParameter keyField,
	NameOrIndexParameter valueField);
</pre></div>

Одной из приятностей BLToolkit является возможность возвращать не просто списки, а словари. Итак, данное семейство возвращает словари скалярных величин из кортежа. Параметры по семантике аналогичны семейству ExecuteScalar, прификс *key* – для ключа, прификс *value* для значения.

Но, на этом еще не все. У данного семейства есть еще подсемейство:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> IDictionary ExecuteScalarDictionary(
	IDictionary          dic,
	MapIndex             index,
	NameOrIndexParameter valueField,
	Type                 valueFieldType);

<span style="color: #0000ff">public</span> Hashtable ExecuteScalarDictionary(
	MapIndex             index, 
	NameOrIndexParameter valueField, 
	Type                 valueFieldType);

<span style="color: #0000ff">public</span> IDictionary&lt;CompoundValue,T&gt; ExecuteScalarDictionary&lt;T&gt;(
	IDictionary&lt;CompoundValue, T&gt; dic, 
	MapIndex                      index, 
	NameOrIndexParameter          valueField);

<span style="color: #0000ff">public</span> Dictionary&lt;CompoundValue,T&gt; ExecuteScalarDictionary&lt;T&gt;(
	MapIndex             index, 
	NameOrIndexParameter valueField)
</pre></div>

Отличается оно тем, что вместо параметров с префиксом *key* используется параметр *index*. Параметр *index* позволяет строить индекс не по одному ключевому полю, а по их совокупности. Таким образом, ключом в результирующем словаре будет экземпляр класса *CompaundValue*, представляющий сложный ключ как единый объект. Мы обязательно рассмотрим пример использования «индексированных» словарей, но ниже.


<span id="executeobject"></span>
#### ExecuteObject

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> <span style="color: #2b91af">object</span> ExecuteObject(<span style="color: #2b91af">object</span> entity);

<span style="color: #0000ff">public</span> <span style="color: #2b91af">object</span> ExecuteObject(<span style="color: #2b91af">object</span> entity, <span style="color: #0000ff">params</span> <span style="color: #2b91af">object</span>[] parameters);

<span style="color: #0000ff">public</span> <span style="color: #2b91af">object</span> ExecuteObject(Type type);

<span style="color: #0000ff">public</span> <span style="color: #2b91af">object</span> ExecuteObject(Type type, <span style="color: #0000ff">params</span> <span style="color: #2b91af">object</span>[] parameters);

<span style="color: #0000ff">public</span> T ExecuteObject&lt;T&gt;();

<span style="color: #0000ff">public</span> T ExecuteObject&lt;T&gt;(<span style="color: #0000ff">params</span> <span style="color: #2b91af">object</span>[] parameters);
</pre></div>

Пожалуй, одно из самых интересных семейств. Предназначено для чтения одной записи возвращаемого кортежа в бизнес объект.

Рассмотрим параметры функций:

- *entity* – объект, куда будет осуществлено чтение.
- *type* – задает требуемый тип возвращаемого объекта.
- *parameters* – дополнительные параметры, которые будут переданы в конструктор. Здесь стоит отметить, что переданы они будут как соответствующее свойство объекта *InitContext*, класс бизнес объекта, в свою очередь, должен иметь конструктор вида *MyObject(InitContext context)*.

Рассмотрим пример:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%">[Test]
<span style="color: #0000ff">public</span> <span style="color: #0000ff">void</span> ExecuteObject()
{
	<span style="color: #0000ff">using</span> (DbManager db = <span style="color: #0000ff">new</span> DbManager())
	{
		Person p = (Person)db
			.SetCommand(<span style="color: #a31515">&quot;SELECT * FROM Person WHERE PersonID = @id&quot;</span>,
			db.Parameter(<span style="color: #a31515">&quot;id&quot;</span>, 1))
			.ExecuteObject(<span style="color: #0000ff">typeof</span>(Person));

		TypeAccessor.WriteConsole(p);
		Assert.AreEqual(1,           p.ID);
		Assert.AreEqual(<span style="color: #a31515">&quot;John&quot;</span>,      p.FirstName);
		Assert.AreEqual(<span style="color: #a31515">&quot;Pupkin&quot;</span>,    p.LastName);
		Assert.AreEqual(Gender.Male, p.Gender);
	}
}
</pre></div>

Кортеж будет иметь следующий вид:

<table border="1" cellspacing="0" cellpadding="2">
	<thead>
		<tr>
			<th>PersonId</th>
			<th>FirstName</th>
			<th>LastName</th>
			<th>MiddleName</th>
			<th>Gender</th>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td>1</td>
			<td>John</td>
			<td>Pupkin</td>
			<td><i>NULL</i></td>
			<td>M</td>
		</tr>
	</tbody>
</table>

Итак, система отобразила выбранную запись на бизнес объект типа Person. Подробней стоит остановиться на полях Person.ID и Person.Gender. Отметим пару интересных моментов:

- В исходном кортеже нет поля ID, а в классе Person поля PersonId. Эта проблема была решена атрибутом MapField(“PersonId”), установленным на поле Person.ID. Так мы сообщили системе, что при мапинге у данного поля будет псевдоним отличный от «родового имени».
- В исходном кортеже поле Gender имеет символьный тип, Person.Gender – является перечислением. Здесь нас выручил атрибут MapValue(“M”) – им мы указали системе, что при отображении данное значение является эквивалентным “M”.


<span id="executelist"></span>
#### ExecuteList

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> ArrayList ExecuteList(Type type);

<span style="color: #0000ff">public</span> ArrayList ExecuteList(Type type, <span style="color: #0000ff">params</span> <span style="color: #2b91af">object</span>[] parameters);

<span style="color: #0000ff">public</span> IList ExecuteList(IList list, Type type);

<span style="color: #0000ff">public</span> IList ExecuteList(
    IList           list, 
    Type            type, 
    <span style="color: #0000ff">params</span> <span style="color: #2b91af">object</span>[] parameters);

<span style="color: #0000ff">public</span> List&lt;T&gt; ExecuteList&lt;T&gt;();

<span style="color: #0000ff">public</span> List&lt;T&gt; ExecuteList&lt;T&gt;(<span style="color: #0000ff">params</span> <span style="color: #2b91af">object</span>[] parameters);

<span style="color: #0000ff">public</span> IList&lt;T&gt; ExecuteList&lt;T&gt;(IList&lt;T&gt; list);

<span style="color: #0000ff">public</span> IList&lt;T&gt; ExecuteList&lt;T&gt;(IList&lt;T&gt; list, <span style="color: #0000ff">params</span> <span style="color: #2b91af">object</span>[] parameters);

<span style="color: #0000ff">public</span> L ExecuteList&lt;L,T&gt;(L list, <span style="color: #0000ff">params</span> <span style="color: #2b91af">object</span>[] parameters)
    <span style="color: #0000ff">where</span> L : IList&lt;T&gt;;

<span style="color: #0000ff">public</span> L ExecuteList&lt;L,T&gt;(<span style="color: #0000ff">params</span> <span style="color: #2b91af">object</span>[] parameters)
    <span style="color: #0000ff">where</span> L : IList&lt;T&gt;, <span style="color: #0000ff">new</span>();
</pre></div>

Данное семейство предназначено для чтения списка объектов из выбранного кортежа. Параметры аналогичны семейству ExecuteObject, поэтому на них мы останавливаться не будем.

<blockquote style="background-color: #FFE4E4; color: #FF5555;">
Не используйте данное семейство для вычитки списка скалярных величин, для этого существует семейство <b>ExecuteScalarList</b>.
</blockquote>

Рассмотрим небольшой пример использования данного семейства:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%">[Test]
<span style="color: #0000ff">public</span> <span style="color: #0000ff">void</span> ExecuteList1()
{
	<span style="color: #0000ff">using</span> (DbManager db = <span style="color: #0000ff">new</span> DbManager())
	{
		ArrayList list = db
			.SetCommand(<span style="color: #a31515">&quot;SELECT * FROM Person&quot;</span>)
			.ExecuteList(<span style="color: #0000ff">typeof</span>(Person));
		
		Assert.IsNotEmpty(list);
	}
}
</pre></div>


<span id="executedictionary"></span>
#### ExecuteDictionary

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> Hashtable ExecuteDictionary(
	NameOrIndexParameter keyField,
	Type                 keyFieldType,
	<span style="color: #0000ff">params</span> <span style="color: #2b91af">object</span>[]      parameters);

<span style="color: #0000ff">public</span> IDictionary ExecuteDictionary(
	IDictionary          dictionary,
	NameOrIndexParameter keyField,
	Type                 type,
	<span style="color: #0000ff">params</span> <span style="color: #2b91af">object</span>[]      parameters);

<span style="color: #0000ff">public</span> Dictionary&lt;TKey, TValue&gt; ExecuteDictionary&lt;TKey, TValue&gt;(
	NameOrIndexParameter keyField,
	<span style="color: #0000ff">params</span> <span style="color: #2b91af">object</span>[]      parameters);

<span style="color: #0000ff">public</span> IDictionary&lt;TKey, TValue&gt; ExecuteDictionary&lt;TKey, TValue&gt;(
	IDictionary&lt;TKey, TValue&gt; dictionary,
	NameOrIndexParameter      keyField,
	<span style="color: #0000ff">params</span> <span style="color: #2b91af">object</span>[]           parameters);

<span style="color: #0000ff">public</span> IDictionary&lt;TKey, TValue&gt; ExecuteDictionary&lt;TKey, TValue&gt;(
	IDictionary&lt;TKey, TValue&gt; dictionary,
	NameOrIndexParameter      keyField,
	Type                      destObjectType,
	<span style="color: #0000ff">params</span> <span style="color: #2b91af">object</span>[]           parameters)
</pre></div>

Позволяет вычитывать словарь бизнес объектов из кортежа. Семантика параметров аналогична ExecuteScalarDictionary и ExecuteObject. Параметру type и destObjectType – задают требуемый тип бизнес объекта.

<blockquote style="background-color: #FFE4E4; color: #FF5555;">
Не используйте данное семейство для вычитки словарей скалярных величин, для этого есть семейство <b>ExecuteScalarDictionary</b>.
</blockquote>

Как и в случае с ExecuteScalarDictionary ExecuteDictionary имеет «индексированное» подсемейство:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> Hashtable ExecuteDictionary(
	MapIndex        index,
	Type            type,
	<span style="color: #0000ff">params</span> <span style="color: #2b91af">object</span>[] parameters);

<span style="color: #0000ff">public</span> IDictionary ExecuteDictionary(
	IDictionary     dictionary,
	MapIndex        index,
	Type            type,
	<span style="color: #0000ff">params</span> <span style="color: #2b91af">object</span>[] parameters);

<span style="color: #0000ff">public</span> Dictionary&lt;CompoundValue, TValue&gt; ExecuteDictionary&lt;TValue&gt;(
	MapIndex        index,
	<span style="color: #0000ff">params</span> <span style="color: #2b91af">object</span>[] parameters);

<span style="color: #0000ff">public</span> IDictionary&lt;CompoundValue, TValue&gt; ExecuteDictionary&lt;TValue&gt;(
	IDictionary&lt;CompoundValue, TValue&gt; dictionary,
	MapIndex                           index,
	<span style="color: #0000ff">params</span> <span style="color: #2b91af">object</span>[]                    parameters);

<span style="color: #0000ff">public</span> IDictionary&lt;CompoundValue, TValue&gt; ExecuteDictionary&lt;TValue&gt;(
	IDictionary&lt;CompoundValue, TValue&gt; dictionary,
	MapIndex                           index,
	Type                               destObjectType,
	<span style="color: #0000ff">params</span> <span style="color: #2b91af">object</span>[]                    parameters)
</pre></div>

Опять-таки, нам тут все знакомо, поэтому для закрепления понимания сразу перейдем к примеру:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">private</span> <span style="color: #0000ff">const</span> <span style="color: #2b91af">int</span>     _id = 1;

[Test]
<span style="color: #0000ff">public</span> <span style="color: #0000ff">void</span> DictionaryMapIndexTest3()
{
	<span style="color: #0000ff">using</span> (DbManager db = <span style="color: #0000ff">new</span> DbManager())
	{
		Hashtable table = <span style="color: #0000ff">new</span> Hashtable();
		db
			.SetCommand(<span style="color: #a31515">&quot;SELECT * FROM Person&quot;</span>)
			.ExecuteDictionary(table,
			<span style="color: #0000ff">new</span> MapIndex(<span style="color: #a31515">&quot;@PersonID&quot;</span>, 2, 3), <span style="color: #0000ff">typeof</span>(Person));

		Assert.IsNotNull(table);
		Assert.IsTrue(table.Count &gt; 0);

		Person actualValue = (Person)table[<span style="color: #0000ff">new</span> CompoundValue(_id, <span style="color: #a31515">&quot;&quot;</span>, <span style="color: #a31515">&quot;Pupkin&quot;</span>)];
		Assert.IsNotNull(actualValue);
		Assert.AreEqual(<span style="color: #a31515">&quot;John&quot;</span>, actualValue.FirstName);
	}
}
</pre></div>

В примере используется сложный ключ, состоящий из полей PersonId, третьего поля в кортеже (все считается с нуля) – SecondName и четвертого поля в кортеже – MiddleName. Ключом в словаре является объект класса CompaundValue.

Ну и как всегда, если нам не нужны такие изыски (сложные ключи) то можно сделать все гораздо проще:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%">[Test]
<span style="color: #0000ff">public</span> <span style="color: #0000ff">void</span> GenericsDictionaryTest()
{
	<span style="color: #0000ff">using</span> (DbManager db = <span style="color: #0000ff">new</span> DbManager())
	{
		Dictionary&lt;<span style="color: #2b91af">int</span>, Person&gt; dic = db
			.SetCommand(<span style="color: #a31515">&quot;SELECT * FROM Person&quot;</span>)
			.ExecuteDictionary&lt;<span style="color: #2b91af">int</span>, Person&gt;(<span style="color: #a31515">&quot;ID&quot;</span>);

		Assert.IsNotNull(dic);
		Assert.IsTrue(dic.Count &gt; 0);

		Person actualValue = dic[1];
		Assert.IsNotNull(actualValue);
		Assert.AreEqual(<span style="color: #a31515">&quot;John&quot;</span>, actualValue.FirstName);
	}
}
</pre></div>

<blockquote style="background-color: #FFE4E4; color: #FF5555;">
Как видно из примеров, в одном случае используется «@PersonId» а в другом «ID». Разница в следующем: если не указано '@', то значение берётся из поля уже смапленного объекта, если '@' присутствует, то из исходной записи.
</blockquote>

<blockquote style="background-color: #FFE4E4; color: #FF5555;">
Зачем это надо. Первый случай может пригодиться, если словарь строится по полю, которое явно не отображается на исходную запись. Например, какое-нибудь составное поле в объекте. Второй случай может понадобиться, когда нужно построить словарь по полю, которое есть в исходном рекордсете, но не отображается на объект. Если ключевое поле один в один отображается на объект, то разницы нет.
</blockquote>

<blockquote style="background-color: #FFE4E4; color: #FF5555;">
Оригинал by <a href="http://rsdn.ru/Users/1.aspx">Игорь Ткачев</a> – <a href="http://rsdn.ru/forum/message/3017055.1.aspx">здесь</a>.
</blockquote>


<span id="executeforeach"></span>
#### ExecuteForEach

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> <span style="color: #2b91af">int</span> ExecuteForEach(ICollection collection);

<span style="color: #0000ff">public</span> <span style="color: #2b91af">int</span> ExecuteForEach&lt;T&gt;(ICollection&lt;T&gt; collection);

<span style="color: #0000ff">public</span> <span style="color: #2b91af">int</span> ExecuteForEach(DataTable table);

<span style="color: #0000ff">public</span> <span style="color: #2b91af">int</span> ExecuteForEach(DataSet dataSet);

<span style="color: #0000ff">public</span> <span style="color: #2b91af">int</span> ExecuteForEach(DataSet dataSet, NameOrIndexParameter nameOrIndex);
</pre></div>

Ранее я уже приводил пример данного семейства. Но не грех и повторить: данное семейство выполняет SQL выражение для заданного множества. Сначала команда готовит выражение, используя метод *Prepare()* после чего выполняет *ExecuteNonQuery()* для каждого элемента коллекции (из элементов коллекции заполняются значения параметров).

Параметры, подробно описывать не буду, замечу только, что для *dataSet* без *nameOrIndex* выражение будет выполнено для первой таблицы (индекс == 0).


<span id="executeresultsets"></span>
#### ExecuteResultSets

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> MapResultSet[] ExecuteResultSet(<span style="color: #0000ff">params</span> MapResultSet[] resultSets);

<span style="color: #0000ff">public</span> MapResultSet[] ExecuteResultSet(
	Type masterType, 
	<span style="color: #0000ff">params</span> MapNextResult[] nextResults);

<span style="color: #0000ff">public</span> MapResultSet[] ExecuteResultSet&lt;T&gt;(<span style="color: #0000ff">params</span> MapNextResult[] nextResults);
</pre></div>

Это семейство позволяет выполнять комплексное отображение данных на сложную связанную иерархию объектов.

Можно долго рассказывать, как и что, но проще разобрать все на примере. В примере заданы следующие связи (в рамках объектной модели):

- Parent к Child – ко многим 
- Child к Parent – к одному
- Child к Grandchild – ко многим
- Grandchild к Child – к одному.

Таким образом, имеем иерархию из 3 взаимосвязанных классов.

Особое внимание следует обратить на то, что повязка осуществляется по именам для отображения.

Читайте, наслаждайтесь (примечания в комментариях к тексту):

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%">[TestFixture]
<span style="color: #0000ff">public</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">ComplexMapping</span>
{
	<span style="color: #008000">// запрос с 3 связанными таблицами</span>
	<span style="color: #0000ff">const</span> <span style="color: #2b91af">string</span> TestQuery = <span style="color: #a31515">@&quot;</span>
<span style="color: #a31515">		-- Parent Data</span>
<span style="color: #a31515">		SELECT       1 as ParentID</span>
<span style="color: #a31515">		UNION SELECT 2 as ParentID</span>

<span style="color: #a31515">		-- Child Data</span>
<span style="color: #a31515">		SELECT       4 ChildID, 1 as ParentID</span>
<span style="color: #a31515">		UNION SELECT 5 ChildID, 2 as ParentID</span>
<span style="color: #a31515">		UNION SELECT 6 ChildID, 2 as ParentID</span>
<span style="color: #a31515">		UNION SELECT 7 ChildID, 1 as ParentID</span>

<span style="color: #a31515">		-- Grandchild Data</span>
<span style="color: #a31515">		SELECT       1 GrandchildID, 4 as ChildID</span>
<span style="color: #a31515">		UNION SELECT 2 GrandchildID, 4 as ChildID</span>
<span style="color: #a31515">		UNION SELECT 3 GrandchildID, 5 as ChildID</span>
<span style="color: #a31515">		UNION SELECT 4 GrandchildID, 5 as ChildID</span>
<span style="color: #a31515">		UNION SELECT 5 GrandchildID, 6 as ChildID</span>
<span style="color: #a31515">		UNION SELECT 6 GrandchildID, 6 as ChildID</span>
<span style="color: #a31515">		UNION SELECT 7 GrandchildID, 7 as ChildID</span>
<span style="color: #a31515">		UNION SELECT 8 GrandchildID, 7 as ChildID</span>
<span style="color: #a31515">&quot;</span>;
	<span style="color: #008000">// верхний класс</span>
	<span style="color: #0000ff">public</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">Parent</span>
	{
		[MapField(&quot;ParentID&quot;)]
		<span style="color: #0000ff">public</span> <span style="color: #2b91af">int</span> ID;
		<span style="color: #008000">// Список подчиненных объектов</span>
		<span style="color: #0000ff">public</span> List&lt;Child&gt; Children = <span style="color: #0000ff">new</span> List&lt;Child&gt;();
	}
	<span style="color: #008000">// класс связанный с Parent</span>
	[MapField(&quot;ParentID&quot;, &quot;Parent.ID&quot;)]
	<span style="color: #0000ff">public</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">Child</span>
	{
		[MapField(&quot;ChildID&quot;)]
		<span style="color: #0000ff">public</span> <span style="color: #2b91af">int</span> ID;
		<span style="color: #008000">// родительский объект</span>
		<span style="color: #0000ff">public</span> Parent Parent = <span style="color: #0000ff">new</span> Parent();
		<span style="color: #008000">//Список подчиненных объектов</span>
		<span style="color: #0000ff">public</span> List&lt;Grandchild&gt; Grandchildren = <span style="color: #0000ff">new</span> List&lt;Grandchild&gt;();
	}
	<span style="color: #008000">// Класс связи связанный с Child </span>
	[MapField(&quot;ChildID&quot;, &quot;Child.ID&quot;)]
	<span style="color: #0000ff">public</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">Grandchild</span>
	{
		[MapField(&quot;GrandchildID&quot;)]
		<span style="color: #0000ff">public</span> <span style="color: #2b91af">int</span> ID;
		<span style="color: #008000">// родительский объект</span>
		<span style="color: #0000ff">public</span> Child Child = <span style="color: #0000ff">new</span> Child();
	}

	[Test]
	<span style="color: #0000ff">public</span> <span style="color: #0000ff">void</span> Test()
	{
		<span style="color: #008000">// список родительских объектов – «корень» который будет заполнен</span>
		List&lt;Parent&gt;   parents = <span style="color: #0000ff">new</span> List&lt;Parent&gt;();
		<span style="color: #008000">// массив резалтсетов</span>
		<span style="color: #008000">/*[/a]*/</span>MapResultSet<span style="color: #008000">/*[/a]*/</span>[] sets    = <span style="color: #0000ff">new</span> MapResultSet[3];

		<span style="color: #008000">//создадим резалтсет для корневого списка</span>
		<span style="color: #008000">// в качестве параметров переданы тип корневого объекта и </span>
		<span style="color: #008000">// и список объектов, который следует заполнить</span>
		sets[0] = <span style="color: #0000ff">new</span> MapResultSet(<span style="color: #0000ff">typeof</span>(Parent), parents);
		sets[1] = <span style="color: #0000ff">new</span> MapResultSet(<span style="color: #0000ff">typeof</span>(Child));
		sets[2] = <span style="color: #0000ff">new</span> MapResultSet(<span style="color: #0000ff">typeof</span>(Grandchild));

		<span style="color: #008000">// зададим связь резалтсету «Parent» устанавливается подчиненная</span>
		<span style="color: #008000">// связь к резалтсету «Child»</span>
		<span style="color: #008000">// параметры:</span>
		<span style="color: #008000">// имя поля отображения по которому осуществляется связь в подчиненном объекте</span>
		<span style="color: #008000">// имя поля отображения по которому осуществляется связь в родительском объекте</span>
		<span style="color: #008000">// имя поля отображения в родительском объекте для заполнения дочерними</span>
		sets[0].AddRelation(sets[1], <span style="color: #a31515">&quot;ParentID&quot;</span>, <span style="color: #a31515">&quot;ParentID&quot;</span>, <span style="color: #a31515">&quot;Children&quot;</span>);
		<span style="color: #008000">// все практически аналогично, но теперь задается обратная связь</span>
		<span style="color: #008000">// от Child к Parent</span>
		<span style="color: #008000">// таким образом в результате отображения будет заполнено не только</span>
		<span style="color: #008000">// поле Parent.Children но и для каждого Child из Children будет задан Parent</span>
		sets[1].AddRelation(sets[0], <span style="color: #a31515">&quot;ParentID&quot;</span>, <span style="color: #a31515">&quot;ParentID&quot;</span>, <span style="color: #a31515">&quot;Parent&quot;</span>);

		<span style="color: #008000">// Аналогично, но уже от Child к Grandchild и наоборот</span>
		sets[1].AddRelation(sets[2], <span style="color: #a31515">&quot;ChildID&quot;</span>, <span style="color: #a31515">&quot;ChildID&quot;</span>, <span style="color: #a31515">&quot;Grandchildren&quot;</span>);
		sets[2].AddRelation(sets[1], <span style="color: #a31515">&quot;ChildID&quot;</span>, <span style="color: #a31515">&quot;ChildID&quot;</span>, <span style="color: #a31515">&quot;Child&quot;</span>);

		<span style="color: #0000ff">using</span> (DbManager db = <span style="color: #0000ff">new</span> DbManager())
		{
			db
				.SetCommand      (TestQuery)
				.ExecuteResultSet(sets);
		}
		<span style="color: #008000">// здесь проверки правильности заполнения</span>
		Assert.IsNotEmpty(parents);

		<span style="color: #0000ff">foreach</span> (Parent parent <span style="color: #0000ff">in</span> parents)
		{
			Assert.IsNotNull(parent);
			Assert.IsNotEmpty(parent.Children);

			<span style="color: #0000ff">foreach</span> (Child child <span style="color: #0000ff">in</span> parent.Children)
			{
				Assert.AreEqual(parent, child.Parent);
				Assert.IsNotEmpty(child.Grandchildren);

				<span style="color: #0000ff">foreach</span> (Grandchild grandchild <span style="color: #0000ff">in</span> child.Grandchildren)
				{
					Assert.AreEqual(child,  grandchild.Child);
					Assert.AreEqual(parent, grandchild.Child.Parent);
				}
			}
		}
	}
}
</pre></div>

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
</blockquote>

<blockquote style="background-color: #F5F9FF; color: #506580;">
Орлы из оракла не пользуются SqlString, SqlInt32 и т.п. типами, а напридумывали велосипедов. Поэтому приходится прилагать так много усилий, чтобы привести всякие OracleDecimal хотя бы к System.Decimal.
</blockquote>

<blockquote style="background-color: #F5F9FF; color: #506580;">
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

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #008000">// шаблон имени выгляди следующим образом:</span>
<span style="color: #008000">// ConvertToDestinatonType(object value) ;</span>
<span style="color: #008000">// где DestinatonType – тип в который необходимо преобразовать.</span>
<span style="color: #008000">// к Int32</span>
ConvertToInt32(<span style="color: #2b91af">object</span> <span style="color: #0000ff">value</span>);
<span style="color: #008000">// к Int32?</span>
ConvertToNullableInt32(<span style="color: #2b91af">object</span> <span style="color: #0000ff">value</span>);
<span style="color: #008000">// к SqlInt32</span>
ConvertToSqlInt32(<span style="color: #2b91af">object</span> <span style="color: #0000ff">value</span>);
</pre></div>

По умолчанию *MappingSchema* делегирует подобные вызовы к классу *Convert* (*BLToolkit.Common.Convert*). Данный класс можно расценивать как замену стандартному классу *System.Convert*, который можно смело назвать "младшим братом", т.к. *BLToolkit.Common.Convert* значительно превосходит его по возможностям.

Отдельно стоит отметить следующие "высокоуровневые" функции:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> <span style="color: #0000ff">virtual</span> <span style="color: #2b91af">object</span> ConvertChangeType(
	<span style="color: #2b91af">object</span> <span style="color: #0000ff">value</span>, 
	Type   conversionType);

<span style="color: #0000ff">public</span> <span style="color: #0000ff">virtual</span> <span style="color: #2b91af">object</span> ConvertChangeType(
	<span style="color: #2b91af">object</span> <span style="color: #0000ff">value</span>, 
	Type   conversionType, 
	<span style="color: #2b91af">bool</span>   isNullable);

<span style="color: #0000ff">public</span> <span style="color: #0000ff">virtual</span> T ConvertTo&lt;T, P&gt;(P <span style="color: #0000ff">value</span>);
</pre></div>

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

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> <span style="color: #0000ff">void</span> MapSourceToDestination(
	IMapDataSource      source, <span style="color: #2b91af">object</span> sourceObject, 
	IMapDataDestination dest,   <span style="color: #2b91af">object</span> destObject,
	<span style="color: #0000ff">params</span> <span style="color: #2b91af">object</span>[]     parameters);

<span style="color: #0000ff">public</span> <span style="color: #0000ff">void</span> MapSourceToDestination(
	<span style="color: #2b91af">object</span>          sourceObject,
	<span style="color: #2b91af">object</span>          destObject,
	<span style="color: #0000ff">params</span> <span style="color: #2b91af">object</span>[] parameters);

<span style="color: #0000ff">public</span> <span style="color: #0000ff">virtual</span> <span style="color: #0000ff">void</span> MapSourceListToDestinationList(
	IMapDataSourceList      dataSourceList,
	IMapDataDestinationList dataDestinationList,
	<span style="color: #0000ff">params</span> <span style="color: #2b91af">object</span>[]         parameters)
</pre></div>

Последний метод отличается от двух первых – он отображает списки объектов.

И как всегда, подробнее о параметрах:

- source – источник, наследник IMapDataSource – предоставляет методы для извлечения данных из источника.
- dest – получатель, наследник IMapDataDestination – предоставляет методы для записи данных.
- sourceObject – собственно объект, из которого происходит отображение.
- destObject – объект в который происходит отображение.
- parameters – набор параметров передаваемый в конструктор объекта получателя через экземпляр InitContext.
- dataSourceList и dataDestinationList – то же, что и source и dest, только для списков.

Разберем пример отображения DataReader на Person:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> Person MapDataReaderToPerson(IDataReader reader, Person p)
{
	MappingSchema schema     = <span style="color: #0000ff">new</span> MappingSchema();

	IMapDataSource source    = schema.CreateDataReaderMapper(reader);
	IMapDataDestination dest = schema.GetObjectMapper       (p.GetType());

	Schema.MapDataReaderToObject(source, reader, dest, p);
	
	<span style="color: #0000ff">return</span> p;
}
</pre></div>

Вот, примерно так оно и происходит.

Теперь давайте подробней рассмотрим IDataSource и IDataDestination. Данные интерфейсы описывают методы, которые предоставляют возможности чтенья из источника и записи в получателя. 

Вкратце рассмотрим данные интерфейсы:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> <span style="color: #0000ff">interface</span> IMapDataSource
{
	<span style="color: #008000">// общее количество доступных для чтения полей</span>
	<span style="color: #2b91af">int</span>      Count { <span style="color: #0000ff">get</span>; }

	<span style="color: #008000">// тип поля по индексу</span>
	Type     GetFieldType (<span style="color: #2b91af">int</span> index);
	<span style="color: #008000">// имя поля по индексу</span>
	<span style="color: #2b91af">string</span>   GetName      (<span style="color: #2b91af">int</span> index);
	<span style="color: #008000">// получает индекс поля по имени</span>
	<span style="color: #2b91af">int</span>      GetOrdinal   (<span style="color: #2b91af">string</span> name);
	<span style="color: #008000">// получить значение из объекта по заданному индексу</span>
	<span style="color: #2b91af">object</span>   GetValue     (<span style="color: #2b91af">object</span> o, <span style="color: #2b91af">int</span> index);
	<span style="color: #008000">// получить значение из объекта по заданному имени</span>
	<span style="color: #2b91af">object</span>   GetValue     (<span style="color: #2b91af">object</span> o, <span style="color: #2b91af">string</span> name);

	<span style="color: #008000">// поле по заданному индексу IsNull</span>
	<span style="color: #2b91af">bool</span>     IsNull       (<span style="color: #2b91af">object</span> o, <span style="color: #2b91af">int</span> index);

	<span style="color: #008000">// поддерживает типизированные значения для поля по индексу</span>
	<span style="color: #2b91af">bool</span>     SupportsTypedValues(<span style="color: #2b91af">int</span> index);

	<span style="color: #008000">// получить типизированное значение по заданному индексу</span>
	SByte    GetSByte     (<span style="color: #2b91af">object</span> o, <span style="color: #2b91af">int</span> index);
	Int16    GetInt16     (<span style="color: #2b91af">object</span> o, <span style="color: #2b91af">int</span> index);
	<span style="color: #008000">// и так далее</span>
	<span style="color: #008000">// XXX GetXXX (object o, int index);</span>
}

<span style="color: #0000ff">public</span> <span style="color: #0000ff">interface</span> IMapDataDestination
{
	<span style="color: #008000">// тип поля по индексу</span>
	Type GetFieldType (<span style="color: #2b91af">int</span> index);
	<span style="color: #008000">// получает индекс поля по имени</span>
	<span style="color: #2b91af">int</span>  GetOrdinal   (<span style="color: #2b91af">string</span> name);
	<span style="color: #008000">// устанавливает значение value в объекте о по индексу</span>
	<span style="color: #0000ff">void</span> SetValue     (<span style="color: #2b91af">object</span> o, <span style="color: #2b91af">int</span> index,   <span style="color: #2b91af">object</span> <span style="color: #0000ff">value</span>);
	<span style="color: #008000">// устанавливает значение value в объекте о по имени</span>
	<span style="color: #0000ff">void</span> SetValue     (<span style="color: #2b91af">object</span> o, <span style="color: #2b91af">string</span> name, <span style="color: #2b91af">object</span> <span style="color: #0000ff">value</span>);

	<span style="color: #008000">// устанавливает значение null в объекте по индексу</span>
	<span style="color: #0000ff">void</span> SetNull      (<span style="color: #2b91af">object</span> o, <span style="color: #2b91af">int</span> index);

	<span style="color: #008000">// поддерживает типизированные значения для поля по индексу</span>
	<span style="color: #2b91af">bool</span> SupportsTypedValues(<span style="color: #2b91af">int</span> index);

	<span style="color: #008000">// устанавливают типизированное значение value в объекте о по byltrce</span>
	<span style="color: #0000ff">void</span> SetSByte     (<span style="color: #2b91af">object</span> o, <span style="color: #2b91af">int</span> index, SByte    <span style="color: #0000ff">value</span>);
	<span style="color: #0000ff">void</span> SetInt16     (<span style="color: #2b91af">object</span> o, <span style="color: #2b91af">int</span> index, Int16    <span style="color: #0000ff">value</span>);
	<span style="color: #008000">// и так далее</span>
	<span style="color: #008000">// SetXXX(object o, int index, XXX value);</span>
}
</pre></div>

Про поддержку типизированных значений стоит написать отдельно, и не своими словами:

<blockquote style="background-color: #F5F9FF; color: #506580;">
В интерфейсах IMapDataSource и IMapDataDestination есть методы типа 
</blockquote>

<blockquote style="background-color: #F5F9FF; color: #506580;">
Int32    GetInt32     (object o, int index);
</blockquote>

<blockquote style="background-color: #F5F9FF; color: #506580;">
Означающие "возьмите у объекта o поле за номером index и верните его как int".
</blockquote>

<blockquote style="background-color: #F5F9FF; color: #506580;">
Смысл в том, что если у нас есть миллион объектов с двумя полями типа int, то при мапинге через GetValue/SetValue половина работы уходит на boxing/unboxing.
</blockquote>

<blockquote style="background-color: #F5F9FF; color: #506580;">
Т.е. CLR на полном серьёзе выделяет в куче 2 миллиона маленьких объектов, оборачивает в них наши числа и потом как-нибудь эти 2 миллиона объектов высвобождает.
</blockquote>

<blockquote style="background-color: #F5F9FF; color: #506580;">
Получаем на ровном месте фрагментацию памяти и лишние вызовы сборщика мусора.
</blockquote>

<blockquote style="background-color: #F5F9FF; color: #506580;">
При мапинге через TypedValues boxing'а не происходит. GetInt32 вычитывает целое число и сохраняет его в регистр EAX.
</blockquote>

<blockquote style="background-color: #F5F9FF; color: #506580;">
SetInt32 берёт из EAX и выставляет нашему полю это значение. Если поле имеет тип, например, Int64, то код будет более мудрёным:
</blockquote>

<blockquote style="background-color: #F5F9FF; color: #506580;">
destMapper.SetInt64(destObj,dstIndex,Converter.ConvertInt32ToInt64(srcMapper.GetInt32(srcObj, srcIndex));
</blockquote>

<blockquote style="background-color: #F5F9FF; color: #506580;">
Опять-таки никакого выделения/освобождения памяти.
</blockquote>

<blockquote style="background-color: #F5F9FF; color: #506580;">
Так вот, SupportsTypedValues как раз и сообщает маперу, что источник/получатель умеет работать с числами, датами и т.п. без boxing'а.
</blockquote>

<blockquote style="background-color: #F5F9FF; color: #506580;">
Оригинал by <a href="http://rsdn.ru/Users/507.aspx">Павел Блудов</a> – <a href="http://rsdn.ru/forum/message/2999766.1.aspx">здесь</a>.
</blockquote>

Дополню, что SupportsValueTypes работает в паре с GetFieldType, который должен сообщить правильный тип поля.

Для отображения списков (коллекции, таблицы и т.п.) существуют еще два дополнительных интерфейса:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> <span style="color: #0000ff">interface</span> IMapDataSourceList
{
	<span style="color: #0000ff">void</span> InitMapping      (InitContext initContext);
	<span style="color: #2b91af">bool</span> SetNextDataSource(InitContext initContext);
	<span style="color: #0000ff">void</span> EndMapping       (InitContext initContext);
}

<span style="color: #0000ff">public</span> <span style="color: #0000ff">interface</span> IMapDataDestinationList
{
	<span style="color: #0000ff">void</span>                InitMapping       (InitContext initContext);
	IMapDataDestination GetDataDestination(InitContext initContext);
	<span style="color: #2b91af">object</span>              GetNextObject     (InitContext initContext);
	<span style="color: #0000ff">void</span>                EndMapping        (InitContext initContext);
}
</pre></div>

InitMapping и EndMapping – инициализация и окончание отображения. В остальном, все сводится к тому, что при маппинге списков производится поочередное отображение каждого их элемента. 

Рассмотренные интерфейсы позволяют описать некий источник или получатель данных, и, при необходимости, вы легко можете расширить систему отображения необходимыми вам источниками и получателями.


<span id="mappingmapper"></span>
#### MemberMapper

Теперь вы получили представление о том, как работает отображение объектов. Но это еще не все. Кроме представлений объектов, есть еще представления полей. Для этого используются *ValueMapper* и *MemberMapper*. Первый используется для отображения скалярных полей, второй – всех прочих. Механизм *ValueMapper* инкапсулирован в BLT, поэтому рассматривать мы его не будем. А вот *MemberMapper* мы рассмотрим более подробно.

*MemberMapper* позволяет вам… ну тут проще показать. Приведу простой пример: допустим, у некоторого объекта есть свойство со словарем строк. При сохранении данного объекта необходимо так же сохранить и словарь. 

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">SimpleDictionaryMapper</span> : MemberMapper
{
	<span style="color: #0000ff">public</span> <span style="color: #0000ff">override</span> <span style="color: #2b91af">object</span> GetValue(<span style="color: #2b91af">object</span> o)
	{
		Dictionary&lt;<span style="color: #2b91af">string</span>, <span style="color: #2b91af">string</span>&gt; dic = <span style="color: #0000ff">base</span>.GetValue(o) <span style="color: #0000ff">as</span> Dictionary&lt;<span style="color: #2b91af">string</span>, <span style="color: #2b91af">string</span>&gt;;
		
		<span style="color: #0000ff">if</span> (dic == <span style="color: #0000ff">null</span>) <span style="color: #0000ff">return</span> <span style="color: #0000ff">null</span>;

		StringBuilder sb = <span style="color: #0000ff">new</span> StringBuilder();
		<span style="color: #0000ff">foreach</span> (<span style="color: #2b91af">string</span> key <span style="color: #0000ff">in</span> dic.Keys)
			sb.AppendFormat(<span style="color: #a31515">&quot;{0}={1};&quot;</span>, key, dic[key]);

		<span style="color: #0000ff">return</span> sb.ToString();

	}

	<span style="color: #0000ff">public</span> <span style="color: #0000ff">override</span> <span style="color: #0000ff">void</span> SetValue(<span style="color: #2b91af">object</span> o, <span style="color: #2b91af">object</span> <span style="color: #0000ff">value</span>)
	{
		<span style="color: #2b91af">string</span> s = MappingSchema.ConvertToString(<span style="color: #0000ff">value</span>);
		
		<span style="color: #0000ff">if</span> (s == <span style="color: #2b91af">string</span>.Empty) <span style="color: #0000ff">base</span>.SetValue(o, <span style="color: #0000ff">null</span>);
		
		Dictionary&lt;<span style="color: #2b91af">string</span>, <span style="color: #2b91af">string</span>&gt; dic = <span style="color: #0000ff">new</span> Dictionary&lt;<span style="color: #2b91af">string</span>, <span style="color: #2b91af">string</span>&gt;();

		<span style="color: #0000ff">foreach</span> (<span style="color: #2b91af">string</span> pair <span style="color: #0000ff">in</span> s.Split(<span style="color: #a31515">&#39;;&#39;</span>))
		{
			<span style="color: #0000ff">if</span> (pair.Length &lt; 3) <span style="color: #0000ff">continue</span>;

			<span style="color: #2b91af">string</span>[] keyValue = pair.Split(<span style="color: #a31515">&#39;=&#39;</span>);

			<span style="color: #0000ff">if</span> (keyValue.Length != 2) <span style="color: #0000ff">continue</span>;

			dic.Add(keyValue[0], keyValue[1]);
		}

		<span style="color: #0000ff">base</span>.SetValue(o, dic);
	}
}
</pre></div>

Приведенный пример отображает словарь на строку в заданном формате (*GetValue*) при обратном отображении (*SetValue*) данная строка разбирается и из нее заново собирается словарь.

Используется *MemberMapper* следующим образом:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">TestObject</span>
{
	[MemberMapper(typeof(SimpleDictionaryMapper)]
	<span style="color: #0000ff">public</span> Dictionary&lt;<span style="color: #2b91af">string</span>, <span style="color: #2b91af">string</span>&gt; Dictionary;
}
</pre></div>



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

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #008000">// Задает алиас полю Field1.</span>
<span style="color: #008000">// Данный подход можно использовать для «переименования» унаследованных полей.</span>
[MapField(&quot;MapName&quot;, &quot;Field1&quot;)]
<span style="color: #0000ff">public</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">Object1</span>
{
	<span style="color: #0000ff">public</span> <span style="color: #2b91af">int</span> Field1;
	[MapField(&quot;intfld&quot;)]
	<span style="color: #0000ff">public</span> <span style="color: #2b91af">int</span> Field2;
}

[MapValue(true,  &quot;Y&quot;)]
[MapValue(false, &quot;N&quot;)]
<span style="color: #0000ff">public</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">Object2</span>
{
	<span style="color: #0000ff">public</span> <span style="color: #2b91af">bool</span> Field1;
	<span style="color: #0000ff">public</span> <span style="color: #2b91af">int</span>  Field2;
}

<span style="color: #0000ff">public</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">Object3</span>
{
	<span style="color: #0000ff">public</span> Object2 Object2 = <span style="color: #0000ff">new</span> Object2();
	<span style="color: #0000ff">public</span> Object4 Object4;
}

<span style="color: #008000">//При необходимости пожно задать алиасы для полей вложенных объектов.</span>
[MapField(&quot;fld1&quot;, &quot;Object3.Object2.Field1&quot;)]
[MapField(&quot;fld2&quot;, &quot;Object3.Object4.Str1&quot;)]
<span style="color: #0000ff">public</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">Object4</span>
{
	<span style="color: #0000ff">public</span> Object3 Object3 = <span style="color: #0000ff">new</span> Object3();
	<span style="color: #0000ff">public</span> <span style="color: #2b91af">string</span>  Str1;
	<span style="color: #008000">// Простой способ для отображения вложенных объектов и их полей – </span>
	<span style="color: #008000">// задать формат алиаса.</span>
	[MapField(Format=&quot;InnerObject_{0}&quot;]
	<span style="color: #0000ff">public</span> Object2 InnerObject = <span style="color: #0000ff">new</span> Object2();
}
</pre></div>

**MapValueAttribute** – позволяет задать для значений их синонимы. Мы уже сталкивались с данным атрибутом, при отображении перечислений, но этим его возможности не заканчиваются, приведу еще один пример:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">Object1</span>
{
	[MapValue(true,  &quot;Y&quot;)]
	[MapValue(false, &quot;N&quot;)]
	<span style="color: #0000ff">public</span> <span style="color: #2b91af">bool</span> Bool1;

	[MapValue(true,  &quot;Y&quot;, &quot;Yes&quot;)]
	[MapValue(false, &quot;N&quot;, &quot;No&quot;)]
	<span style="color: #0000ff">public</span> <span style="color: #2b91af">bool</span> Bool2;
}
</pre></div>

Использовать атрибут можно так же и на весь класс, задавая таким образом синонимы по умолчанию для полей данного типа:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%">[MapValue(true,  &quot;Y&quot;)]
[MapValue(false, &quot;N&quot;)]
<span style="color: #0000ff">public</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">Object2</span>
{
	<span style="color: #0000ff">public</span> <span style="color: #2b91af">bool</span> Bool1;

	[MapValue(true,  &quot;Y&quot;, &quot;Yes&quot;)]
	[MapValue(false, &quot;N&quot;, &quot;No&quot;)]
	<span style="color: #0000ff">public</span> <span style="color: #2b91af">bool</span> Bool2;
}
</pre></div>

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

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> <span style="color: #0000ff">enum</span> Gender
{
	[MapValue(&quot;F&quot;)] Female,
	[MapValue(&quot;M&quot;)] Male,
	[MapValue(&quot;U&quot;)] Unknown,
	[MapValue(&quot;O&quot;)] Other
}
<span style="color: #0000ff">public</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">Person</span>
{
	[MapField(&quot;PersonID&quot;), NonUpdatable, PrimaryKey]
	<span style="color: #0000ff">public</span> <span style="color: #2b91af">int</span>    ID;
	<span style="color: #0000ff">public</span> <span style="color: #2b91af">string</span> FirstName;
	<span style="color: #0000ff">public</span> <span style="color: #2b91af">string</span> MiddleName;
	<span style="color: #0000ff">public</span> <span style="color: #2b91af">string</span> LastName;
	<span style="color: #0000ff">public</span> Gender Gender;
}

[Test]
<span style="color: #0000ff">public</span> <span style="color: #0000ff">void</span> SqlQueryTest()
{
    SqlQuery&lt;Person&gt; da = <span style="color: #0000ff">new</span> SqlQuery&lt;Person&gt;();
    
    Person p1 = da.SelectByKey(1);
    
    Assert.IsNotNull(p1);
    Assert.AreEqual(1,           p1.ID);
    Assert.AreEqual(<span style="color: #a31515">&quot;John&quot;</span>,      p1.FirstName);
    Assert.AreEqual(Gender.Male, p1.Gender);

    p1.ID        = 101;
    p1.FirstName = <span style="color: #a31515">&quot;John II&quot;</span>;
    
    <span style="color: #2b91af">int</span> r = da.Update(p1);

    Assert.AreEqual(1, r);

    Person p2 = da.SelectByKey(1);

    Assert.IsNotNull(p2);
    Assert.AreEqual(1,           p2.ID);
    Assert.AreEqual(<span style="color: #a31515">&quot;John II&quot;</span>,   p2.FirstName);
    Assert.AreEqual(Gender.Male, p2.Gender);

    da.Delete(p1); <span style="color: #008000">// da.DeleteByKey(1);</span>
    
    p2 = da.SelectByKey(p1);

    Assert.IsNull(p2);

    List&lt;Person&gt; persons = da.SelectAll();

    Assert.IsNotNull(persons);
}
</pre></div>

В ходе данного теста были сгенерированы и выполнены следующие запросы:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #008000">-- SelectByKey</span>
<span style="color: #0000ff">SELECT</span>
	[PersonId],
	[FirstName],
	[MiddleName],
	[LastName],
	[Gender]
<span style="color: #0000ff">FROM</span> [Person]
<span style="color: #0000ff">WHERE</span> [PersonId] = @PersonId_W 

<span style="color: #008000">-- Update</span>
<span style="color: #0000ff">UPDATE</span> [Person] 
<span style="color: #0000ff">SET</span>
	[FirstName] = @FirstName,
	[MiddleName] = @MiddleName,
	[LastName] = @LastName,
	[Gender] = @Gender
<span style="color: #0000ff">WHERE</span> [PersonId] = @PersonId_W 

<span style="color: #008000">-- Delete, DeleteByKey</span>
<span style="color: #0000ff">DELETE</span> <span style="color: #0000ff">FROM</span> [Person]
<span style="color: #0000ff">WHERE</span> [PersonId] = @PersonId_W 

<span style="color: #008000">-- SelectAll</span>
<span style="color: #0000ff">SELECT</span>
	[PersonId],
	[FirstName],
	[MiddleName],
	[LastName],
	[Gender]
<span style="color: #0000ff">FROM</span> [Person]
</pre></div>

Как вы видите в коде запросов используются символы экранирования и имена параметров специфичные для MS SQL Server, поэтому следует оговориться, что в генерации запросов участвует *DataProvider*, а если быть точным то его метод *Convert(...)*, именно через него код запроса «наделяется» спецификой конкретного сервера. Если бы запрос генерировался с использованием *OdpDataProvider* то он бы выглядел примерно так:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">SELECT</span>
	PersonId,
	FirstName,
	MiddleName,
	LastName,
	Gender
<span style="color: #0000ff">FROM</span> Person
<span style="color: #0000ff">WHERE</span> PersonId = :PersonId_W 
</pre></div>

А для *FdpDataProvider* так:

<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">SELECT</span>
	<span style="color: #a31515">&quot;PersonId&quot;</span>,
	<span style="color: #a31515">&quot;FirstName&quot;</span>,
	<span style="color: #a31515">&quot;MiddleName&quot;</span>,
	<span style="color: #a31515">&quot;LastName&quot;</span>,
	<span style="color: #a31515">&quot;Gender&quot;</span>
<span style="color: #0000ff">FROM</span> <span style="color: #a31515">&quot;Person&quot;</span>
<span style="color: #0000ff">WHERE</span> <span style="color: #a31515">&quot;PersonId&quot;</span> = @PersonId_W 
</pre></div>

Так же следует отметить, что генерация запроса происходит только при первом обращении, после чего запрос кэшируется и при следующих обращениях возвращается из кэша.



<span id="dataaccessor"></span>
### DataAcessor

В отличие от *SqlQuery*, используемого для генерации SQL запросов, *DataAccessor* используется для эмита кода, что избавляет программиста от нудных и рутинных операций. Для начала рассмотрим небольшой пример: 

<!-- HTML generated using hilite.me --><div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">namespace</span> DataAccessorTest
{
    <span style="color: #0000ff">public</span> <span style="color: #0000ff">enum</span> Gender
    {
                [MapValue(&quot;F&quot;)] Female,
                [MapValue(&quot;M&quot;)] Male,
                [MapValue(&quot;U&quot;)] Unknown,
                [MapValue(&quot;O&quot;)] Other
    }
        <span style="color: #0000ff">public</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">Person</span>
    {
                [MapField(&quot;PersonID&quot;)]
                <span style="color: #0000ff">public</span> <span style="color: #2b91af">int</span>    ID;
                <span style="color: #0000ff">public</span> <span style="color: #2b91af">string</span> FirstName;
                <span style="color: #0000ff">public</span> <span style="color: #2b91af">string</span> MiddleName;
                <span style="color: #0000ff">public</span> <span style="color: #2b91af">string</span> LastName;
                <span style="color: #0000ff">public</span> Gender Gender;
    } 

    <span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">PersonAccessor</span> : DataAccessor&lt;Person, PersonAccessor&gt;
    {
         [SqlQuery(@&quot;
               SELECT 
                   p.PersonId,
                   p.FirstName,
                   p.SecondName,
                   p.MiddleName,
                   p.Gender
               FROM Person p
               WHERE p.PersonId = @PersonId&quot;)]
        <span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> Person GetPerson(<span style="color: #2b91af">int</span> personId);
    }

    [Test]
    <span style="color: #0000ff">public</span> <span style="color: #0000ff">void</span> Test()
    {
        PersonAccessor pa = PersonAccessor.CreateInstance();

        Person p = pa.GetPerson(1);

        Assert.IsNotNull(p);
        Assert.AreEqual(<span style="color: #a31515">&quot;John&quot;</span>, p.FirstName);
    }
}
</pre></div>

Как видно, класс PersonAccessor – абстрактный, абстрактным так же является его метод GetPerson. При обращении к PersonAccessor.CreateInstance() BLToolkit эмитит сборку, где определяет наследника от PersonAccessor и реализует абстрактные методы, ссылку на экземпляр этого наследника и возвращает CreateInstance(). Если посмотреть на реализацию этого класса (допустим, при помощи  Reflector), то мы увидим примерно следующее:

<!-- HTML generated using hilite.me --><div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%">[BLToolkitGenerated]
<span style="color: #0000ff">public</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">PersonAccessor</span> : DataAccessorTest.PersonAccessor
{
    DataAccessorTest.Person person = <span style="color: #0000ff">null</span>;
    DbManager dbManager = <span style="color: #0000ff">this</span>.GetDbManager();
    <span style="color: #0000ff">try</span>
    {
        Type type = <span style="color: #0000ff">typeof</span>(DataAccessorTest.Person);
        IDbDataParameter[] parameters = <span style="color: #0000ff">new</span> IDbDataParameter[] 
            { 
                dbManager.Parameter(
                    <span style="color: #0000ff">this</span>.GetQueryParameterName(dbManager, <span style="color: #a31515">&quot;personId&quot;</span>), 
                    personId) };

        person = (DataAccessorTest.Person) dbManager
            .SetCommand(<span style="color: #a31515">&quot;SELECT p.PersonId, p.FirstName, p.SecondName, p.MiddleName, p.Gender FROM Person WHERE p.PersonId = @PersonId&quot;</span>, <span style="color: #0000ff">this</span>.PrepareParameters(dbManager, parameters))
            .ExecuteObject(type);
    }
    <span style="color: #0000ff">finally</span>
    {
        <span style="color: #0000ff">this</span>.Dispose(dbManager);
    }
    <span style="color: #0000ff">return</span> person;
}
</pre></div>

Если не вдаваться в детали, то BLToolkit за нас написал примерно следующее: 

<!-- HTML generated using hilite.me --><div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">using</span> (DbManager db = <span style="color: #0000ff">new</span> DbManager())
{
    <span style="color: #0000ff">return</span> db
        .SetCommand<span style="color: #a31515">@&quot;</span>
<span style="color: #a31515">                     SELECT </span>
<span style="color: #a31515">                         p.PersonId,</span>
<span style="color: #a31515">                         p.FirstName,</span>
<span style="color: #a31515">                         p.SecondName,</span>
<span style="color: #a31515">                         p.MiddleName,</span>
<span style="color: #a31515">                         p.Gender</span>
<span style="color: #a31515">                     FROM Person p</span>
<span style="color: #a31515">                     WHERE p.PersonId = @PersonId&quot;</span>,
            db.Parameter(<span style="color: #a31515">&quot;@PersonId&quot;</span>, personId)
        .ExecuteObject(<span style="color: #0000ff">typeof</span>(Person));
}
</pre></div>

Иными словами, при помощи DataAccessor можно задекларировать какой код нам нужен для работы с данными, и по этой декларации BLToolkit сгенерирует за программиста этот код. Декларация происходит методом объявления наследника от DataAccessor и объявления в нем методов, каждая часть объявления метода имеет свое значение, а именно: 

- Возвращаемое значение.
- Имя метода.
- Параметры.
- Атрибуты, примененные к методу и его параметрам. 



<span id="dataaccessorreturntype"></span>
#### Возвращаемое значение

По типу возвращаемого значения определяется метод DbManager, который следует вызвать: 

<table border="1" cellspacing="0" cellpadding="2">
	<thead>
		<tr>
			<th>Тип</th>
			<th>Метод</th>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td>IDataReader</td>
			<td>ExecuteDataReader</td>
		</tr>
		<tr>
			<td>Наследник от DataSet</td>
			<td> ExecuteDataSet </td>
		</tr>
		<tr>
			<td>Наследник от DataTable</td>
			<td>ExecuteDataTable</td>
		</tr>
		<tr>
			<td>Наследник от IList</td>
			<td>ExecuteList или ExecuteScalarList</td>
		</tr>
		<tr>
			<td>Наследник от IDictionary</td>
			<td>ExecuteDictionary или ExecuteScalarDictionary</td>
		</tr>
		<tr>
			<td>void</td>
			<td>ExecuteNonQuery</td>
		</tr>
		<tr>
			<td>ValueType (int, string, byte[])</td>
			<td>ExecuteScalar</td>
		</tr>
		<tr>
			<td>Иное</td>
			<td>ExecuteObject</td>
		</tr>
	</tbody>
</table>



<span id="dataaccessormethodname"></span>
#### Имя метода

По умолчанию DataAccessor использует хранимые процедуры. По имени метода определяется имя хранимой процедуры. Нотация следующая: [Имя типа]_[Имя Процедуры] 

<!-- HTML generated using hilite.me --><div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> List&lt;Person&gt; SelectAll() <span style="color: #008000">// вызовет Person_SelectAll</span>
<span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> <span style="color: #0000ff">void</span> Insert(Person p) <span style="color: #008000">// вызовет Person_Insert</span>
</pre></div>

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

<!-- HTML generated using hilite.me --><div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%">[SprocActionName(&quot;GetPersons&quot;, &quot;Person_SelectAll&quot;)]
<span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">PersonAccessor</span> : DataAccessor&lt;Person, PersonAccessor&gt;
{
    <span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> List&lt;Person&gt; SelectAll();
    
    [Action(&quot;SelectAll&quot;)]
    <span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> List&lt;Person&gt; GetAll();

    [SprocName(&quot;Person_SelectAll&quot;)]
    <span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> List&lt;Person&gt; LoadPersons();

    <span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> List&lt;Person&gt; GetPersons();
}
</pre></div>

**DiscoverParametersAttribute** – при использовании хранимых процедур через DataAccessor по умолчанию полагается, что имена параметров функции совпадают с именем параметров хранимки, в таком случае порядок параметров в функции не имеет значения. Если же к функции применен данный атрибут BLToolkit получит информацию о параметрах хранимой процедуры и применит ее к параметрам метода по порядку, в данном случае имена параметров будут проигнорированы. 

<!-- HTML generated using hilite.me --><div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%">[TestFixture]
<span style="color: #0000ff">public</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">DiscoverParameters</span>
{
    <span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">PersonAccessor</span> : DataAccessor
    {
        [DiscoverParameters]
        <span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> Person SelectByName(<span style="color: #2b91af">string</span> anyParameterName, <span style="color: #2b91af">string</span> rParameterName);
    }

    [Test]
    <span style="color: #0000ff">public</span> <span style="color: #0000ff">void</span> Test()
    {
        PersonAccessor pa = DataAccessor.CreateInstance&lt;PersonAccessor&gt;();
        Person         p  = pa.SelectByName(<span style="color: #a31515">&quot;Tester&quot;</span>, <span style="color: #a31515">&quot;Testerson&quot;</span>);

        Assert.AreEqual(2, p.ID);
    }
}
</pre></div>

**SqlQueryAttribute** – говорит методу, что следует выполнить указанный SQL запрос. Выше уже приводился стандартный метод использования данного атрибута. Кроме того, у атрибута есть свойство IsDynamic?. Если данное свойство выставлено в true то для получения кода запроса используется метод атрибута GetSqlText?(…). 

Рассмотрим пример: 

<!-- HTML generated using hilite.me --><div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">TestQueryAttribute</span> : SqlQueryAttribute
{
        <span style="color: #0000ff">public</span> TestQueryAttribute()
        {
                IsDynamic = <span style="color: #0000ff">true</span>;
        }

        <span style="color: #0000ff">private</span> <span style="color: #2b91af">string</span> _oracleText;
        <span style="color: #0000ff">public</span>  <span style="color: #2b91af">string</span>  OracleText
        {
                <span style="color: #0000ff">get</span> { <span style="color: #0000ff">return</span> _oracleText;  }
                <span style="color: #0000ff">set</span> { _oracleText = <span style="color: #0000ff">value</span>; }
        }

        <span style="color: #0000ff">public</span> <span style="color: #0000ff">override</span> <span style="color: #2b91af">string</span> GetSqlText(DataAccessor accessor, DbManager dbManager)
        {
                <span style="color: #0000ff">switch</span> (dbManager.DataProvider.Name)
                {
                        <span style="color: #0000ff">case</span> <span style="color: #a31515">&quot;Sql&quot;</span>   :
                        <span style="color: #0000ff">case</span> <span style="color: #a31515">&quot;SqlCe&quot;</span> : <span style="color: #0000ff">return</span> SqlText;

                        <span style="color: #0000ff">case</span> <span style="color: #a31515">&quot;Oracle&quot;</span>:
                        <span style="color: #0000ff">case</span> <span style="color: #a31515">&quot;ODP&quot;</span>   : <span style="color: #0000ff">return</span> OracleText ?? SqlText;
                }

                <span style="color: #0000ff">throw</span> <span style="color: #0000ff">new</span> ApplicationException(<span style="color: #2b91af">string</span>.Format(<span style="color: #a31515">&quot;Unknown data ider &#39;{0}&#39;&quot;</span>, dbManager.DataProvider.Name));
        }
}

<span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">PersonAccessor</span> : DataAccessor&lt;Person, Person&gt;
{
        [TestQuery(
                SqlText    = &quot;SELECT * FROM Person WHERE LastName = @lastName&quot;,
                OracleText = &quot;SELECT * FROM Person WHERE LastName = :lastName&quot;)]
        <span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> List&lt;Person&gt; SelectByLastName(<span style="color: #2b91af">string</span> lastName);
}
</pre></div>

В данном примере, в зависимости от имени используемого DataProvider-а выполняются разные запросы. 

**FormatAttribute** – применяется к параметру, указывает, что данный параметр используется для генерации текста SQL запроса или имени хранимой процедуры.

<!-- HTML generated using hilite.me --><div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">PersonAccessor</span> : DataAccessor
{
    [SqlQuery(&quot;SELECT TOP {0} * FROM Person&quot;)]
    <span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> List&lt;Person&gt; GetPersonList([Format] <span style="color: #2b91af">int</span> top);
}

[Test]
<span style="color: #0000ff">public</span> <span style="color: #0000ff">void</span> Test()
{
    PersonAccessor pa   = DataAccessor.CreateInstance&lt;PersonAccessor&gt;();
    List&lt;Person&gt;   list = pa.GetPersonList(2); <span style="color: #008000">// SELECT TOP 2 * FROM Person</span>

    Assert.That(list,       Is.Not.Null);
    Assert.That(list.Count, Is.LessThanOrEqualTo(2));
}
</pre></div>

**ParamNameAttribute** – позволяет явно задать имя параметра.

**ParamDbTypeAttribute** – позволяет явно задать тип параметра.

**ParamSizeAttribute** – позволяет явно задать размер параметра.

**ParamNullValueAttribute** – позволяет указать какое значение параметра считать за NULL. 

<!-- HTML generated using hilite.me --><div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">TestAccessor</span> : DataAccessor
{

    [SqlQuery(&quot;SELECT * FROM Person WHERE PersonID = @personId&quot;)]
    <span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> Person SelectByKey([ParamName(<span style="color: #a31515">&quot;personId&quot;</span>)]<span style="color: #2b91af">int</span> id);

    <span style="color: #008000">// при id == 1 значение параметра будет заменено на NULL</span>
    <span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> Person SelectByKey([ParamNullValue(1)] <span style="color: #2b91af">int</span> id);

    [SqlQuery(&quot;SELECT {0} = {1} FROM Person WHERE PersonID = 1&quot;)]
    <span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> <span style="color: #0000ff">void</span> SelectJohn(
        [ParamSize(50), ParamDbType(DbType.String)] <span style="color: #0000ff">out</span> <span style="color: #2b91af">string</span> name,
        [Format] <span style="color: #2b91af">string</span> paramName,
        [Format] <span style="color: #2b91af">string</span> fieldName); 
}

[Test]
<span style="color: #0000ff">public</span> <span style="color: #0000ff">void</span> AccessorTest()
{
    <span style="color: #0000ff">using</span> (DbManager db = <span style="color: #0000ff">new</span> DbManager())
    {
        TestAccessor ta = DataAccessor.CreateInstance&lt;TestAccessor&gt;(db);

        <span style="color: #2b91af">string</span> actualName;

        <span style="color: #008000">// SELECT @name  = FirstName FROM Person WHERE PersonID = 1</span>
        <span style="color: #008000">// полученое значение будет возвращено в параметр name</span>
        ta.SelectJohn(<span style="color: #0000ff">out</span> actualName, <span style="color: #a31515">&quot;@name&quot;</span>, <span style="color: #a31515">&quot;FirstName&quot;</span>);

        Assert.AreEqual(<span style="color: #a31515">&quot;John&quot;</span>, actualName);
    }
}
</pre></div>

**DirectionAttribute** – позволяет для параметра явно задать его направление.

**DestinationAttribute** – указывает, что в данный параметр следует отмапить выбранные значения. 

<!-- HTML generated using hilite.me --><div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">PersonAccessor</span> : DataAccessor&lt;Person, PersonAccessor&gt;
{
    [SqlQuery(&quot;SELECT * FROM Person&quot;)]
    <span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> <span style="color: #0000ff">void</span> SelectAll([Destination]List&lt;Person&gt; list);
    
    <span style="color: #008000">// CREATE Procedure Person_Insert_OutputParameter</span>
    <span style="color: #008000">//   @FirstName  nvarchar(50),</span>
    <span style="color: #008000">//   @LastName   nvarchar(50),</span>
    <span style="color: #008000">//   @MiddleName nvarchar(50),</span>
    <span style="color: #008000">//   @Gender     char(1),</span>
    <span style="color: #008000">//   @PersonID   int output</span>
    <span style="color: #008000">//   AS </span>
    <span style="color: #008000">//</span>
    <span style="color: #008000">//     INSERT INTO Person</span>
    <span style="color: #008000">//       ( LastName,  FirstName,  MiddleName,  Gender)</span>
    <span style="color: #008000">//     VALUES</span>
    <span style="color: #008000">//       (@LastName, @FirstName, @MiddleName, @Gender)</span>
    <span style="color: #008000">//</span>
    <span style="color: #008000">//     SET @PersonID = Cast(SCOPE_IDENTITY() as int)</span>
    [SprocName(&quot;Person_Insert_OutputParameter&quot;)]
    <span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> <span style="color: #0000ff">void</span> Insert([Direction.Output(<span style="color: #a31515">&quot;PersonId&quot;</span>)] Person p);
}
</pre></div>

**IndexAttribute** – позволяет указать индекс для словаря (по умолчанию используется первичный ключ): 

<!-- HTML generated using hilite.me --><div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">PersonAccessor</span> : DataAccessor&lt;Person&gt;
{
    <span style="color: #008000">// Для ключа словаря будет использован первичный ключ объекта Person,</span>
    <span style="color: #008000">// т.е. поля, помеченные атрибутом PrimaryKey</span>
    [ActionName(&quot;SelectAll&quot;)]
    <span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> Dictionary&lt;<span style="color: #2b91af">int</span>,Person&gt; GetPersonDictionary1();

    <span style="color: #008000">// Явно задаем индекс. ID – поле класса Person.</span>
    <span style="color: #008000">//</span>
    [ActionName(&quot;SelectAll&quot;)]
    [Index(&quot;ID&quot;)]
    <span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> Dictionary&lt;<span style="color: #2b91af">int</span>,Person&gt; GetPersonDictionary2();

    <span style="color: #008000">// Явно задаем индекс. </span>
    <span style="color: #008000">// &quot;@PersonID&quot;- поле полученного в результате выборки кортежа!.</span>
    <span style="color: #008000">// Важно: собачка - &#39;@&#39; заставляет BLToolkit для индекса </span>
    <span style="color: #008000">// брать значения из полей кортежа!</span>
    <span style="color: #008000">//</span>
    [ActionName(&quot;SelectAll&quot;)]
    [Index(&quot;@PersonID&quot;)]
    <span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> Dictionary&lt;<span style="color: #2b91af">int</span>,Person&gt; GetPersonDictionary3();

    <span style="color: #008000">// Будет вычитан словарь со скалярнми величинами.</span>
    <span style="color: #008000">//</span>
    [SqlQuery(&quot;SELECT PersonID, FirstName FROM Person&quot;)]
    [Index(&quot;PersonID&quot;)]
    [ScalarFieldName(&quot;FirstName&quot;)]
    <span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> Dictionary&lt;<span style="color: #2b91af">int</span>,<span style="color: #2b91af">string</span>&gt; GetPersonNameDictionary();
}
</pre></div>

**ObjectTypeAttribute** – явно задает тип возвращаемого абстрактным методом объекта.

**ActualTypeAttribute** – явно задает тип возвращаемого абстрактным методом объекта, имеет более низкий приоритет чем ObjectType.

<!-- HTML generated using hilite.me --><div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> <span style="color: #0000ff">interface</span> IName
{
    <span style="color: #2b91af">string</span> Name { <span style="color: #0000ff">get</span>; }
}

<span style="color: #0000ff">public</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">NameBase</span> : IName
{
    <span style="color: #0000ff">private</span> <span style="color: #2b91af">string</span> _name;
    <span style="color: #0000ff">public</span>  <span style="color: #2b91af">string</span>  Name { <span style="color: #0000ff">get</span> { <span style="color: #0000ff">return</span> _name; } <span style="color: #0000ff">set</span> { _name = <span style="color: #0000ff">value</span>; } }
}

<span style="color: #0000ff">public</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">Name1</span> : NameBase {}
<span style="color: #0000ff">public</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">Name2</span> : NameBase {}

[ActualType(typeof(IName), typeof(Name1))]
<span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">TestAccessor</span> : DataAccessor
{
    <span style="color: #008000">// Вернет объект класса Name1</span>
    [SqlQuery(&quot;SELECT &#39;John&#39; as Name&quot;)]
    <span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> IName GetName1();

    <span style="color: #008000">// Вернет объект класса Name2</span>
    [SqlQuery(&quot;SELECT &#39;John&#39; as Name&quot;), ObjectType(typeof(Name2))]
    <span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> IName GetName2();

    [SqlQuery(&quot;SELECT &#39;John&#39; as Name&quot;)]
    <span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> IList&lt;IName&gt; GetName1List();

    [SqlQuery(&quot;SELECT &#39;John&#39; as Name&quot;), ObjectType(typeof(Name2))]
    <span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> IList&lt;IName&gt; GetName2List();

    [SqlQuery(&quot;SELECT 1 as ID, &#39;John&#39; as Name&quot;), Index(&quot;@ID&quot;)]
    <span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> IDictionary&lt;<span style="color: #2b91af">int</span>, IName&gt; GetName1Dictionary();

    [SqlQuery(&quot;SELECT 1 as ID, &#39;John&#39; as Name&quot;), Index(&quot;@ID&quot;),
     ObjectType(typeof(Name2))]
    <span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> IDictionary&lt;<span style="color: #2b91af">int</span>, IName&gt; GetName2Dictionary();
}


[ObjectType(typeof(Person)]
<span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">PersonAccessor</span> : DataAccessor
{
    <span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> ArrayList SelectAll();

    <span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> <span style="color: #2b91af">object</span> SelectByKey(<span style="color: #2b91af">int</span> personId);
}
</pre></div>



<span id="dataaccessrecommends"></span>
### Рекомендации по использванию


<span id="dataaccessrecommendsmanual"></span>
#### Реализация методов ручками

Эмит кода, это конечно хорошо, но переодически возникает необходимость сделать метод руками, в таком случае рекомендуется делать это так: 

<!-- HTML generated using hilite.me --><div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">MyAccessor</span> : DataAccessor
{
    <span style="color: #0000ff">public</span> List&lt;Person&gt; SelectAll()
    {
        DbManager db = GetDbManager();
        <span style="color: #0000ff">try</span>
        {
            <span style="color: #0000ff">return</span> SelectAll(db);
        }
        <span style="color: #0000ff">finally</span>
        {
            Dispose(db);
        }
    }

    <span style="color: #0000ff">public</span> SelectAll(DbManager db)
    {
        <span style="color: #0000ff">return</span> db.SetCommand(<span style="color: #a31515">&quot;SELECT * FROM Person&quot;</span>).ExecuteList&lt;Person&gt;();
    }
}
</pre></div>

Это типовой шаблон реализации методов как для всех наследников DataAccessorBase, коими являются как DataAccessor так и SqlQuery. Ключевыми являются использование функций GetDbManager() и Dispose(DbManager db). Первый возвращает эеземпляр DbManager, переданный в конструктор, если передавался, иначе новый экзкмпляр. Второй освобождает экземпляр DbManager, в случае если оный не был передан через конструктор. 



<span id="dataaccessrecommendsabstract"></span>
#### Генерация SQL запросов в абстрактном аксессоре

Абстрактные аксессоры не поддерживают генерацию SQL запросов, однако, при необходимости можно реализовать своего наследника, допустим, следующего вида: 

<!-- HTML generated using hilite.me --><div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;"><pre style="margin: 0; line-height: 125%"><span style="color: #0000ff">public</span> <span style="color: #0000ff">abstract</span> <span style="color: #0000ff">class</span> <span style="color: #2b91af">MyAccessorBase</span>&lt;T, TA&gt; : DataAccessor&lt;T, TA&gt; <span style="color: #0000ff">where</span> TA : DataAccessor&lt;T&gt;
{
        SqlQuery&lt;T&gt; _query = <span style="color: #0000ff">new</span> SqlQuery&lt;T&gt;();

        <span style="color: #0000ff">private</span> <span style="color: #0000ff">delegate</span> R Func&lt;P1, P2, R&gt;(P1 par1, P2 par2);
        <span style="color: #0000ff">private</span> <span style="color: #0000ff">delegate</span> R Func&lt;P1, R&gt;(P1 par1);

        <span style="color: #0000ff">private</span> R Exec&lt;R, P&gt;(Func&lt;DbManager, P, R&gt; op, P obj)
        {
                DbManager db = GetDbManager();
                <span style="color: #0000ff">try</span>
                {
                        <span style="color: #0000ff">return</span> op(db, obj);
                }
                <span style="color: #0000ff">finally</span>
                {
                        Dispose(db);
                }
        }

        <span style="color: #0000ff">private</span> R Exec&lt;R&gt;(Func&lt;DbManager, R&gt; op)
        {
                DbManager db = GetDbManager();
                <span style="color: #0000ff">try</span>
                {
                        <span style="color: #0000ff">return</span> op(db);
                }
                <span style="color: #0000ff">finally</span>
                {
                        Dispose(db);
                }
        }

        <span style="color: #0000ff">public</span> <span style="color: #0000ff">virtual</span> <span style="color: #2b91af">int</span>  Insert(T obj)
        {
                <span style="color: #0000ff">return</span> Exec&lt;<span style="color: #2b91af">int</span>, T&gt;(Insert, obj);
        }
                        
        <span style="color: #0000ff">public</span> <span style="color: #0000ff">virtual</span> <span style="color: #2b91af">int</span> Insert(DbManager db, T obj)
        {
                <span style="color: #0000ff">return</span> _query.Insert(db, obj);
        }

        <span style="color: #0000ff">public</span> <span style="color: #0000ff">virtual</span> <span style="color: #2b91af">int</span> Update(T obj)
        {
                <span style="color: #0000ff">return</span> Exec&lt;<span style="color: #2b91af">int</span>, T&gt;(Update, obj);
        }

        <span style="color: #0000ff">public</span> <span style="color: #0000ff">virtual</span> <span style="color: #2b91af">int</span> Update(DbManager db, T obj)
        {
                <span style="color: #0000ff">return</span> _query.Update(db, obj);
        }

        <span style="color: #0000ff">public</span> <span style="color: #0000ff">virtual</span> <span style="color: #2b91af">int</span> Delete(T obj)
        {
                <span style="color: #0000ff">return</span> Exec&lt;<span style="color: #2b91af">int</span>, T&gt;(Delete, obj);
        }
                        
        <span style="color: #0000ff">public</span> <span style="color: #0000ff">virtual</span> <span style="color: #2b91af">int</span> Delete(DbManager db, T obj)
        {
                <span style="color: #0000ff">return</span> _query.Delete(db, obj);
        }

        <span style="color: #0000ff">public</span> <span style="color: #0000ff">virtual</span> <span style="color: #2b91af">int</span> DeleteByKey(<span style="color: #2b91af">object</span>[] keys)
        {
                <span style="color: #0000ff">return</span> Exec&lt;<span style="color: #2b91af">int</span>, <span style="color: #2b91af">object</span>[]&gt;(DeleteByKey, keys);
        }

        <span style="color: #0000ff">public</span> <span style="color: #0000ff">virtual</span> <span style="color: #2b91af">int</span> DeleteByKey(DbManager db, <span style="color: #2b91af">object</span>[] keys)
        {
                <span style="color: #0000ff">return</span> _query.DeleteByKey(db, keys);
        }

        <span style="color: #0000ff">public</span> <span style="color: #0000ff">virtual</span> List&lt;T&gt; SelectAll()
        {
                <span style="color: #0000ff">return</span> Exec&lt;List&lt;T&gt;&gt;(SelectAll);
        }

        <span style="color: #0000ff">public</span> <span style="color: #0000ff">virtual</span> List&lt;T&gt; SelectAll(DbManager db)
        {
                <span style="color: #0000ff">return</span> _query.SelectAll(db);
        }

        <span style="color: #0000ff">public</span> <span style="color: #0000ff">virtual</span> T SelectByKey(<span style="color: #2b91af">object</span>[] keys)
        {
                <span style="color: #0000ff">return</span> Exec&lt;T, <span style="color: #2b91af">object</span>[]&gt;(SelectByKey, keys);
        }

        <span style="color: #0000ff">public</span> <span style="color: #0000ff">virtual</span> T SelectByKey(DbManager db, <span style="color: #2b91af">object</span>[] keys)
        {
                <span style="color: #0000ff">return</span> _query.SelectByKey(db, keys);
        }
}
</pre></div>




## Исходники

Тут описание структуры исходников, что где и зачем.



<span id="materials"></span>
## При написании использовалось

- Редактор markdown [MarkdownPad](http://markdownpad.com/).
- Подсветка синтаксиса [http://hilite.me/](http://hilite.me/).
- Кэш из вебархива ([http://web.archive.org/web/20131106072643/http://projects.rsdn.ru/RFD/wiki/BLToolkit](http://web.archive.org/web/20131106072643/http://projects.rsdn.ru/RFD/wiki/BLToolkit)) оригинальной страницы [http://projects.rsdn.ru/RFD/wiki/BLToolkit](http://projects.rsdn.ru/RFD/wiki/BLToolkit).
- Оригинальная документация [http://files.rsdn.ru/49168/BLToolkit.doc](http://files.rsdn.ru/49168/BLToolkit.doc).
- Исходник этой статьи на markdown - [http://liiws.bitbucket.org/bltoolkit.md](http://liiws.bitbucket.org/bltoolkit.md).

