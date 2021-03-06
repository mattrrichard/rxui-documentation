# Basic Property Binding 

A core part of being able to use the MVVM pattern is the very specific
relationship between the ViewModel and View - that is, the View is connected
in a one-way dependent manner to the ViewModel via *bindings*. 

ReactiveUI provides its own implementation of this concept, which has a number
of advantages compared to platform-specific implementations such as XAML-based
bindings.

* Bindings work on **all platforms** and operate the same.
* Bindings are written via Expressions. This means that they are strongly typed, 
  and renaming a ViewModel property, or a control in the UI layout without 
  updating the binding, the build will fail.
* Controlling how types bind to properties is flexible and can be customized.

### Getting Started

In order to use bindings in the View, you must first implement
`IViewFor<TViewModel>` on your View. Depending on the platform, you must
implement it differently:

* **iOS** - change your base class to one of the Reactive UIKit classes (i.e.
  `ReactiveUIViewController`) and implement `ViewModel` using
  `RaiseAndSetIfChanged`, *or* implement `INotifyPropertyChanged` on your View and
  ensure that ViewModel signals changes.

* **Android:** - change your base class to one of the Reactive Activity /
  Fragment classes (i.e. `ReactiveActivity<T>`), *or* implement
  `IViewFor<T>` on your View and ensure that your ViewModel signals
  changes.

* **Xaml-based:** - Implement `IViewFor<T>` by hand and ensure that ViewModel
  is a `DependencyProperty`.
  
For a detailed overview of the bindings on each platform, see the "Binding" section.

### Types of Bindings

Once you implement `IViewFor<T>`, binding methods are now available as
extension methods on your class. Like many other things in ReactiveUI, you
should only set up bindings in a constructor or setup method when the view is
created.

* **OneWayBind:** - Sets up a one-way binding from a property on the ViewModel
  to the View.

```cs
var disp = this.OneWayBind(ViewModel, x => x.Name, x => x.Name.Text);
```

* **Bind:** - Sets up a two-way binding between a property on the ViewModel to
  the View.

```cs
this.Bind(ViewModel, x => x.Name, x => x.Name.Text);
```

* **BindCommand:** - Bind an `ICommand` to a control, or to a specific event
  on that control (how this is implemented depends on the UI framework):

```cs
// Bind the OK command to the button. 
// This uses the default "Click" event on XAML-based platforms and equivalent 
// events on Android and iOS.
this.BindCommand(ViewModel, x => x.OkCommand, x => x.OkButton);

// Bind the OK command to when the user presses a key
this.BindCommand(ViewModel, x => x.OkCommand, x => x.RootView, "KeyUp");
```

`OneWayBind`, `Bind`, and `BindCommand` return an `IDisposable`.  
In general, you shouldn't care about this return value, except if you want to 
break a binding manually, or you're on a XAML-based platform, where bindings can 
leak memory (Look in the "Binding" section under "XAML" for more on this).

### Converting between types

Direct bindings between properties are convenient, but often the two types are
not assignable to each other. For example, binding an "Age" property to a
TextBox would normally fail, because the TextBox expects a string value. Instead,
ReactiveUI has an extensible system for coercing between types.

For simple one-way bindings, you can use the conversion parameter in the 
`OneWayBind` method. This parameter is a `Func<TIn, TOut>`, so if the ViewModel property
is of type `int` and the view property is of type `string`, it's a `Func<int, string>`
which means that you don't have to write custom converter classes, like in 
standard WPF bindings, and can just write a simple, statically typed, conversion function.

```cs

// Note: Age is an integer, Text is a string
this.OneWayBind(ViewModel, x => x.Age, x => x.Name.Text, x => x.ToString()); // In the last parameter, we .ToString() the integer
```

For type conversion on two-way bindings, see the details about Binding Type Converters 
in the "Customization" section for more information about how to extend property type conversion.

### Choosing when to update the source

By default, the source of a binding will be updated when the target changes,
which is equivalent to setting `UpdateSourceTrigger = PropertyChanged` on a WPF
binding. Sometimes it is desirable to have more fine-grained control over when
the source will be updated (for example, the binding updating will trigger
some expensive work which isn't necessary on every keystroke).

One common requirement is to update the source when a user interface control
loses keyboard (or logical) focus - WPF even provides `UpdateSourceTrigger = 
LostFocus` for this.

ReactiveUI bindings allow this by providing an `IObservable` as a parameter to
the binding, which disables the default behavior and causes the source to
update whenever the observable fires. The parameter (`SignalViewUpdate`) can
be of any `IObservable<object>`, meaning that any event on a user interface 
control is capable of causing the binding to update.

For example, to have a binding update when a control loses keyboard focus, the
following binding setup can be used:

```cs
// Note: We're using the ReactiveUI.Events NuGet package here, which wraps 
// traditional .NET events on UI controls into IObservables and exposes them 
// via the Events() extension method
this.Bind(ViewModel, vm => vm.SomeProperty, v => v.SomeTextBox, SomeTextBox.Events().LostKeyboardFocus);
```

### "Hack" bindings and BindTo

Should you find that direct one and two-way bindings aren't enough to get the
job done (or should you want View => ViewModel bindings), a flexible, Rx-based
approach is also available, via combining `WhenAnyValue` with the `BindTo`
operator, which allows you to bind an arbitrary `IObservable` to a property on
an object.

For example, here is a simple example of binding a ListBox's `SelectedItem` to
a ViewModel:

```cs
public MainView()
{
    // Bind the View's SelectedItem to the ViewModel
    this.WhenAnyValue(x => x.SomeList.SelectedItem)
        .BindTo(this, x => x.ViewModel.SelectedItem);

    // Bind ViewModel's IsSelected via SelectedItem. Note that this
    // is only for illustrative purposes, it'd be better to bind this
    // at the ViewModel layer (i.e. WhenAnyValue + ToProperty)
    this.WhenAnyValue(x => x.SomeList.SelectedItem)
        .Select(x => x != null)
        .BindTo(this, x => x.ViewModel.IsSelected);
}
```

`BindTo` applies the same binding hooks and type conversion that other property
binding methods do, so the types don't necessarily have to match between the
source and the target property.

While you could certainly build complex bindings (even ones between two view
models!), keep in mind that binding logic that you put in the View is
untestable, so keeping the meaningful logic out of bindings is usually a Good
Idea.

### Hack Command Bindings

Similarly to property bindings, you can also add custom Hack bindings for
commands as well. Two methods that are useful for this are `InvokeCommand` and
`WhenAnyObservable`. The former allows you to invoke a command whenever an
Observable signals, and the latter allows you to safely get an Observable from
a ViewModel in a safe way. Here's how they apply to commands:

```cs
//
// This code is all in the View constructor
//

// Invoke a command whenever the Escape key is pressed
this.Events().KeyUpObs
    .Where(x => x.EventArgs.Key == Key.Escape)
    .InvokeCommand(this, x => x.ViewModel.Cancel);

// Subscribe to Cancel, and close the Window when it happens
this.WhenAnyObservable(x => x.ViewModel.Cancel)
    .Subscribe(_ => this.Close());
```
