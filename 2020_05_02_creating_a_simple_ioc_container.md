
# Creating a simple IoC container

Recently, I've been considering using an IoC container in my WPF app. It came to a point where the app was getting pretty huge in terms of number of different views, view models and code in general. So to make my life easier, I wanted to use an IoC container to handle all kinds of services differently for production and testing.

However, all IoC libraries I could find were pretty big, containing thousands of lines of code I wouldn't use and I wanted to keep it simple and small in size. So I've decided to make my own simple IoC container.

## What should it do?

My requirements for the IoC container were:
 - Registering an implementation of an interface
 - Getting an instance of a service by the interface it implements
 
## The code

Here's what I've got so far

```Csharp
public static class CustomIocContainer
{
    private static Dictionary<Type, object> _services = new Dictionary<Type, object>();

    public static T Get<T>() where T : class
    {
        if (!_services.ContainsKey(typeof(T)))
        {
            throw new Exception($"Service of type {typeof(T)} hasn't been registered");
        }

        var result = _services[typeof(T)] as T;

        if(result == null)
        {
            throw new Exception($"The implementation of type {_services[typeof(T)]?.GetType()} couldn't be cast to type {typeof(T)} or is null");
        }

        return result;
    } 

    public static void Register<T, TImpl>(TImpl implementation)
        where T : class
        where TImpl : class, T
    {
        if (_services.ContainsKey(typeof(T)))
        {
            _services[typeof(T)] = implementation;
        }
        else
        {
            _services.Add(typeof(T), implementation);
        }
    }
}
```

## How can I use it?

With the code above, the custom IoC container can be used like this:

```Csharp
//register a service (in the App.xaml.cs for example)
var someImplementation = new SomeImplementation();
CustomIocContainer.Register<ISomeInterface, SomeImplementation>(someImplementation);

//and then get an instance and use it (wherever in the code)
var instance = CustomIocContainer.Get<ISomeInterface>();
instance.DoWork();
```

## That's it

And that is it. It's simple, doesn't do any crazy stuff. It just does what I needed, nothing more. I hope you might find this helpful, too.

Cheers!
