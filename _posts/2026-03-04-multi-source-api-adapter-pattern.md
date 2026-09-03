---
layout: post
title: "Multi-Source API Adapter Pattern in Ruby"
date: 2026-03-04 09:00:00 -0600
summary: "Each vendor adapter says what units it returns, a converter turns them into the units the customer asked for, and Alba serializers shape the JSON. Adding a vendor doesn't touch the converter, the outputs, or the other adapters."
tags: [ruby, architecture, api, patterns]
model: "Claude"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

One vendor returns Fahrenheit and miles per hour, with humidity as `85`. Another returns Celsius, meters per second, and precipitation in millimeters. A third mixes systems inside a single response: Fahrenheit temperatures next to precipitation in centimeters, visibility in meters, and moon phase as a number from 0 to 360 rather than 0 to 1. Each wraps those values in its own JSON structure.

Our weather API pulls from 10+ vendors, and customers pick the units they want back. A request for SI has to come back in SI whether the data came from the Fahrenheit vendor or the Celsius one. With that many vendors, adding the next one can't mean editing the ones already running.

## The Solution

Four parts, each with one job:

- **Source adapters** turn each vendor's JSON into one internal shape
- **Source units** say what units each adapter returns
- **A converter** turns source units into the units the customer asked for
- **Output serializers** format the final JSON

Adapters parse, the converter does math, and serializers format. None of them looks inside another.

## Implementation

### The Main Interface

One class handles a request. It takes the source name, the coordinates, and the units the customer wants. Every value it returns has already been through the converter:

```ruby
class Api::Weather
  def initialize(args = {})
    @source = (args[:source] || "mock").freeze
    @return_units_base = (args[:units].presence || "us").freeze
    # ... other params
  end

  def converter
    @_converter ||= Api::Converter.new(
      source_units: source.source_units,
      return_units: return_units
    )
  end

  def currently
    @_currently ||= Api::Currently.new(
      temperature: converter.temperature(currently_data.temperature),
      wind_speed: converter.wind_speed(currently_data.wind_speed),
      # ... other fields
    )
  end
end
```

We build the converter once per request, from the units the source declares and the units the customer asked for. Nothing past `currently` sees a raw vendor value.

### Source Adapters

Each vendor gets one class that inherits from a base. The base sets three methods: which units this source uses, how to return current conditions, and an optional hook for fetching data in parallel.

```ruby
class Api::Sources::Base
  # Sentinel values for documentation
  DATA_MISSING = nil  # Data unavailable from this source
  DATA_PENDING = nil  # Available but not yet implemented
  DATA_SKIPPED = nil  # Available but intentionally not fetched

  def source_units
    Api::Units.build(:us)  # Default; override in subclasses
  end

  def currently
    Api::Currently.new  # Override to return actual data
  end

  def preload(_output)
    # Override to eager-load data in parallel
  end
end
```

A real adapter is mostly a map from the vendor's field names to ours:

```ruby
class Api::Sources::ExampleWeather < Api::Sources::Base
  def currently
    Api::Currently.new(
      temperature: data.dig(:current, :temp_f),
      wind_speed: data.dig(:current, :wind_mph),
      humidity: data.dig(:current, :humidity_pct),
      # Map each external field to internal structure
    )
  end

  def source_units
    Api::Units.build(:us,
      percentage: "integer"  # This source returns 85 not 0.85
    )
  end
end
```

The adapter doesn't know what units the customer wants. It says what units it returns, US with whole-number percentages here, and the converter does the rest. The three `DATA_*` constants in the base class are all `nil` at runtime. They exist so that a field set to `DATA_MISSING` tells the next person who reads the adapter something different from one set to `DATA_PENDING` or `DATA_SKIPPED`.

### Source Units vs Return Units

One unit system per source wasn't enough, because real vendors mix systems in one response. So the units object takes a base system plus overrides for single fields:

```ruby
# Source A: returns Fahrenheit, mph, percentages as integers
def source_units
  Api::Units.build(:us, percentage: "integer")
end

# Source B: returns Celsius, m/s, precip in mm, percentages as integers
def source_units
  Api::Units.build(:si,
    percentage: "integer",
    precip_accumulation: "mm"
  )
end

# Source C: returns mixed units (tricky!)
def source_units
  Api::Units.build(:us,
    moon_phase: "degrees",      # 0-360 not 0-1
    percentage: "integer",
    precip_accumulation: "cm",  # Metric precip
    precip_intensity: "mmhr",   # but US temps
    visibility: "m"
  )
end
```

