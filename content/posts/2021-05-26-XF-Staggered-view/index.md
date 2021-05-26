---
title: "Xamarin Forms staggered view - Masonry"
date: 2021-05-26T12:01:17Z
categories: [development,xamarin forms]
tags: [masonry,staggered view,html,javascript,css,xamarin forms]
draft: false
---

### Some story background

When developing mobile apps, you'll often find different design trends that look great and sometimes you even want to adopt some of those trends in your own apps.

For me one of those iconic design trends that I always wanted to try out in an app, was the staggered view.
I you look at any kind of portfolio website template or go search for photo related mobile app designs, the staggered view will for sure be featured multiple times!

The overall look and feel is shown in the following image:

{{< resize-image src="images/masonry-layout.png" alt="masonry layout">}}

In general it presents all items on the page in a **staggered** layout, this has many different names depending in what development/design environment you are in.
Some examples are **waterfall, masonry, flexbox, flow layout, cascading grid** and maybe many others that I do not know about.

But the overall idea is always the same, flow items from left to right and top to bottom, but if needed give each items it's own width and/or height.
This way not all items will be presented in the same fixed width/height square or rectangular shape. Resulting in a more natural pleasing view.

### The road to success

Of course I wanted to try this view out in a mobile app and that means I needed to find a way to get it rendered through Xamarin Forms.  
But this somehow seemed more difficult than first assessed...

