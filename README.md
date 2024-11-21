# Representing Measures

**Stage**: 1

**Champion**: Ben Allen [@ben-allen](https://github.com/ben-allen)

**Author**: Ben Allen [@ben-allen](https://github.com/ben-allen)


# Goals and needs

Modeling units of measure is useful for any task that involves measurements from the physical world. It can also be useful for other types of measurement; for example, measurements of currency amounts. 

We propose to create a new object for representing measurements, for producing formatted string representations of measurements, and for converting measurements between scales.

Common user needs that can be addressed by a robust API for measurements include, but are not limited to:

* The need to convert measurements from one scale to another

* The need to format measurements into string representations

* Related to both of the above, the need to localize measurements. This can be relatively straightforward, as in the case of converting temperature measurements between Celsius and Fahrenheit, or Celsius and Kelvin. It can be more complex, though: notably, many locales represent measurements of certain types of object using mixed units
    - For example, in the United States it is customary to represent the heights of people in feet and inches.

* The need to represent and manipulate derived units, such as square meters for area and meters cubed for volume

* The need to represent and manipulate compound units, such as velocity expressed in kilometers per hour or density expressed in unit of mass per unit of volume

* The need to keep track of the precision of measured values. A measurement value represented with a large number of significant figures can imply that the measurements themselves are more precise than the apparatus used to take the measurement can support.

* The need to represent currency values. Often users will want to keep track of money values together with the currency in which those values are denominated.
    - As in the "feet and inches" example, sometimes it is useful to present currency values with the major and minor units separated, as in "19 dollars and 17 cents." See: [Design for currency and unit inputs that carry their values ](https://github.com/tc39/ecma402/issues/911#issuecomment-2238619851)

* The need to define custom units.
    - For example, CLDR includes units for velocity and acceleration, but none of the higher-order derivatives of position with respect to time.

# Description

We propose creating a new `Measure` API, with the following properties.

Note: ⚠️  All property/method names up for bikeshedding.

* `unit`, a String representing the measurement unit. This could be expressed using the names used by [CLDR's units.xml](https://github.com/unicode-org/cldr/blob/main/common/supplemental/units.xml). The API design below presumes that these names are used. For example, if a user wanted to use the `convertTo` method to convert a length measurement to feet and inches, they would provide the string `"foot-and-inch"` as the value of the `unit` argument.

* `value`, the numerical value of the measurement.

* `precision`, a number indicating the precision of the measurement. This precision is (provisionally) represented in terms of the number of fractional digits displayed.

* `exponent`, an integer representing the power to which the unit is raised. "centimeters squared" could be `{unit: "centimeter", exponent: 2}`. It may be preferable to use CLDR names for commonly-used units; `"cubic-meter"` instead of `{unit: "meter", exponent: 3}`, for example.

* `usage`, the type of thing being measured. Useful for localization.

### Precision
A big question is how we should handle precision. Currently this explainer assumes precision means fractional 
digits, not because it seems good but instead because it seems least-bad. [The Java Units of Measurement API](https://unitsofmeasurement.github.io/unit-api/) 
appears to resolve this problem by not handling precision at all.

## Constructor

* `Measure(value, {unit, precision, exponent, usage})`. Constructs a Measure with `value` as the numerical value of the Measure and `unit` as the unit of measurement, with the optional `precision` parameter used to specify the precision of the measurement. In the case of `unit` values indicating mixed units, the `value` is given in terms of the quantity of the *largest* unit. If no unit is provided, the value of unit is `"dimensionless"`. `exponent` indicates the power to which the underlying unit is raised; `exponent` of 2 on an object with "centimeter" as the `unit` value indicates centimeter-squared.

The object prototype would provide the following methods:

* `convertTo(unit, precision)`. This method returns a Measure in the scale indicated by the `unit` parameter, with the value of the new Measure being the value of the Measure it is called on converted to the new scale.

* `toString()`. This method returns a string representation of the unit.

* `toComponents()`. This method returns each component of the measurement as an object in an array. Intended for use
with mixed units.

```js
    let centimeters = new Measurement(30, {unit: "centimeter"})
    centimeters.toString()
    // "30 centimeters"

    footAndInch.toComponents()
    // [ {value: 5, unit: "foot"}, {value: 6, unit: "inch"}]
```

### Mixed units

We absolutely must include mixed units in Measurement, because they're absolutely 
needed for Smart Units. We can't just include foot-and-inch in Smart Units
and not Measurement, since that invites specifically the type of abuse of 
i18n tools for non-i18n purposes that we're trying to avoid with the Measure proposal

The value of mixed units should be expressed in terms of the largest unit in the mixed unit. 
(Alternately, the Measurement's value could be expressed in terms of the smallest unit. We're
going with largest for the example below)

```js
    let footAndInch = new Measurement(5.5, {unit: "foot-and-inch"})
    footAndInch.toComponents()
    // [ {value: 5, unit: "foot"}, {value: 6, unit: "inch"}]
    footAndInch.toString()
    // "5 feet and 6 inches"
```


## Mathematical operations

Below is a list of mathematical operations that we should consider supporting. 
Assume that we use [CLDR data](https://github.com/unicode-org/cldr/blob/main/common/supplemental/units.xml) 
for now, and that both our unit names and the conversion constants are as in CLDR.


### Proposed mathematical operations

Raise a Measurement to an exponent:

```js
    let measurement = new Measurement(10, {unit: "centimeter"});
    measurement.exp(3);
    // { value: 1000, unit: "cubic-centimeter"}
```

* Multiply/divide a measurement by a scalar

```js
    let measurement = new Measurement(10, {unit: "centimeter"});
    measurement.multiply(20);
    // {value: 200, unit: "centimeter"};
    measurement.divide(10);
    // {value: 20, unit: "centimeter"};
```

* Add/subtract two measurements of the same dimension

```js
    let measurement1 = new Measurement(10, {unit: "centimeter"});
    let measurement2 = new Measurement(5, {unit: "centimeter"});
    measurement1.add(measurement2);
    // {value: 15, unit: "centimeter"}

    let measurement3 = new Measurement(5, {unit: "meter"});
    measurement1.add(measurement3);
    // in the units of the Measurement that `add` is called on?
    // {value: 510, unit: "centimeter"}

    measurement1.subtract(measurement2);
    // {value: 505, unit: "centimeter"}
    
    // Precision is given in fractional digits. If doing calculation with units with
    // precision values, the precision should be set to the least precise (taking 
    // into account that units will have to be converted to the same scale)
    let measurementWithPrecision1 = new Measurement(10.12, {unit: "centimeter", precision: 2};
    let measurementWithPrecision2 = new Measurement(10.1234 {unit: "centimeter", precision: 4};
    measurementWithPrecision1.add(measurementWithPrecision2);;
    // {value: 20.24, unit: "centimeter", precision: 2}

```

* Multiply / divide a Measurement by another Measurement 

```js
    let gallons = new Measurement(2, {unit: "gallon"});
    let miles = new Measurement(30, {unit: "mile"});
    miles.divide(gallon);
    // {value: 15, unit: "miles-per-gallon"}

    let centimeters1 = new Measurement(10, {unit: "centimeter"});
    let centimeters2 = new Measurement(5, {unit: "centimeter"});
    centimeters1.multiply(centimeters2);
    // {value: 50, unit: "square-centimeter" }
    // alternately: {value: 50, unit: "centimeter", exponent: 2}

    centimeters1.divide(centimeters2);
    // {value: 10, unit: "dimensionless"}
```

* Convert between scales

```js
    let inches = new Measure(12, {unit: "inch"});
    inches.convertTo("centimeter");
    // {value: 30.48, unit: "centimeter}
    // using optional `precision` option
    inches.convertTo("centimeter", 1);
    // { value: 30.5, unit: "centimeter" }
```

* All of the above operations throw if incompatible dimensions are used. For example, adding 
a measure of volume to a measure of speed would throw, as would adding a scalar value to a 
non-`"dimensionless"` Measurement.

### User-defined units

Users can specify values for `unit` other than the ones we support. The only mathematical 
operations that apply to Measurements with non-standard units are the ones involving scalars.

### `convertToLocale` method shifted to Smart Units

The method `convertToLocale` is shifted to Smart Units. This method can use 
a `usage` option for Measurements in order to properly localize measurements of
certain types of thing.

* convertToLocale(locale)

```js
    let centimeters = Measurement(30.48, {unit: "centimeter", usage: "person-height"});
    centimeters.convertToLocale("en");
    // {value: 5.5, unit: "foot-and-inch"}

    centimeters.toLocaleString('en');
    // "5 feet and 6 inches"
```

