csharp-ember-handlebars
=======================

.NET Library for pre-compilation of Handlebars templates directly into Ember's `TEMPLATES` collection. 

![NuGet-Install](https://raw.github.com/MilkyWayJoe/csharp-ember-handlebars/master/nuget.png)
## Example

In order to register a template, the builder expects a string with the template name and and another 
with its corresponding markup:

```csharp
var builder = new Ember.Handlebars.TemplateBuilder();
builder.Register("application", "<h1>{{App.name}}</h1>");
```

To retrieve the pre-compiled templates, simply call `builder.ToString()`. This will return the following:     

```javascript
Ember.TEMPLATES["application"] = Ember.Handlebars.template(
    function anonymous(Handlebars, depth0, helpers, partials, data) {
        helpers = helpers || Ember.Handlebars.helpers; data = data || {};
        var buffer = '', stack1, escapeExpression=this.escapeExpression;
        data.buffer.push("<h1>");
        stack1 = helpers._triageMustache.call(depth0, "App.name", {hash:{},contexts:[depth0],data:data});
        data.buffer.push(escapeExpression(stack1) + "</h1>");
        return buffer;
    }
);
```

The above shows the basic registration of templates, but going forward you might want to add this functionality 
as part of your application's build process including this functionality directly into the `BundleConfig` class.

## Built-in IBundleTransform Implementation
This library now has a built-in implementation of `IBundleTransform` which can be used directly in ASP.NET MVC's 
`BundleConfig` class as shown below:

```csharp
bundles.Add(new Bundle("~/bundles/templates", new EmberHandlebarsBundleTransform())
       .Include("~/scripts/app/templates/*.handlebars"));
```

The `EmberHandlebarsBundleTransform` class has the flag `MinifyTemplates` which specifies whether or not the returning 
JavaScript will be minified. The default is `true`, but if for some reason you need the template functions not to be 
minified, do the following:

```csharp
bundles.Add(new Bundle("~/bundles/templates", 
            new EmberHandlebarsBundleTransform() { MinifyTemplates = false })
       .Include("~/scripts/app/templates/*.handlebars"));
```

Note: The built-in `EmberHandlebarsBundleTransform` allows your templates to have whichever extension 
that better suits your development process. The examples above use *.handlebars as an extension but it could be 
something else, like *.html for example.

## Creating a custom implementation of IBundleTransform
The code snippet below shows the implementation of the built-in `EmberHandlebarsBundleTransform` class which you can 
rename and adapt to your needs:

```csharp
using System;
using System.IO;
using System.Web;
using System.Web.Optimization;
using Microsoft.Ajax.Utilities;

public class EmberHandlebarsBundleTransform : IBundleTransform {
    
    private bool minifyTemplates = true;
    
    public bool MinifyTemplates {
        get { return this.minifyTemplates; }
        set { this.minifyTemplates = value; }
    }

    public void Process( BundleContext context, BundleResponse response ) {
        var builder = new Ember.Handlebars.TemplateBuilder();

        foreach ( var assetFile in response.Files ) {
            var path = context.HttpContext.Server.MapPath(assetFile.VirtualFile.VirtualPath.Replace("/", "\\"));
            var template = File.ReadAllText( path );
            var templateName = Path.GetFileNameWithoutExtension( path ).Replace("-", "/");
            builder.Register( templateName, template );
        }

        var content = builder.ToString();
        if ( minifyTemplates ) {
            var minifier = new Minifier();
            var c = minifier.MinifyJavaScript(content);
            if ( minifier.ErrorList.Count <= 0 ) {
                content = c;
            }
        }

        response.ContentType = "text/javascript";
        response.Cacheability = HttpCacheability.Public;
        response.Content = content;

    }
}
```

