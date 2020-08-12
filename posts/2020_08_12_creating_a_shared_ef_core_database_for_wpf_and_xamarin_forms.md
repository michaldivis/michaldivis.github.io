# Creating a shared EF Core database project for use in WPF and Xamarin.Forms apps

Todays topic is how to use a single EntityFramework Core project in both WPF and Xamarin.Forms apps. In theory, it's dead simple, but there are a few little things you have tinker with in order to actually make it work. 

The structure of the example will be:
- Data (.NET Standard class library) - this project will contain the EF Core database context, data models and some helpers.
- Logic (.NET Standard class library) - this project will contain a simple view model that will be shared by the other apps. This will be referencing the Data project.
- DesktopApp (.NET Core WPF) - an example WPF app that will use PostgreSQL as it's database. This will be referencing the Data and Logic projects.
- MobileApp (Xamarin.Forms) - an example mobile app that will use SQLite as it's database. This will be referencing the Data and Logic projects.

All access to the database is going to be done strictly through the Data project.

Let's begin!

## Used versions

### SDK
- .NET Core 3.1
- .NET Standard 2.0
- Xamarin.Forms 4.8

### Nuget packages
- Microsoft.EntityFrameworkCore 3.1.7
- Npgsql.EntityFrameworkCore.PostgreSQL 3.1.4
- Microsoft.EntityFrameworkCore.Sqlite 3.1.7

## Installing nuget packages

We're going to need to install these nuget packages for these projects

### Data
- Microsoft.EntityFrameworkCore
- Microsoft.EntityFrameworkCore.Design
- Microsoft.EntityFrameworkCore.Tools
- Npgsql.EntityFrameworkCore.PostgreSQL
- Microsoft.EntityFrameworkCore.Sqlite

### Logic
- Microsoft.EntityFrameworkCore

### DesktopApp
- Microsoft.EntityFrameworkCore
- Npgsql.EntityFrameworkCore.PostgreSQL

### MobileApp
- Microsoft.EntityFrameworkCore
- Microsoft.EntityFrameworkCore.Sqlite

## Data project

### Folders
I've created two folders in the data project
- Models - that's where the data model classes will be
- Migrations - that's where our db migrations will be
- Helpers - that's where the helper classes for making this usable on multiple platforms will be

### Models
Let's say we want to have a database of fish. Here's a class called `Fish`.

```csharp
using System.ComponentModel.DataAnnotations;

namespace Data.Models
{
    public class Fish
    {
        [Key]
        public string Name { get; set; }
        public int Length { get; set; }
        public bool EatsHumans { get; set; }
    }
}
```
I'm going to using this class as the example data model.

### Database context basics
Now let's create a database context class. I'll call it `OceanDbContext` since we'll be storing fish in it.

```csharp
using Data.Models;
using Microsoft.EntityFrameworkCore;

namespace Data
{
    public class OceanDbContext : DbContext
    {
        public DbSet<Fish> Fishes { get; set; }

        protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
        {
            //important stuff will be here
        }
    }
}
```

This is just the basic version, the important things will go in the overriden `OnConfiguring` method.

For now we've:
- created a `DbSet` of `Fish` called `Fishes`, that's an eqivalent of a database table.
- overriden the `OnConfiguring` method, leaving it empty for now

### A way to use different database providers for different platforms

Since we want to use different databases (PostgreSQL, SQLite) for the two apps, we'll need a way to make the Data project take in some configuration (wanted database provider, connection string).

For that I've created following items in the "Helpers" folder:

#### DbProvider.cs
This is an enum that will help us determine what database provider to use
```csharp
namespace Data.Helpers
{
    public enum DbProvider
    {
        Sqlite,
        PostgreSql
    }
}
```

#### IConnectionString.cs
An abstraction of a connection string, which differs from one provider to another
```csharp
namespace Data.Helpers
{
    public interface IConnectionString
    {
        string Construct();
        DbProvider GetProvider();
    }
}
```

#### PostgreSqlConnectionString.cs
A connection string definition for PostgreSQL
```csharp
namespace Data.Helpers
{
    public class PostgreSqlConnectionString : IConnectionString
    {
        public string Host { get; set; }
        public string Port { get; set; }
        public string Database { get; set; }
        public string Username { get; set; }
        public string Password { get; set; }

        public string Construct()
        {
            return $"host={Host};port={Port};database={Database};user id={Username};password={Password};";
        }

        public DbProvider GetProvider()
        {
            return DbProvider.PostgreSql;
        }
    }
}
```

#### SqliteConnectionString.cs
A connection string definition for SQLite
```csharp
namespace Data.Helpers
{
    public class SqliteConnectionString : IConnectionString
    {
        public string DbFilePath { get; set; }

        public string Construct()
        {
            return $"Data Source={DbFilePath};";
        }

        public DbProvider GetProvider()
        {
            return DbProvider.Sqlite;
        }
    }
}
```

