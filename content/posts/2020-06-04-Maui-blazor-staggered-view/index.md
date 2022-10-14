---
title: "Maui Blazor Staggered View"
date: 2021-06-04T13:02:12Z
categories: [development,maui,blazor]
tags: [masonry,staggered view,html,javascript,css,maui,blazor]
draft: false
---

Do you remember my staggered view solution for Xamarin Forms? If not, I blogged about it here: [https://blog.depechie.com/posts/2021-05-26-xf-staggered-view/](https://blog.depechie.com/posts/2021-05-26-xf-staggered-view/).

But since we have preview 4 of .Net 6 we are able to do something even better! We can now create full HTML apps with **.Net MAUI Blazor** ( more details here: [https://devblogs.microsoft.com/aspnet/asp-net-core-updates-in-net-6-preview-4/#net-maui-blazor-apps](https://devblogs.microsoft.com/aspnet/asp-net-core-updates-in-net-6-preview-4/#net-maui-blazor-apps)).

So why is this good you could ask yourself... well several reasons, but the major one, is the fact that our staggered view solution relies upon HTML and JavaScript. So having a development environment with the same target would suit us more then mixing XAML and HTML.  
Secondly we will now have a fully working app on iOS and Android but also a Mac OS app too! Ow and let's not forget we also get a Windows desktop version with WinUI 3.  
Without any hassle, this is just mind blowing. And thanks to CSS media queries, we can tailor the view of our staggered view automatically.  
In these desktop cases, when we go over 768 pixels in width, we change from a 2 column to a 3 column masonry view. You can see the result below in the video.

I can't stress this enough that having 1 dev environment for all possible targets is a big plus with .Net MAUI and now with all the web power thanks to Blazor we can create nice solutions.

The principle of the code is the same as the Xamarin Forms version, our **Index.razor** now contains the setup of the image list.

```
@page "/"
@inherits IndexViewModel
@implements IDisposable

<div class="grid are-images-unloaded">
    <div class='grid__col-sizer'></div>
    @foreach (var image in Images)
    {
        <div class="grid__item"><img src="@image"></div>
    }
</div>
```

And the companion ViewModel handles the interaction with **JavaScript** and image loading.

```
using Microsoft.AspNetCore.Components;
using Microsoft.JSInterop;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using masonrymauiblazor.Data;

namespace masonrymauiblazor.ViewModels
{
    public class IndexViewModel : ComponentBase
    {
        protected string result;
        private DotNetObjectReference<IndexViewModel> _objRef;
        protected bool IsScrollTrackingEnabled { get; set; } = false;
        protected List<string> Images { get; set;}

        [Inject]
        protected IJSRuntime JS { get; set; }

        [Inject]
        protected DataService DataService { get; set; }

        public void Dispose()
        {
            _objRef?.Dispose();
        }

        [JSInvokable]
        public bool IsAtWindowBottom(double contentScrollTop, double contentHeight, double containerHeight)
        {
            bool retVal = (contentScrollTop + contentHeight) >= containerHeight;
            return retVal;
        }

        [JSInvokable]
        public bool IsNearWindowBottom(double contentScrollTop, double contentHeight, double containerHeight)
        {
            bool retVal = (contentScrollTop + contentHeight) > (containerHeight - 100);
            return retVal;
        }

        [JSInvokable]
        public async Task LoadMoreImages(int pageIndex)
        {
            var newImages = DataService.GetPhotos(Artists.Depechie, pageIndex);

            //TODO: Glenn - This list of new images should be added to the Blazor list with Images.AddRange(newImages)
            //TODO: Glenn - But this will have the bad positioning effect, we need to trigger the imagesloaded function
            //TODO: Glenn - Also this should be doable according to https://stackoverflow.com/questions/64593058/blazor-force-full-render-instead-of-differential-render
            //TODO: Glenn - But I was not able to get it working correctly, hence the roundtrip and pushing the string to JavaScript
            if (newImages != null && newImages.Any())
                _ = await JS.InvokeAsync<string>("blazorExtensions.triggerMasonry", string.Join("#", newImages));
        }

        public async Task ToggleDotNetTrackScroll()
        {
            _ = await JS.InvokeAsync<string>("blazorExtensions.toggleTrackScroll", _objRef);
        }

        [JSInvokable]
        public bool ToggleTrackScroll() { IsScrollTrackingEnabled = !IsScrollTrackingEnabled; return IsScrollTrackingEnabled; }

        protected override async Task OnInitializedAsync()
        {
            Images = DataService.GetPhotos(Artists.Depechie, 0);
        }

        protected async override Task OnAfterRenderAsync(bool firstRender)
        {
            if (firstRender)
            {
                _objRef = DotNetObjectReference.Create(this);
                await JS.InvokeVoidAsync("blazorExtensions.toggleTrackScroll", _objRef);
                StateHasChanged();

                await JS.InvokeVoidAsync("blazorExtensions.initMasonry", _objRef);
            }
        }
    }
}
```

You can see there is one small issue with the loading more images. I had to go with the same Xamarin Forms solution and let JavaScript add the newer images, due to the imagesloaded function of Masonry. But I'm sure someone with more **Blazor** knowledge than me could fix this and have the image adding functionality purely in the ViewModel.

All the code is up on my GitHub here [https://github.com/Depechie/Masonry/tree/main/src/mauiblazor](https://github.com/Depechie/Masonry/tree/main/src/mauiblazor).

Major thanks to [Paul Schroeder](https://twitter.com/PaulBSchroeder) and [Christophe Peugnet](https://twitter.com/tossnet1) for their guidance on some of the Blazor code, without that I would have been lost ;)

 ![alt text](images/masonrymauiblazor.gif)