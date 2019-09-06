# Execute a Stored Procedure that gets Data from Multiple Tables in EF Core

In EF Core there is not quite yet an elegant way to fetch data from multiple tables by a stored procedure. Lets build one.

## Why am I writing this.
The above issue is the reason why I feel this article is necessary. At the time of writing this article,
EF Core’s prescribed way of executing a stored procedure is context.Blogs.FromSql("EXEC Sp_YourSp")
but that is only possible if your stored procedure returns data from a particular DB Set (one table or one entity).
Lets look at how we can work it around until the above issue is resolved if a stored procedure gets data from multiple tables.

## The setup
We are going to recreate a scenario where we need to do just the above. We are going to write a small ‘Todo’ API.
Here is the [gist](https://gist.github.com/saurabhpati/23ed20815545baebee01c601f6591e53) link containing all the files shown below

**Entities/Models**

**User.cs**
```csharp
public class User : EntityBase
{
    public User()
    {
        UserTeams = new HashSet<UserTeam>();
        TaskItems = new HashSet<TaskItem>();
    }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string Email { get; set; }
    public string Username { get; set; }
    /// <summary>
    /// Gets or sets the tasks assigned to this user.
    /// </summary>
    public ICollection<TaskItem> TaskItems { get; set; }
    public ICollection<UserTeam> UserTeams { get; set; }
}
```

**Team.cs**
```csharp
public class Team : EntityBase
{
    public Team()
    {
        UserTeams = new HashSet<UserTeam>();
    }
    /// <summary>
    /// Gets or sets the name of the team.
    /// </summary>
    public string Name { get; set; }
    public ICollection<UserTeam> UserTeams { get; set; }
}
```
**UserTeam.cs**
```csharp
public class UserTeam : IEntityBase
{
    public int UserId { get; set; }
    public User User { get; set; }
    public int TeamId { get; set; }
    public Team Team { get; set; }
}
```
**TaskItem.cs**
```csharp
public class TaskItem : EntityBase
{
    public TaskItem()
    {
        Notes = new HashSet<Note>();
    }
    /// <summary>
    /// Gets or sets the name of the task.
    /// </summary>
    public string Name { get; set; }
    /// <summary>
    /// Gets or sets the description of the task.
    /// </summary>
    public string Description { get; set; }
    /// <summary>
    /// Gets or sets the status id of the task.
    /// </summary>
    public int StatusId { get; set; }
    /// <summary>
    /// Gets or sets the status of the task.
    /// </summary>
    public Status Status { get; set; }
    public int UserId { get; set; }
    /// <summary>
    /// Gets or sets the person this task is assigned to.
    /// </summary>
    public User User { get; set; }
}
```
**Status.cs**
```csharp
public class Status : EntityBase
{
    public Status()
    {
        TaskItems = new HashSet<TaskItem>();
    }
    /// <summary>
    /// Gets or sets the status name.
    /// </summary>
    public string Name { get; set; }
    /// <summary>
    /// Gets or sets the tasks currently in this status.
    /// </summary>
    public ICollection<TaskItem> TaskItems { get; set; }
}
```
**Entity.cs**
```csharp
/// <summary>
/// Will default the primary key of the entity to be int.
/// </summary>
public class EntityBase : EntityBase<int>
{
}
/// <summary>
/// The base entity.
/// </summary>
/// <typeparam name="TKey">The primary key. </typeparam>
public class EntityBase<TKey> : IEntityBase 
    where TKey: IEquatable<TKey>
{
    /// <summary>
    /// Gets or sets the Id of the entity
    /// </summary>
    public int Id { get; set; }
}
public interface IEntityBase
{
}
```

Now lets say we need to see a progress report where we need the information of teams and users along with the count of all tasks and the tasks which are in todo, in progress and done state. A stored procedure that accepts optional teamId and userId to get the progress report of all/one team(s) fits the solution to our requirement.

## The stored procedure
Lets create a stored procedure as discussed above.

**Sp_ProgressReport.sql**
```code
CREATE PROCEDURE [dbo].[Sp_ProgressReport]
@teamId INT NULL,
@userId INT NULL
AS
  SELECT ut.TeamId, ut.UserId, COUNT(t.Id) TotalTasks,
  SUM(CASE WHEN t.StatusId = 1 THEN 1 ELSE 0 END) Todo,
  SUM(CASE WHEN t.StatusId = 2 THEN 1 ELSE 0 END) InProgress,
  SUM(CASE WHEN t.StatusId = 3 THEN 1 ELSE 0 END) Done
  FROM
  [UserTeam] ut
  JOIN [TaskItem] t
  ON ut.UserId = t.UserId
  JOIN [Status] s
  ON t.StatusId = s.Id
  WHERE ut.TeamId = @teamId OR @teamId IS NULL
  AND 
  ut.UserId = @userId OR @userId IS NULL
  GROUP BY ut.TeamId, ut.UserId, s.Id
GO
```
## The solution
Lets create an entity which is the response of this stored procedure. You can omit creating an entity if you want to as you can directly map the response to an object or dynamic but it helps to have an entity when

* You have to modify or process the response to apply business logic.
* There are several related responses of such kind and you want to use some object oriented principles to reuse your entity.

**ProgressReportEntity.cs**
```csharp
public class ProgressReportEntity
{
    public int TeamId { get; set; }
    public int UserId { get; set; }
    public int TotalTasks { get; set; }
    public int Todo { get; set; }
    public int InProgress { get; set; }
    public int Done { get; set; }
}
```
We are going to create a repository for executing the stored procedure. I am not too great a fan of repositories but a lot of developers find it helpful, so I am going to stick to the repository pattern for now. You can create whatever data access abstraction you like.
Lets create a few extension methods that will help us irrespective of any pattern we follow.

**SprocRepositoryExtensions.cs**
![Repository Extension](https://github.com/levonaramyan/Coding_Best_Practices/blob/master/Execute%20a%20Stored%20Procedure%20that%20gets%20Data%20from%20Multiple%20Tables%20in%20EF%20Core/Repository_Extension.png)