#### DbHelper.cs
This static class is going to be used to configure the connection from the other projects (DesktopApp, MobileApp)
```csharp
using System;

namespace Data.Helpers
{
    public static class DbHelper
    {
        private static bool _connectionStringInitialized;
        private static IConnectionString _connectionString = null;

        public static OceanDbContext GetContext()
        {
            return new OceanDbContext();
        }

        public static void SetConnectionString(IConnectionString cs)
        {
            _connectionStringInitialized = true;
            _connectionString = cs;
        }

        public static IConnectionString GetConnectionString()
        {
            if (_connectionStringInitialized)
            {
                return _connectionString;
            }

            throw new Exception($"{nameof(_connectionString)} hasn't been initialized. Make sure to call {nameof(SetConnectionString)} before using the {nameof(GetConnectionString)}");
        }
    }
}
```

### Back to the database context

Now that we've defined a way to abstract and configure the connection from each platform, we can go back to the database context and do following:

#### Add these usings
`using Data.Helpers;`
`using System`

#### Edit the `OnConfiguring` method

Now let's edit the `OnConfiguring` method to acutally use the configured provider and connection string.

The "Migration code" that is commented out will be used only for creating migrations, it won't be used for anything else.

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    #region Migration code

    //DbHelper.SetConnectionString(
    //   new SqliteConnectionString()
    //   {
    //       DbFilePath = "nothing.db"
    //   }
    //);
    //optionsBuilder.UseSqlite(DbHelper.GetConnectionString().Construct());

    //return;

    #endregion

    if (!optionsBuilder.IsConfigured)
    {
        var cs = DbHelper.GetConnectionString();

        switch (cs.GetProvider())
        {
            case DbProvider.Sqlite:
                optionsBuilder.UseSqlite(cs.Construct());
                break;
            case DbProvider.PostgreSql:
                optionsBuilder.UseNpgsql(cs.Construct());
                break;
            default:
                throw new NullReferenceException($"Invalid database provider > {cs.GetProvider()}");
        }
    }
}
```

## MobileApp project

Now let's configure the database from the mobile project.

Let's first add a reference to the Data project.

### Database file path

We'll need a way to get some valid database file path for each platform (Android, iOS). Let's do that.

#### Create an interface called `IDbPathFinder` in the MobileApp project

```csharp
namespace MobileApp
{
    public interface IDbPathFinder
    {
        string GetFullPath(string name);
    }
}
```

#### Create an Android implementation of that interface in the MobileApp.Android project

```csharp
using MobileApp.Droid;
using System;
using System.IO;
using Xamarin.Forms;

[assembly: Dependency(typeof(DbPathFinder))]
namespace MobileApp.Droid
{
    public class DbPathFinder : IDbPathFinder
    {
        public string GetFullPath(string name)
        {
            return Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.Personal), name);
        }
    }
}
```

#### Create an iOS implementation of that interface in the MobileApp.iOS project

```csharp
using MobileApp.iOS;
using System;
using System.IO;
using Xamarin.Forms;

[assembly: Dependency(typeof(DbPathFinder))]
namespace MobileApp.iOS
{
    public class DbPathFinder : IDbPathFinder
    {
        public string GetFullPath(string name)
        {
            return Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments), "..", "Library", name);
        }
    }
}
```

### Configuring on start

In order to configure the database connection before we need to use, we'll make all the call in the `App.xaml.cs` file.

We'll add a method called `ConfigureDatabase`, that will contain the configration and then call that method from the `OnStart` method.

The App.xaml.cs file should now look like this:

```csharp
using Data.Helpers;
using Xamarin.Forms;

namespace MobileApp
{
    public partial class App : Application
    {
        public App()
        {
            InitializeComponent();

            MainPage = new MainPage();
        }

        protected override void OnStart()
        {
            ConfigureDatabase();
        }

        protected override void OnSleep()
        {
        }

        protected override void OnResume()
        {
        }

        private void ConfigureDatabase()
        {
            var dbPath = DependencyService.Get<IDbPathFinder>().GetFullPath("ocean.db");
            DbHelper.SetConnectionString(new SqliteConnectionString { DbFilePath = dbPath });
        }
    }
}
```

## DesktopApp project

Now let's configure the database from the desktop project.

Let's first add a reference to the Data project.

In order to configure the database connection before we need to use, we'll make all the call in the `App.xaml.cs` file.

Add a method called `ConfigureDatabase`, that will contain the configration.

Then override the `OnStartup` method and call the `ConfigureDatabase` from it.

The App.xaml.cs file should now look like this:

```csharp
using Data.Helpers;
using System.Windows;

namespace DesktopApp
{
    public partial class App : Application
    {
        protected override void OnStartup(StartupEventArgs e)
        {
            ConfigureDatabase();
        }