My first stab, was trying to use one of the latest views we received in Xamarin Forms, the [flex layout](https://docs.microsoft.com/en-us/xamarin/xamarin-forms/user-interface/layouts/flex-layout?WT.mc_id=DT-MVP-5000533). Even though the item positioning was correct, the sizing was not dynamic, so each item was contained in a same fixed width/height area.

After that, together with my friend and also Xamarin Forms developer, [Konrad Müller](https://twitter.com/konmue), we tried to forge custom renderers for the Xamarin Forms [collectionview layout](https://docs.microsoft.com/en-us/xamarin/xamarin-forms/user-interface/collectionview/layout?WT.mc_id=DT-MVP-5000533).  
Konrad had already done some work on this while helping out with the [Xappy project](https://github.com/davidortinau/Xappy) by [David Ortinau](https://twitter.com/davidortinau), the original blog posts page had a design spec for this as you can see here [https://github.com/davidortinau/Xappy/issues/6](https://github.com/davidortinau/Xappy/issues/6). He already had the **Android** version up and running and I had an implementation example in SwiftUI that we wanted to try to port to the collectionview. But after an evening of trial and error, the collectionview was to much to handle.  
The way it is implemented internally in Xamarin Forms makes it difficult to adapt.

So I was back to square one and had no idea on how to tackle it correctly. So this got me thinking, we have so many nice implementation of this layout ready available through use of **HTML** and **CSS**, couldn't we leverage this power in our Xamarin Forms app too?
Well it seems we can, with some tweaks and adjustments. Get ready for a lengthy post!

### Coding the view

Let me first show you how our app will look like when all code is correctly implemented:

{{< resize-image src="images/masonry.png" alt="masonry app layout screenshot">}}

I first drafted up a list of must have features that needed to be addressed while creating this layout
- Items should be presented in a staggered layout
- Items should be loaded through C# code and preferable making use of MVVM patterns
- The layout should have infinite scroll capabilities

To help out with the layout, I took a look at some **JavaScript** libraries and decided to use **Masonry js** which is available here [https://masonry.desandro.com/](https://masonry.desandro.com/). It has great documentation is open source and is already used by many websites.
Also one of the key features it has, is the ability to handle dynamic addition of new items to the layout! This will be needed for the infinite scroll option.  
You as a user of this js lib, need to provide at least 3 elements; some HTML with the items that need to be placed in the layout, some JavaScript to initialize Masonry and set some options, and if you want, some CSS to adjust the styling of the layout.

First the HTML.

```
<!DOCTYPE html>
<html>
    <head>
        <meta name='viewport' content='width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=0' />
        <style type='text/css'>##CSS##</style>
    </head>
    <body>
        <div class='grid are-images-unloaded'>
            <div class='grid__col-sizer'></div>
            ##ITEMS##
        </div>

        <script src='https://unpkg.com/masonry-layout@4/dist/masonry.pkgd.js'></script>
        <script src='https://unpkg.com/imagesloaded@4/imagesloaded.pkgd.js'></script>
        <script id='render-js'>##INLINEJS##</script>
    </body>    
</html>
```

This is a very basic HTML body with only a few key elements needed to get things working.
At the bottom you'll notice we need to make a reference to 2 external JavaScript libraries, **Masonry** and **ImagesLoaded**. These 2 will do the heavy lifting for us.  
At the middle we have a **div** that uses the CSS class **grid**, this is the element Masonry will look for to load the items in the view.
Inside that grid we will be placing our elements, to be able to do this from Xamarin Forms with C#, string replacing will be performed, hence why we added the **##ITEMS##** tag.  
There are also 2 other tags, one for adding custom CSS and one for adding extra inline JavaScript. ( more info on this later )

Adding the first items to render inside the grid, is done with C# at the start of our application.

```
private string InitHTMLSource()
{
    IEnumerable<string> artistPictures = _dataService.GetPhotos(Artists.Depechie, 0);
    var body = MasonryHelper.GenerateHTMLSource();
    var items = MasonryHelper.GenerateItemSource(artistPictures);

    return MasonryHelper.InsertItems(body, items);
}
```

> **Spoiler alert: I'm NOT the actual photographer of the demo pictures!**

We use a data service to retrieve an amount of pictures for a given artist, this results in a List of strings that are all URL's pointing to some pictures online. We then use regular find and replace to inject that list into the above provided HTML.

```
public static string GenerateItemSource(IEnumerable<string> items)
{
    StringBuilder itemSource = new StringBuilder();
    foreach (var item in items)
    {
        itemSource.AppendLine(_gridItem.Replace(SEARCHKEY_ITEMSOURCE, item));
    }

    return itemSource.ToString();
}

public static string InsertItems(string htmlSource, string itemSource)
{
    return htmlSource.Replace(SEARCHKEY_ITEMS, itemSource);
}
```

With that in place, we now have our full HTML source that we want to load into the WebView control we are using in our Xamarin Forms page.

```
protected override void OnAppearing()
{
    base.OnAppearing();

    webViewElement.RegisterAction(InvokeCSharpFromJS);
    webViewElement.Source = new HtmlWebViewSource()
    {
        Html = InitHTMLSource()
    };
}
```

You'll notice some extra code here as well, the **RegisterAction**. This is needed for the JavaScript bridge, but more on this later.
First let me show you the page XAML itself.

```
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://xamarin.com/schemas/2014/forms"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             xmlns:controls="clr-namespace:Masonry.Controls"
             x:Class="Masonry.MainPage">

    <Grid RowDefinitions="auto, auto, *">
        <Grid.Margin>
            <OnPlatform x:TypeArguments="Thickness">
                <On Platform="iOS" Value="0,30,0,0" />
                <On Platform="Android" Value="0,10,0,0" />
            </OnPlatform>
        </Grid.Margin>

        <StackLayout Orientation="Horizontal"
                     Margin="18,10,18,5"
                     HorizontalOptions="FillAndExpand"
                     Grid.Row="1">
            <Label Text="Porfolio"
                   FontFamily="JosefinRegular"
                   FontSize="20"
                   TextColor="#4d4d4d"
                   HorizontalOptions="StartAndExpand" VerticalOptions="End"/>

            <Label Text="Glenn Versweyveld"
                   FontFamily="JosefinRegular"
                   FontSize="16"
                   HorizontalOptions="EndAndExpand" VerticalOptions="End" />
        </StackLayout>

        <controls:HybridWebView x:Name="webViewElement"
                                Grid.Row="2" />
    </Grid>
</ContentPage>
```

Big thing to notice here is the fact that we are using a custom WebView control called HybridWebView. This is needed to enable the JavaScript interaction from the WebView to Xamarin Forms and the other way around.  
I followed the great guide by [Udara Alwis](https://github.com/UdaraAlwis), he explained everything in his article about creating a bi-directional interop with a WebView here [https://theconfuzedsourcecode.wordpress.com/2020/01/19/building-a-bi-directional-interop-bridge-with-webview-in-xamarin-forms/](https://theconfuzedsourcecode.wordpress.com/2020/01/19/building-a-bi-directional-interop-bridge-with-webview-in-xamarin-forms/).  
Although Microsoft also has posted an intro article about the same technique on their docs site here [https://docs.microsoft.com/en-us/xamarin/xamarin-forms/app-fundamentals/custom-renderer/hybridwebview](https://docs.microsoft.com/en-us/xamarin/xamarin-forms/app-fundamentals/custom-renderer/hybridwebview?WT.mc_id=DT-MVP-5000533).

In short, you need to create a custom WebView renderer on iOS and Android to enable the cross invoking functionality. Meaning you'll be able to call a C# function from within JavaScript but also call a JavaScript function from your C# code.
In my example I called them InvokeCSharpFromJS and InvokeJSFromCSharp.  
It is this InvokeCSharpFromJS method that needs to be injected into the custom WebView through the **RegisterAction** method that you saw earlier. Once that is done, you can call it from you JavaScript.

In our case we want to tell the C# code when new images need to be loaded in. Remember the first images will already be presented, thanks to our HTML we generated and set as source of the WebView. But when the user scrolls down, we need to dynamically load extra pictures.  
To get this going, we listen for the Scroll event in our JavaScript and use the bride to signal back to our C# code that new pictures are requested.

The complete inline JavaScript looks like this.

```
let pageIndex = 0;
let loadNext = false;

window.addEventListener('scroll', () => {
    const { scrollTop, scrollHeight, clientHeight } = document.documentElement;
    if(clientHeight + scrollTop >= scrollHeight - 100 && loadNext) {
        loadNext = false;
        invokeCSharpAction(pageIndex);
        pageIndex++;
	}
});

var grid = document.querySelector('.grid');
var msnry = new Masonry( grid, {
    columnWidth: '.grid__col-sizer',
    itemSelector: 'none',
    percentPosition: true,
    stagger: 30,
    visibleStyle: { transform: 'translateY(0)', opacity: 1 },
    hiddenStyle: { transform: 'translateY(100px)', opacity: 0 },  
});

imagesLoaded( grid, function() {
    grid.classList.remove('are-images-unloaded');
    msnry.options.itemSelector = '.grid__item';
    let items = grid.querySelectorAll('.grid__item');
    loadNext = true;
    msnry.appended(items);
});

function invokeJSFromCSharp(data) {
    var images = data.split('#');
    var elems = [];
    var fragment = document.createDocumentFragment();
    var i;
    for (i = 0; i < images.length; i++) {
        var elem = getItemElement(images[i]);
        fragment.appendChild(elem);
        elems.push(elem);
    }

    grid.appendChild(fragment);
    var imageLoad = imagesLoaded(grid);
    imageLoad.on( 'progress', function() {  msnry.layout(); });
    loadNext = true;
    msnry.appended(elems);
}

function getItemElement(content) {
  var elem = document.createElement('div');
  elem.className = 'grid__item';
  elem.innerHTML = '<img class=""image-grid__item"" src=""' + content + '"">';
  return elem;
}
```

There is small part for the Masonry grid initialization and options, the new Masonry call. At the top you'll notice the **window.addEventListener**, here the magic callback to C# will happen. As soon as the user has scrolled down and we are roughly 100px from the bottom, we request more images and pass in a **pageIndex** representing how many times the user has reached the bottom of our WebView.  
That way we enable the infinite scroll feature, as long as we are able to request a next set of images, the user will be able to keep scrolling down. The actual C# code to handle that request is show below.

```
private void InvokeCSharpFromJS(string data)
{
    if (!string.IsNullOrWhiteSpace(data) && int.Parse(data) is int pageIndex)
    {
        IEnumerable<string> artistPictures = _dataService.GetPhotos(Artists.Depechie, ++pageIndex);
        if (artistPictures.Any())
        {
            var items = string.Join("#", artistPictures);
            Device.BeginInvokeOnMainThread(async () =>
            {
                string result = await webViewElement.EvaluateJavaScriptAsync($"invokeJSFromCSharp('{items}')");
            });
        }
    }
}
```

When we get a result back from the DataService with a potential new set of images for the requested pageIndex and artist, we will call back into JavaScript and pass that result list as one joined string. You can see in the inline JavaScript function invokeJSFromCSharp, that we unpack that string and use the Masonry **appended** function to dynamically add those new images.

### Final result

With that everything is set and the final result looks like this ( notice the nice fade in animation provided by Masonry ):

 ![alt text](images/masonry.gif)

 Overall I think this solution is great, because while using the WebView and HTML you'll notice no design difference between iOS and Android, what is a big plus!

 > **Warning: don't try this on the Android simulator**

 Somehow using this on the Android simulator with a debug build is very laggy. But deploying a release build to an actual device works like a charm!!

 As always you can find a fully working example on GitHub here [https://github.com/Depechie/Masonry](https://github.com/Depechie/Masonry).

 Enjoy.