#Execute a Stored Procedure that gets Data from Multiple Tables in EF Core

In EF Core there is not quite yet an elegant way to fetch data from multiple tables by a stored procedure. Lets build one.

##Why am I writing this.
The above issue is the reason why I feel this article is necessary. At the time of writing this article,
EF Core’s prescribed way of executing a stored procedure is context.Blogs.FromSql("EXEC Sp_YourSp")
but that is only possible if your stored procedure returns data from a particular DB Set (one table or one entity).
Lets look at how we can work it around until the above issue is resolved if a stored procedure gets data from multiple tables.

##The setup
We are going to recreate a scenario where we need to do just the above. We are going to write a small ‘Todo’ API.
Here is the [gist](https://gist.github.com/saurabhpati/23ed20815545baebee01c601f6591e53) link containing all the files shown below

**Entities/Models**
**User.cs**
```Code
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
