[up](../../../../GRMustache#documentation), [next](rendering_objects.md)

GRMustacheTagDelegate protocol
==============================

Overview
--------

This protocol lets you observe, and possibly alter the rendering of the Mustache tags that are provided with your own data: `{{ name }}`, `{{# name }}...{{/}}`, `{{^ name }}...{{/}}` and `{{$ name }}...{{/}}`, respectively *variable tags*, *section tags*, *inverted section tags*, and *overridable section tags*.

The first three are documented in the [Runtime Guide](runtime.md).

Overridable sections are documented in the [Partials Guide](partials.md).


Observing the rendering of Mustache tags
----------------------------------------

```objc
@protocol GRMustacheTagDelegate<NSObject>
@optional
- (id)mustacheTag:(GRMustacheTag *)tag willRenderObject:(id)object;
- (void)mustacheTag:(GRMustacheTag *)tag didRenderObject:(id)object as:(NSString *)rendering;
- (void)mustacheTag:(GRMustacheTag *)tag didFailRenderingObject:(id)object withError:(NSError *)error;
```

The _object_ argument is the rendered value: a string, a number, an array, depending on the data you provided.

The _tag_ argument represents the rendering tag: `{{ name }}`, `{{# name }}...{{/}}`, etc. It provides you with the following methods:

```objc
@interface GRMustacheTag: NSObject

// The tag type
@property (nonatomic, readonly) GRMustacheTagType type;

// A string describing the tag
- (NSString *)description;
@end

typedef enum {
    GRMustacheTagTypeVariable = 1 << 1,           // The type for tags such as {{ name }} and {{{ name }}}
    GRMustacheTagTypeSection = 1 << 2,            // The type for tags such as {{# name }}...{{/}}
    GRMustacheTagTypeOverridableSection = 1 << 3, // The type for tags such as {{$ name }}...{{/}}
    GRMustacheTagTypeInvertedSection = 1 << 4,    // The type for tags such as {{^ name }}...{{/}}
} GRMustacheTagType;
```

The `description` method provides a clear description of the tag, such as:

    <GRMustacheVariableTag `{{name}}` at line 18 of template /path/to/Profile.mustache>

Note that those methods do not allow you to build a complete "stack trace" of a template rendering.

For instance, a tag like `{{ person.name }}` is rendered once. Thus `mustacheTag:willRenderObject:` will be called once. If the person has been found, the rendered object will be the name of the person. If the person could not be found, the rendered object will be `nil`.

Also: if a section tag `{{# name }}...{{/}}` is provided with an array, its content is rendered several times. However `mustacheTag:willRenderObject:` will be called once, with the array passed in the _object_ argument.


### Observing the rendering of all tags in a template

In order to observe the rendering of all tags rendered by a template, you have your tag delegate enter the *base context* of the template.

The base context contains values and tag delegates that are always available for the template rendering. It contains all the ready for use filters of the [filter library](filters.md), for example. Contexts are detailed in the [Rendering Objects Guide](rendering_objects.md).

```objc
@interface Document : NSObject<GRMustacheTagDelegate>
- (NSString *)render;
@end

@implementation Document

- (NSString *)render
{
    NSString *templateString = @"{{greeting}} {{#person}}{{name}}{{/}}!";
    GRMustacheTemplate *template = [GRMustacheTemplate templateFromString:templateString error:NULL];
    template.baseContext = [template.baseContext contextByAddingTagDelegate:self];
    
    id data = @{
        @"greeting": @"Hello",
        @"person": @{
            @"name": @"Arthur"
        },
    };
    return [template renderObject:data error:NULL];
}

- (void)mustacheTag:(GRMustacheTag *)tag didRenderObject:(id)object as:(NSString *)rendering
{
    NSLog(@"%@ did render %@ as %@", tag, object, rendering);
}

@end

// <GRMustacheVariableTag `{{greeting}}` at line 1> did render Hello as Hello
// <GRMustacheVariableTag `{{name}}` at line 1> did render Arthur as Arthur
// <GRMustacheSectionTag `{{#person}}` at line 1> did render { name = Arthur } as Arthur
[[Document new] render];
```

### Observing the rendering of all tags in a section

When a Mustache section renders an object that conforms to the `GRMustacheTagDelegate` protocol, this object observes the rendering of all tags inside the section. As sections get nested, tags get more and more delegates.

Before we give an example, let's see how tag delegates can also *alter* the rendering.


Altering the rendering of Mustache tags
---------------------------------------

The value returned by the `mustacheTag:willRenderObject:` is the value that will actually be rendered.

You can, for instance, provide default rendering for missing values:

```objc
@interface Document : NSObject<GRMustacheTagDelegate>
- (NSString *)render;
@end

@implementation Document

- (NSString *)render
{
    NSString *templateString = @"{{greeting}} {{#person}}{{name}}{{/}}!";
    GRMustacheTemplate *template = [GRMustacheTemplate templateFromString:templateString error:NULL];
    template.baseContext = [template.baseContext contextByAddingTagDelegate:self];
    
    id data = @{
        @"greeting": @"Hello",
        @"person": @{
            @"firstName": @"Arthur"
        },
    };
    return [template renderObject:data error:NULL];
}

- (id)mustacheTag:(GRMustacheTag *)tag willRenderObject:(id)object
{
    if (object == nil) {
        NSLog(@"Missing value for %@", tag);
        return @"DEFAULT";
    }
    return object;
}

@end

// Missing value for <GRMustacheVariableTag `{{name}}` at line 1>
// Renders "Hello DEFAULT!"
[[Document new] render];
```

### Altering the rendering of tags in a section

As stated above, when a section renders an object that conforms to the `GRMustacheTagDelegate` protocol, this object observes the rendering of all tags inside the section.

The [Localization Sample Code](sample_code/localization.md) will give us an example, but let's have fun with numbers, and have Mustache format all numbers in a section attached to a `NSNumberFormatter` instance:

```objc
// Have NSNumberFormatter conform to the GRMustacheTagDelegate protocol,
// so that a formatter can format all numbers in a section:
@interface NSNumberFormatter(Document)<GRMustacheTagDelegate>
@end

@implementation NSNumberFormatter(Document)

- (id)mustacheTag:(GRMustacheTag *)tag willRenderObject:(id)object
{
    // Format all numbers that happen to be rendered by variable tags such as
    // `{{ count }}`.
    //
    // We avoid messing with sections, since they rely on boolean values of
    // numbers.
    
    if (tag.type == GRMustacheTagTypeVariable && [object isKindOfClass:[NSNumber class]]) {
        return [self stringFromNumber:object];
    }
    return object;
}

@end

NSString *templateString = @"x = {{x}}\n"
                           @"{{#percent}}x = {{x}}{{/percent}}\n"
                           @"{{#decimal}}x = {{x}}{{/decimal}}";
GRMustacheTemplate *template = [GRMustacheTemplate templateFromString:templateString error:NULL];

NSNumberFormatter *percentFormatter = [NSNumberFormatter new];
percentFormatter.numberStyle = NSNumberFormatterPercentStyle;

NSNumberFormatter *decimalFormatter = [NSNumberFormatter new];
decimalFormatter.numberStyle = NSNumberFormatterDecimalStyle;

id data = @{
    @"x": @(0.5),
    @"percent": percentFormatter,
    @"decimal": decimalFormatter
};

// On a French system:
// x = 0.5
// x = 50 %
// x = 0,5
NSString *rendering = [template renderObject:data error:NULL];
```


Tag Delegates as Cross-Platform Filters
---------------------------------------

Let's consider again the number formatting example above. We were able to render `{{#percent}}x = {{x}}{{/percent}}` as `x = 50 %`: The tag delegate attached to the `percent` has formatted the number `x` as a percentage.

You could also use [filters](filters.md) in order to format numbers: `x = {{ percent(x) }}` would render just as well.

However, `{{#percent}}x = {{x}}{{/percent}}` has one advantage over `x = {{ percent(x) }}`: it uses plain Mustache syntax, and is compatible with [other Mustache implementations](https://github.com/defunkt/mustache/wiki/Other-Mustache-implementations).

With such a common template, it's now a matter of providing different data, depending on the platform:

    // common template
    {{#percent}}x = {{x}}{{/percent}}
    
    // data for GRMustache
    {
      "x": 0.5,
      "percent": the_formating_tag_delegate
    }

    // data for other Mustache implementations
    {
      "percent": {
        "x": "50 %"
      }
    }

See? When you, GRMustache user, can provide your raw model data and have tag delegates do the formatting, users of the other implementations can still *prepare* their data and build a "view model" that contains the values that should be rendered. Eventually both renderings are identical.

Let's have a closer look at how you would convert a filter-based template to a tag delegate-based template: let's convert `{{ uppercase(name) }}`.

In order to turn the `uppercase` filter into a tag delegate, we need a Mustache section: `{{#uppercase}}{{name}}{{/uppercase}}`, and a tag delegate attached to the `uppercase` key:

```objc
@interface UppercaseTagDelegate : NSObject<GRMustacheTagDelegate>
@end

@implementation UppercaseTagDelegate
- (id)mustacheTag:(GRMustacheTag *)tag willRenderObject:(id)object
{
    return [[object description] uppercaseString];
}
@end

id data = @{
  @"name": @"Johannes Kepler",
  @"uppercase": [[UppercaseTagDelegate alloc] init],
};

NSString *rendering = [GRMustacheTemplate renderObject:data
                                            fromString:@"{{#uppercase}}{{name}}{{/uppercase}}"
                                                 error:NULL];
```

The final rendering is, as expected: "JOHANNES KEPLER".

Compatibility with other Mustache implementations
-------------------------------------------------

The [Mustache specification](https://github.com/mustache/spec) does not have the concept of "tag delegates".

**As a consequence, if your goal is to design templates that remain compatible with [other Mustache implementations](https://github.com/defunkt/mustache/wiki/Other-Mustache-implementations), use `GRMustacheTagDelegate` with great care.**


Sample code
-----------

The [Localization Sample Code](sample_code/localization.md) uses tag delegates for localizing portions of a template.


[up](../../../../GRMustache#documentation), [next](rendering_objects.md)
