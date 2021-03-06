# DumpEditable
DumpEditable is an extensible 'inline editor' extension for [LINQPad](https://www.linqpad.net), with `Dump`-like semantics. It allows you to dump an editable representation of an object onto the results view - useful for quickly adding interactivity to a query without having to spend time arranging and wiring up controls. 

DumpEditable aims to provide reasonable defaults out of the box but be flexible enough for you to customise it to your specific needs. If you install it into your 'My Extensions', you can build up a library of tailored editors available to all your queries. Or, you can just add extensions that make sense for any particular query.

## Installation

⚠🚨**DumpEditable is currently alpha quality and probably has bugs and things I havent thought about! Take care before using it in important/Production queries**🚨⚠

- Add the [`DumpEditable.LINQPad` NuGet Package](https://nuget.org/packages/DumpEditable.LINQPad) to your query or My Extensions.

- Add `LINQPad.DumpEditable` to your using namespaces.

DumpEditable comes with LINQPad samples, including basic 'How To' guides and a few more complete demos. When you install the package into your query, these samples will show up automatically in the samples pane.

## Usage

This section outlines the basic concepts available in DumpEditable. The LINQPad samples provide more detail. 

### Showing an editor
Getting an editor onto the results pane is as easy as calling `.DumpEditable()` on an object. `DumpEditable()` returns the input object so can be chained similarly to LINQPad's `Dump()`:

![basic poco dumped with editor displayed](https://ryandavis.io/content/images/2019/05/dump-editable/basic.png)
Here we see we got a basic editor for our `Pet` object without writing any code! Clicking any of the properties will allow us to modify the property values. 

### Handling changes 
In some cases, you may be running some kind of loop in your query which will give you the opportunity to read new values on your changed object. In many cases though, you'll want to be notified when a change occurs. You can do this by taking a reference to the `EditableDumpContainer` - an optional `out` parameter on `DumpEditable` - and adding change handlers. There are three ways you can register for change notifications - indiscriminate, reflection-based, and strongly typed - you can pick the one that best fits a given use case.

![demonstration of adding change handlers](https://ryandavis.io/content/images/2019/04/dump-editable/change-handling.png)

### Dumping collections

DumpEditable allows you to dump `IEnumerable`s using the `DumpEditableEnumerable()` extension. Many times you'll want to be dumping a 'materialised' collection (a `List`, array, etc. ) - if you dump an enumerable proper it will be re-evaluated after every change, and if that results in new objects, the changes made to the old objects will not be apparent. However, there are some situations in which re-evaluation can be useful.

![enumerable dump output in results pane](https://ryandavis.io/content/images/2019/05/dump-editable/enumerable.png)
When adding change handlers to dumped collections you can use the object parameters on `OnPropertyValueChanged` `(*obj*, prop, val) => ` and `AddChangeHandler (*obj*, value) => ` to know which item was affected. 

### Dumping anonymous types

In C#, anonymous types are read-only. However, defining and modifying anonymous type instances would be super convenient for interactive LINQPad queries, so DumpEditable makes it possible. 

![anonymous type dump output in results pane](https://ryandavis.io/content/images/2019/05/dump-editable/anonymous.png)
Modifying instances of anonymous types can cause problems in the Real World, because methods like `GetHashCode()` and `Equals()` - which benefit from the assumption that anonymous types are read only - can become incorrect. For basic editor control scenarios this is not a problem - but if you're doing anything more advanced, remember to be aware of the implications. 

### Adding custom editors

DumpEditable works by evaluating each property of the target object against `EditorRule`s to decide whether a specific editor should be displayed. If no `EditorRule`s match for a given property, the default LINQPad output will be displayed instead. DumpEditable ships with basic editors for primitives and enums. You can extend DumpEditable by adding custom `EditorRule`s, which will be evaluated prior to the built-ins (LIFO). Rules consist of a `match` function, which decides whether the rule should apply to a given object property, and a `getEditor` function, which provides the editor content that will be displayed for the object property. The editor content should include any functionality required to get new values.

`EditorRule` has a few helper methods you can use to make things easier. For example, the aptly named `EditorRule.ForTypeWithStringBasedEditor<T>(Func<string, out T, bool> parseFunc)` lets you provide a type parameter `T` and a `TryParse`-style `string -> T` conversion function, and gives you the rest - a basic text box implementation - for free. In the below case, we add support for editing `Guid`s:
![adding a basic guid editor rule](https://ryandavis.io/content/images/2019/05/dump-editable/editor-rule-basic.png)

For more control, you can implement an `EditorRule` from scratch, like in the below case where we match on a specific type's property, and provide an editor with images for content:

![adding a food selector rule](https://ryandavis.io/content/images/2019/04/dump-editable/editor-rule-foodselector.png)
When creating editor rules, it's important to use the provided `setVal` callback, rather than applying reflection directly - the `setVal` callback encapsulates the functionality required to properly change a value, including refreshing the editor contents and applying special processing such as anonymous type mutation. If you want to suppress automatic refresh of an editor - for example, to avoid recreating a control - you can set `DisableAutomaticRefresh` of the editor rule to `false`, or specify it via the optional `disableAutomaticRefresh` parameter on `EditorRule.For` or `EditorRule.ForType`. 

In both of the examples above we added 'global rules' that apply to all `EditableDumpContainer`s. You can also add a rule to an individual instance using the `AddEditorRule` method on that instance. Instance rules are evaluated before global rules.

DumpEditable also includes a slider control wrapper that you can pass to `getEditor`, accessible via the `Editors.Slider` overloads. 

![using a slider editor](https://ryandavis.io/content/images/2019/05/dump-editable/slider-editor.png)
You can provide a minimum and maximum value using the appropriate overload, or rely on the "*" settings button at runtime. As the LINQPad slider works with integers only, using it with types other than integers requires you to provide `Func<T, object> toInt` and `Func<int,T> fromInt` conversion parameters. Note that your conversion function does not need to result in an `int` - rather, it must result in something that can be turned into an `int` using `Convert.ToInt32`, which DumpEditable will call internally. This saves you from needing to pollute your code with casts. A typical use of the conversion functions might be to specify a percentage-based slider by multiplying/dividing a `float` or `double` by 100. 

![using a slider conversion function](https://ryandavis.io/content/images/2019/05/dump-editable/slider-percent.png)
There might be other clever ways to make use of this functionality.

### Nested object editor expansion

As you might have noticed earlier, nested objects within anonymous type instances are made editable automatically. Solving this problem for POCOs requires a bit more smarts (for example, handling of reference loops), which I haven't added yet, so nested objects on POCOs are not automatically evaluated. In the meantime, it's possible to add rules using the `EditorRule.ForExpansion(Func<object, PropertyInfo> match)` helper, which will cause any matching properties to be expanded with an editor. 

## Things to watch for

- this hasn't been tried against any data contexts, database models etc. It might work fine; it might do Bad things like try to pull entire tables into memory. Use at your own risk

- DumpEditable requires the query to 'keep running' (`Util.KeepRunning`) for property changes to work nicely after your query body has finished executing (which is likely to occur if you're using an event-based model). By default, DumpEditable automatically calls `Util.KeepRunning` for each running container. If you want to manage query lifetime yourself, can disable this behaviour by setting `EditableDumpContainer.DefaultOptions.AutomaticallyKeepQueryRunning` to `false` before dumping your first object. 

- Because it's early days, DumpEditable will throw if it encounters an error when trying to generate an editor output. If you don't like this and would prefer it to fall back to displaying the usual LINQPad output, you can make sure `EditableDumpContainer.DefaultOptions.FailSilently` is set to `true` before dumping the offending object.

- The source contains some scary type signatures that could probably be improved (named delegates?) and some of them leak into the extension API. While this is <v1.0 these end up getting refactored. 

- You can't `DumpEditable` a value type directly, but you can wrap it in an anonymous type. For example:
  ```
  2.DumpEditable() // bad 😡
  
  new { N = 2 }.DumpEditable() // good! 🤠
  ```

## Plans 

Right now, the featureset of DumpEditable is mostly driven by things I've thought of needing when putting together interactive/dynamic queries. If you have thoughts on things that you think should be possible, please let me know. In the short term, the things I'd like to add include:

- better support for nested editor expansion
- automatically update the editor display for `INotifyPropertyChanged` types
- better inline configuration api, maybe something fluent like: 
```
.DumpEditable(
    c => c.WithEditorRule(..)
          .AddChangeHandler(..))
```

## Contributions

Welcome! Open an issue to discuss what you're thinking about. 

## Acknowledgements

* The dynamic type generation and anonymous object mutation features were adapted from implementations found in (of course) StackOverflow questions. These are mentioned in the source

* `CompositeDisposal` implementation taken from [System.Reactive](https://github.com/dotnet/reactive)

* Thanks to [Joe Albahari](https://twitter.com/linqpad?lang=en), the creator of LINQPad for creating and continuing to improve such a uniquely powerful, flexible and performant tool. 
