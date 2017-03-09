# Visual Studio 2017

I'm gonna shed light on some of the features, obstacles, tips and tricks in Visual F# Tools that haven't been covered in [the official Microsoft blog](https://blogs.msdn.microsoft.com/dotnet/2017/03/07/announcing-f-4-1-and-the-visual-f-tools-for-visual-studio-2017-2/) 

## Go to All

It's the old "Navigate to" feature from Visual F# Power Tools (VFPT), ported to Roslyn UI. You can launch it with `Ctrl+,` shortcut, as before, or with `Ctrl+T`:

![image](https://cloud.githubusercontent.com/assets/873919/23745235/8b08b0b0-04c8-11e7-9c16-a6ed52d19634.png)

What's new compared to the VFPT (VS 2015) version:

### Filtering symbols by type

* You can add "f" prefix (or launch the feature with `Ctrl+F`) to search for files only:

![image](https://cloud.githubusercontent.com/assets/873919/23745379/f74603a4-04c8-11e7-9444-b49bf0c9d294.png)

* "t" for types:

![image](https://cloud.githubusercontent.com/assets/873919/23745424/2da42afc-04c9-11e7-9538-6407459623a1.png)

and so on.

### Fuzzy search

The search is now fuzzy, which means symbols are found even if you make typos, like this:

![image](https://cloud.githubusercontent.com/assets/873919/23745682/27444c22-04ca-11e7-9c2c-43811d704858.png)

[CamelHumps](https://www.jetbrains.com/help/resharper/2016.3/Navigation_and_Search__CamelHumps.html) is also supported:

![image](https://cloud.githubusercontent.com/assets/873919/23745772/90a92764-04ca-11e7-8723-5807b4dc943b.png)

It works a bit differenctly than JetBrains' one though.

## Guide lines

Hovering over a line shows the scope this line marks:

![image](https://cloud.githubusercontent.com/assets/873919/23745908/1478844a-04cb-11e7-8795-0efa75318faa.png)

You can turn off the guide lines for all languages in the settings dialog:

![image](https://cloud.githubusercontent.com/assets/873919/23746093/c0f4f1a4-04cb-11e7-9c4c-e0f9773c7b40.png)

## Navigation bar

It can be disabled via the settings dialog as well:

![image](https://cloud.githubusercontent.com/assets/873919/23746063/a286b02c-04cb-11e7-9749-54dffba8605b.png)

## Syntax highlighting

By default only types and type parameters are highlighted. You can customize it here:

![image](https://cloud.githubusercontent.com/assets/873919/23746287/71412abe-04cc-11e7-967b-9a6502459db3.png)

and here:

![image](https://cloud.githubusercontent.com/assets/873919/23746313/891342da-04cc-11e7-92da-cb2c38fbc2c1.png)

(use "User Types - Enums" to set color for discreminated unions)

## Issues in RTW

Due to the long Visual Studio stabilization phase, VS 2017 RTW includes a rather outdated Visual F# Tools version, which contains a number of bugs and performance issues:

* Opening any file discards the whole project check results and starts checking from scratch. It means that all the features start working on any just opened file _and all opened file from the same project_ after a pause that's approximately equals to current project compilation time. In short, for large projects it's disaster.

* Any change in any file causes full project diagnostics checking in background. It means you have one of your CPU cores constantly loaded at 100%. For example, after any change in any file in Visual F# Tools solution itself, CPU is loaded for about _20 minutes_.

* Checking referenced projects is cancelled after short period by Roslyn, which means you get a lot of unresolved types, functions and so on (shown as red squiggles) even though the whole solution is built successfully.

* Find Usages returns duplicated results for linked files.

All these (and many other) issues has been fixed in master branch, and some new goodies has been added:

* ReSharper-like completion list ordering for members

![image](https://cloud.githubusercontent.com/assets/873919/23749154/e1b63824-04d7-11e7-95d1-ae97c66b918d.png)

(own properties, inherited properties, own fields, inherited fields, own methods, inherited methods,..., everything else)

* Proper glyphs for extension methods

![image](https://cloud.githubusercontent.com/assets/873919/23749355/d7009086-04d8-11e7-965a-9460f2ea7031.png)

* Completion glyphs reflect symbols accessibility

![image](https://cloud.githubusercontent.com/assets/873919/23749771/ddc5c1fa-04da-11e7-9772-4fc0fb9860dd.png)
