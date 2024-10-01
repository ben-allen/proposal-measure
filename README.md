# Representing Measures

**Stage**: Not presented yet to TC39

**Champion**: Ben Allen [@ben-allen](https://github.com/ben-allen)

**Author**: Ben Allen [@ben-allen](https://github.com/ben-allen)


# Use cases and goals

There is a frequent need to keep numerical measurements together with their units of measurement. Moreover, there is a frequent need to convert measurements to different scales. We propose to create a new object for representing measurements and for converting measurements between scales.

This proposal was originally inspired by, and is to some extent a prerequisite for, the Smart Units proposal for localizable measurements. However, because the need to represent measurements and to convert between measurement scales transcends the context of localization, we have decided to split it out into its own proposal. Below find some of the use cases we'd like to address.

* Localizing measurements. Content with measurements represented in unfamiliar scales &mdash; inches where centimeters are expected, temperatures given in degrees Fahrenheit where degrees Celsius are expected &mdash; can be incomprehensible to users. There are complexities involved in localizing measurements that can be more easily solved by creating a Measure object:

    - Need to represent mixed-unit measurements. For example, in the locale `"en-CA"` the heights of people are typically measured in feet and inches rather than in centimeters, despite this locale using the metric system for most types of measurements.

    - Relatedly, localizing a measurement requires presenting that measurement at the appropriate precision. Formatting "3 miles" as "4.8280 kilometers" is poor localization; generally, people don't use four decimal places to give road distances. Additionally, the over-precise representation inappropriately implies that the original measurement itself was that precise.

* Localizing currency values.
   
    - Often users will want to keep track of money values together with the currency in which those values are denominated. As with the "feet and inches" example, sometimes it is useful to present currency values with the major and minor units separated, as in "19 dollars and 17 cents." See: [Design for currency and unit inputs that carry their values ](https://github.com/tc39/ecma402/issues/911#issuecomment-2238619851)

* General unit conversion. It is necessary when localizing measurements to perform mathematical operations converting between scales. However, the need to convert between measurement scales is, needless to say, not confined to internationalization/localization. It is common for internationalization-related tools to be misused for non-internationalization-related purposes. In the interest of minimizing this misuse, we would like to provide a limited unit-conversion tool that can perform the same conversions needed for localizing measurements.


# Description

We propose creating a new object, `Measure`, with the following properties. Note: ⚠️  all property/method names up for bikeshedding.

* `unit`, a String representing the measurement unit.
* `value`, the value (which could be a Number, a decimal String, a BigInt, a Decimal...) of the major unit in the measurement. In the case of a foot-and-inch measurement, this would be the number of feet.
* `minorValue`, the value (if one exists) of the minor unit in the measurement. In the case of a foot-and-inch measurement, this would be the number of inches.

The object prototype would provide the following methods:

* `convert(unit, precision)`. This method returns a Measure in the scale given in the parameter, with the value of this new Measure being the value of the Measure it is called on converted to the new scale. The `precision` parameter is optional; the default should be a relatively large number of digits of precision.

* `localeConvert(locale, usage)`. This method returns a Measure in the customary scale and at the customary precision for `locale`. Some locales use different measurement scales and precision based on the type of thing being measured and the measurement value itself; for example, in `en-US` it's customary to give measurements of road distance that are less than half of a mile in feet instead of miles.

We propose using a limited subset of the units and conversion factors in CLDR's [units.xml](https://github.com/unicode-org/cldr/blob/main/common/supplemental/units.xml) to determine what units and what conversions are supported. Since this proposal emerges from the context of internationalization/localization, we prefer that the units supported be the units useful for that context, with additional units only added sparingly.

# Examples

```js

let m = new Measure(1.8, "meter");
m.convert('foot', 2);
// { value: 5.91, unit: "foot" }
m.localeConvert("en-CA", "personHeight")
// {value: 5, minorValue: 11}
```

```js
let m = new Measure (4000, "foot");
m.convert("kilometer");
// {value:  1.2192, unit: "kilometer"}
m.localeConvert('en-US', "road");
// {value: 0.76, unit: "mile"}
```
