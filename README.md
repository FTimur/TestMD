## Обзор
- [SLA для бизнеса](#sla-для-бизнеса)
    - [Основные сущности](#основные-сущности)
    - [Основные сценарии](#основные-сценарии)
- [Реализация](#реализация)
    - [Сущности](#сущности)
        - [**Процесс**](#процесс)
        - [**Команда**](#команда)
        - [**Работник**](#работник)
        - [**Правило**](#правило)
        - [**Check**](#check)
        - [**Нарушение**](#нарушение)
        - [**Поля**](#поля)
        - [**Ticket**](#ticket)
        - [**DataSource**](#datasource)
    - [Добавление правила](#добавление-правила)
    - [Создание нарушения](#создание-нарушения)
    - [Отправка уведомления в Slack](#отправка-уведомления-в-slack)

## SLA для бизнеса
SLA(Service Level Agreement) - cоглашение об уровне сервиса. Разрабатываемое SLA используется как средство автоматического мониторинга рабочей деятельности сотрудников компании. В основном это средство следит за временными показателями задач различных команд от команд разработки до команд которые занимаются продажами.

### Основные сущности
* Процесс - (?) . В процессе может быть задействовано несколько команд. На каждый процесс существует отдельный Slack - канал. Примеры процессов: 
* Команда - группа людей, задействованных в процессе. Одна команда может быть задействована в нескольких проессах
* Правило - выставленные условия на процессы (Напр. одна карточка не может находиться в работе больше 5 дней и тд.)
* Нарушение - несоответствие правилу. Определяется временем просрочки.

### Основные сценарии
Добавление процесса:
Для добавления процесса на странице продукта
## Реализация
### Сущности
Для понимания работы приведен упрощенный код. 
#### **Процесс**
Процесс состоит из команды и списка правил. Также в процессе есть поле в котором можно указать канал в slack. Slack канал используется для ежедневного уведомления о процессе: разработки, продаж, и т.д.
```C#
public class Process
{
	int Id;

	string Name;

	string Description;

	string SlackChannel;

	Set<Rule> Rules;

	Set<Team> ProcessTeams;
}
```
#### **Команда**
```C#
public class Team
{
	int Id;

	Set<Employee> Employees;

	string ExternalId;

	string Name;

	string SlackChannel;
}
```
#### **Работник**
```C#
public class Employee
{
	int Id;

	string Name;

	Set<Team> Team;

	string TrelloUsername;

	string PipedriveEmail;

	string SlackUsername;

	string FogBugzUsername;
}
```

#### **Правило** 
Описывается абстрактным классом `Rule.cs`, в котором так же содержится вся логика созданий нарушений (см. Создание нарушения). В унаследованных `PipedriveRule.sc`, `FogBugzRule.sc`, `TrelloRule.sc` переопределен метод `GetTickets(...)`, который возвращает ticket'ы из соответствующего DataSource. Так же у каждого правила есть свойство `Settings`, которое является строкой и в дальнейшем десериализуется при проверке нарушения в специфичные для данного правила настройки.
```C#
public abstract class Rule
{
	int Id;

	Set<Process> Process;

	string Discriminator;

	string Settings;

	string CheckSystemName;

	bool IsEnabled;

	bool IsDeleted;

	string SeverityLevels;

	string SeverityPeriodTypeSystemName;

	Check Check;
}
```
#### **Check**
 Check - это абстрактная модель в которой указан интерфейс для определения нарушения. Основным методом этого класса является `GetCheckResult` который возвращает результат проверки текущего правила. `RequiredFields` - используется для проверки возможности посчитать правило по полям которые есть в Ticket, если в нем недостаточно полей, то проверить такой Ticket нельзя. 
```C#
public abstract class Check : NamedObject
{
	string CheckSettingsType { get; }

    public abstract IEnumerable<CheckResult> GetCheckResult(
			IEnumerable<Ticket> tickets,
			CheckSettings checkSettings,
			ModelContext modelContext,
			AdjustablePeriodType severityPeriodType);

	abstract IEnumerable<Field> RequiredFields { get; }

	abstract CheckSettings DeserializeSettings(string settings);
}
```

#### **Нарушение**  
Описывается классом RuleViolation.cs, по сути напрямую транслирует все колонки из сущности бд на свойства. 

#### **Поля** 
Поле - это перечисление всех возможных полей которые могут быть в `Ticket`. Это перечисление используется в `проверках` для того чтобы определить возможность расчитать правило. Это перечисление нужно т.к. в обьекте `Ticket` может не быть некоторых полей, по которым расчитывается нарушение.
```C#
public enum Field
{
	OpenDateTime,

	WorkStartedDateTime,

	LastListChangeDate,

	Team,

	Employee,

	CheckLists,

	Steps,

	DueDate,

	Labels,

	Members,

	RelatedTicketsCount
}
```
#### **Ticket**  
Обобщенное понятие карточки (в trello - карточка, в pipedrive - сделка, в FogBugz - (?)). В коде  `Ticket.cs` - класс, содержащий в себе все возможные атрибуты карточки из разных сервисов как свойства. 
```C#
public class Ticket
{
	DateTime? OpenDateTime { get; set; }

	DateTime? WorkStartedDateTime { get; set; }

	DateTime? LastListChangeDate { get; set; }

	Team Team { get; set; }

	Employee Employee { get; set; }

	string SourceId { get; set; }

	string SourceName { get; set; }

	IEnumerable<CheckList> CheckLists { get; set; }

	IEnumerable<Step> Steps { get; set; }

	DateTime? DueDate { get; set; }

	IEnumerable<Label> Labels { get; set; }

	IEnumerable<Member> Members { get; set; }

	int? RelatedTicketsCount { get; set; }
}
```
#### **DataSource**  
Атрибуты, доступные в tickеt'е, определяются классом `DataSource.cs`, от которого унаследованы `TrelloDataSource.cs`, `PipedriveDataSource.cs`, `FogBugsDataSource.cs`.

### Добавление правила
Про добавление правила.

### Создание нарушения  
Вызывается `Rule.CreateViolations(...)`, где получается коллекция  `tickets`, дальше передается в метод `GetCheckResults(...)`, который возвращает коллекцию CheckResult для каждого ticket'а. После этого для каждого CheckResult'а и для каждой команды формируются нарушения и возвращается коллекция нарушений. 

### Отправка уведомления в Slack 