        private void ConfigureDatabase()
        {
            DbHelper.SetConnectionString(new PostgreSqlConnectionString
            {
                Host = "localhost",
                Port = "5434",
                Database = "ocean",
                Username = "postgres",
                Password = "postgres"
            });
        }
    }
}
```

Note that in this example I'm hard coding the connection string, which is a terrible idea. Don't do that!

## Almost ready, let's migrate

Now before we use the app, there's one last step - migrations.

In order to create an initial (or any other) migration, we'll to do following:

### Uncomment the migration code in the `OceanDbContext` class

```csharp
#region Migration code

DbHelper.SetConnectionString(
  new SqliteConnectionString()
  {
      DbFilePath = "nothing.db"
  }
);
optionsBuilder.UseSqlite(DbHelper.GetConnectionString().Construct());

return;

#endregion
```

### Change the Data project's target framework to multiple targets in order to run migrations with it

Go to the Data.csproj and change this:
```
<TargetFramework>netstandard2.0</TargetFramework>
```

to this

```
<TargetFrameworks>netcoreapp2.0;netstandard2.0</TargetFrameworks>
```

The reason we do that is to be able to run commands using the Data project in the Package Manager Console (which needs a runnable project type to function).

### Migrate

Open the Package Manager Console (Tools -> Nuget Package Manager -> Package Manager Console in Visual Studio 2019)

Make sure to have the Data project selected as the default project.

<img src="images/2020_08_12_efCoreMigrationsPmcConfig.jpg">

And run the following command to create a migration:
`Add-Migration Migration001 -Context OceanDbContext -OutputDir Migrations`

If successful, you should now have new files in the Migrations folder.
- SomeLongNumber_Migration001.cs
- OceanDbContextModelSnapshot.cs

### Cleanup

Now that the migration is done, let's comment out the migration code in `OceanDbContext`

```csharp
#region Migration code

// DbHelper.SetConnectionString(
//   new SqliteConnectionString()
//   {
//       DbFilePath = "nothing.db"
//   }
// );
// optionsBuilder.UseSqlite(DbHelper.GetConnectionString().Construct());

// return;

#endregion
```

We're now ready to use the thing!

## Let's try it

### Basic view model

In order to try this, I'll create a simple class called `MainViewModel` that will serve as a view model for both the DesktopApp and the MobileApp.

I've referenced the Data project and created the following class:
```csharp
using Data.Helpers;
using Data.Models;
using Microsoft.EntityFrameworkCore;
using System;
using System.Collections.ObjectModel;
using System.Diagnostics;
using System.Linq;

namespace Logic
{
    public class MainViewModel
    {
        public ObservableCollection<Fish> Fishes { get; set; } = new ObservableCollection<Fish>();

        public MainViewModel()
        {
            InitializeDatabase();
            LoadFishes();
        }

        public void AddRandomFish()
        {
            var randy = new Random();

            using (var db = DbHelper.GetContext())
            {
                db.Fishes.Add(new Fish
                {
                    Name = new[] { "Shark", "Blue whale", "Nemo" } [randy.Next(3)],
                    Length = randy.Next(1,15),
                    EatsHumans = randy.Next(2) == 1
                });
                db.SaveChanges();
            }
        }

        public void LoadFishes()
        {
            using (var db = DbHelper.GetContext())
            {
                var fishes = db.Fishes.ToList();
                foreach (var fish in fishes)
                {
                    Fishes.Add(fish);
                }
            }
        }

        public void InitializeDatabase()
        {
            try
            {
                using (var db = DbHelper.GetContext())
                {
                    db.Database.Migrate();
                }
            }
            catch (Exception ex)
            {
                Debug.WriteLine("Oops! The database could not be initialized. " + ex.ToString());
            }
        }
    }
}
```

### Desktop app

I've added a reference to the Logic project and edited the MainWindow to look like this:

MainWindow.xaml.cs
```csharp
using Logic;
using System.Windows;

namespace DesktopApp
{
    public partial class MainWindow : Window
    {
        private MainViewModel _model;
        public MainWindow()
        {
            InitializeComponent();
            DataContext = _model = new MainViewModel();
        }

        private void BtnAddFish_Click(object sender, RoutedEventArgs e)
        {
            _model.AddRandomFish();
        }
    }
}
```

MainWindow.xaml
```xaml
<Window
    x:Class="DesktopApp.MainWindow"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
    xmlns:local="clr-namespace:DesktopApp"
    xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
    Title="Fish list"
    Width="800"
    Height="450"
    WindowStartupLocation="CenterScreen"
    mc:Ignorable="d">
    <Grid>
        <ListView ItemsSource="{Binding Fishes}" />
        <Button Click="BtnAddFish_Click" Content="Add random fish" HorizontalAlignment="Right" VerticalAlignment="Bottom" Margin="10" Padding="10"/>
    </Grid>
</Window>
```