Source C uses US units for temperature and wind but metric for precipitation and visibility, and its declaration says so field by field. The converter trusts that declaration. It raises an error on a unit pair it doesn't know, but it can't tell whether a declared unit matches the data.

### The Converter

The converter is a lookup table keyed by `[from, to]` pairs, one table per kind of measurement. Each entry is a small function of the value and nothing else:

```ruby
class Api::Converter
  include Api::Concerns::Rounding  # round_f(val, precision = 2)

  attr_accessor :source_units, :return_units

  # One vendor sends -999 for "no data"; treat it like nil everywhere
  MISSING_DATA_SENTINEL = -999

  TEMPERATURE_CONVERSIONS = {
    ["c", "f"] => ->(v) { (v * 9.0/5.0) + 32 },
    ["f", "c"] => ->(v) { (v - 32) * 5.0/9.0 },
    # ... other conversions
  }.freeze

  def temperature(val)
    return nil if missing?(val)

    val = val.to_f

    # No conversion needed if units match
    return round_f(val) if source_units.temperature == return_units.temperature

    converter = TEMPERATURE_CONVERSIONS[[source_units.temperature, return_units.temperature]]
    raise Api::Weather::NotImplementedError, "#{source_units.temperature} -> #{return_units.temperature}" unless converter

    round_f(converter.call(val))
  end

  private

  def missing?(val)
    val.nil? || val == MISSING_DATA_SENTINEL
  end
end
```

When the source and return units match, the value is only rounded. An unknown pair raises instead of guessing. Wind speed, pressure, visibility, precipitation, and the rest follow the same shape, most with a table of multipliers instead of lambdas.

### Output Serializers with Alba

The last layer formats the JSON. We use [Alba](https://github.com/okuramasafumi/alba) serializers. Each output is its own resource class that lists the fields it carries and how many rows of each series it returns:

```ruby
class Api::Outputs::Base
  include Alba::Resource

  LIMITS = { minutely: 60, hourly: 48, daily: 8 }

  transform_keys :lower_camel

  attribute :latitude, &:lat
  attribute :longitude, &:lon

  association :currently do
    attributes :temperature, :humidity, :wind_speed, :icon  # ...
  end

  association :hourly do
    attributes :summary

    association :data, proc { |data, _, _| data.first(LIMITS[:hourly]) } do
      attributes :temperature, :precip_probability, :time  # ...
    end
  end
end
```

Output variants don't inherit from the base. A fuller output is a second class with the same shape, higher limits, and more fields:

```ruby
class Api::Outputs::Full
  include Alba::Resource

  LIMITS = { minutely: 999, hourly: 999, daily: 999 }

  transform_keys :lower_camel

  association :currently do
    attributes :temperature, :humidity, :wind_speed, :icon,
      :aqi, :pressure_trend  # Additional fields
  end
  # ...
end
```

The serializer only sees converted values, so one class serves a request in any unit system. The coordinator picks the minimal, full, or custom output by class name. There's no branch inside it.

## Data Flow

A request for source `foo` in SI units goes through the layers in order:

```
Request (source=foo, units=si)
    ↓
Api::Weather (coordinator)
    ↓
Api::Sources::Foo (adapter)
    → Fetches external API
    → Returns normalized shape
    → Declares source_units
    ↓
Api::Converter
    → source_units → return_units
    → Converts each data point
    ↓
Api::Outputs::Base (Alba)
    → Serializes to JSON
    ↓
Response (data in SI units)
```

## Adding a New Source

Adding a source is a fixed set of steps:

1. Create `Api::Sources::NewSource < Api::Sources::Base`
2. Implement data-fetching methods
3. Map external fields to internal shape
4. Declare `source_units` accurately
5. Add to the source registry

The converter, the outputs, and the existing sources don't change. Step 4 is the risky one. If you declare Celsius for a vendor that sends Fahrenheit, the converter accepts it, because `["c", "f"]` is a valid pair. A 70 comes out as 158, and nothing downstream can tell.

## Lessons Learned

- **Declare units per source and per field.** Guessing a vendor's units causes quiet bugs, and a vendor can mix systems in one response.
- **Fetch lazily, because external APIs charge per request.** Accessors on the adapter fetch only what the output asks for, and only once.
- **Keep converters pure.** Same inputs, same outputs, nothing else touched, so each conversion is easy to test on its own.
- **Use named constants to say why a field is nil.** `DATA_MISSING`, `DATA_PENDING`, and `DATA_SKIPPED` are all nil, but each records a different reason: the vendor doesn't have it, we haven't implemented it yet, or we chose not to fetch it.
