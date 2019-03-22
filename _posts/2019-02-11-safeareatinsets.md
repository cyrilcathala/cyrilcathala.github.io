---
layout: post

title: Best way to use SafeAreaInsets
---

Using a [safe area](https://docs.microsoft.com/en-us/xamarin/xamarin-forms/platform/ios/page-safe-area-layout) is a great way to support the iPhone X* and safely avoid overlaps with the virtual home button or the notch.

In Xamarin.Forms, the easy way is to set a safe area using the platform specific API of a `Page`:

```csharp
using Xamarin.Forms.PlatformConfiguration;
using Xamarin.Forms.PlatformConfiguration.iOSSpecific;

public MyPage() 
{
    InitializeComponent();
    On<iOS>().SetUseSafeArea(true);
}
````

Ok, pretty straightforward. But suppose you want to apply a gradient background underneath the status bar and the virtual home button:

<div style="display:flex; justify-content:space-evenly;">
<figure>
  <img src="/assets/img/2019-02-11-safeareainsets/screen.png" height="400"/>
  <figcaption>What we want.</figcaption>
</figure>

<figure>
  <img src="/assets/img/2019-02-11-safeareainsets/screen-safearea.png" height="400"/>
  <figcaption>What we get with SetUseSafeArea(true).</figcaption>
</figure>
</div>

To achieve that, we'd rather play with the `On<iOS>().SafeAreaInsets()` API. Following the documentation, we should use the `Page.OnAppearing` method:

```xml
<ContentPage>
    <controls:GradientView StartColor="#2196F3"
                           EndColor="#E10AEB" />
    <Grid x:Name="MainContainer">
        <!-- Content -->
    </Grid>
</ContentPage>
```
```csharp
protected override void OnAppearing()
{
    base.OnAppearing();

    MainContainer.Margin = On<Xamarin.Forms.PlatformConfiguration.iOS>().SafeAreaInsets();
}
```

Simple, right? Let's see it in action:

<img src="/assets/img/2019-02-11-safeareainsets/01.gif" height="400"/>

It does work but it is not perfect because the margin gets applied **after** the navigation animation. Let's review a few solutions!

## Option #1: LayoutChanged

`LayoutChanged` is an event raised when bounds have changed. It is usely first raised when the view is mounted, way before the `OnAppearing`.

```csharp
public MyPage() 
{
    InitializeComponent();

    LayoutChanged += MyPage_LayoutChanged;
}

private void MyPage_LayoutChanged(object sender, EventArgs e) 
{
    // We only need the first call, so we unsubscribe immediately
    LayoutChanged -= MyPage_LayoutChanged;

    MainContainer.Margin = On<Xamarin.Forms.PlatformConfiguration.iOS>().SafeAreaInsets();
}
```

Nice try, but not good enough:
- The inset is correctly set for the first Page appearing, but for some reason, the following pages get an empty inset.
- You don't get notified when the value changes (e.g. orientation).

## Option #2: Property changed

We have a way to be notified when a value changes: `PropertyChanged`. Let's use it for the `Page`!

```csharp
public MyPage() 
{
    PropertyChanged += ContentPageBase_PropertyChanged;
}

private void ContentPageBase_PropertyChanged(object sender, PropertyChangedEventArgs e)
{
    if (e.PropertyName == "SafeAreaInsets")
    {
        MainContainer.Margin = On<Xamarin.Forms.PlatformConfiguration.iOS>().SafeAreaInsets();
    }
}
```

If the phone is rotated or a split window is created, the page would be notified. We can then create a `ContentPageBase` and use this technique from every page. 

But what if we need this value from a `ContentView` somewhere else in the app?

## Option #3: Property changed + Dynamic resource

There is an easy way to propagate a value in the app: the [`DynamicResource`](https://docs.microsoft.com/en-us/xamarin/xamarin-forms/user-interface/styles/xaml/dynamic) markup. It is just like an usual `StaticResource`, except we can change the value and any XAML bound to its value is changed.

```csharp
public MyPage() 
{
    PropertyChanged += ContentPageBase_PropertyChanged;
}

private void ContentPageBase_PropertyChanged(object sender, PropertyChangedEventArgs e)
{
    if (e.PropertyName == "SafeAreaInsets")
    {
        var safeAreaInsets = On<Xamarin.Forms.PlatformConfiguration.iOS>().SafeAreaInsets();
        Application.Current.Resources["SafeAreaInsets"] = safeAreaInsets;
    }
}
```

```xml
<Application>
    <Application.Resources>
        <Thickness x:Key="SafeAreaInsets"/>
    </Application.Resources>
</Application>
```

```xml
<Grid>
<controls:GradientView StartColor="{StaticResource PrimaryColor}"
                       EndColor="{StaticResource SecondaryColor}" />
    <Grid Margin="{DynamicResource SafeAreaInsets}">
        <!-- ... -->
    </Grid>
</Grid>
```

Nice & clean! Let's review the result:

<img src="/assets/img/2019-02-11-safeareainsets/02.gif" height="400"/>

## Conclusion

We saw a few tricks along the way. Some can feel quite "hacky", but the right solution seems to be a combination of `PropertyChanged` and `DynamicResource`.