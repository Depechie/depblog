---
title: "Xamarin Forms – Set entry clear icon color on iOS"
date: 2021-03-28T17:32:45Z
categories: [development,xamarin forms]
tags: [iOS,effects,entry,xamarin forms]
draft: false
---

Here is a small piece of code that you can use as a Xamarin Forms effect to hook into your entry.
It allows changing the color of the clear icon that is available inside the native entry control of iOS ( when enabled ).

``` 
if (Control is UITextField textField)
{
    //Enable the clear icon
    textField.ClearButtonMode = UITextFieldViewMode.WhileEditing;
    //Get hold of the clear icon inside the text field
    UIButton clearButton = textField.ValueForKey(new Foundation.NSString("clearButton")) as UIButton;
    //Set the desired color
    clearButton.TintColor = UIColor.Red;
    UIImage clearImage = UIImage.GetSystemImage("xmark.circle.fill");
    //Override the regular image with the colored one
    clearButton.SetImage(clearImage.ImageWithRenderingMode(UIImageRenderingMode.AlwaysTemplate), UIControlState.Normal);
}
```

With this platform specific code, we can get a hold of the UIButton inside the UITextField and set a color. Than again pasting it into the UIButton.

{{< resize-image src="images/entryclear.png" alt="entry clear icon color screenshot">}}

As always a full example can be found here [https://github.com/Depechie/EntryClearIconColor](https://github.com/Depechie/EntryClearIconColor).