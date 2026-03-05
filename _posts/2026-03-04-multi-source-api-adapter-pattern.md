---
layout: post
title: "Multi-Source API Adapter Pattern in Ruby"
date: 2026-03-04 09:00:00 -0600
summary: "A pattern for normalizing multiple external APIs into a unified interface with automatic unit conversion."
tags: [ruby, architecture, api, patterns]
---

## The Problem

When building an application that aggregates data from multiple external APIs, you quickly hit several challenges:

1. **Inconsistent data formats** - Each source returns different JSON structures
2. **Unit mismatches** - One API returns Celsius, another Fahrenheit; one uses km/h, another mph
3. **Customer flexibility** - Users want data in their preferred units regardless of source
4. **Maintainability** - Adding new sources shouldn't require touching existing code

## The Solution

We built a multi-source adapter pattern with three key concepts:

- **Source adapters** normalize external API data into a standard internal shape
- **Source units** declare what units each adapter returns (so the system knows what it's working with)
- **A converter** transforms data from source units to requested return units
- **Output serializers** render the final response format

The architecture keeps each concern isolated: adapters focus on parsing external data, the converter handles math, and serializers handle formatting.

## Implementation

### The Main Interface

The entry point coordinates everything. It accepts parameters for source selection, coordinates, and desired output units:

```ruby
class Api::Weather
  def initialize(args = {})
    @source = (args[:source] || "default").freeze
    @return_units_base = (args[:units] || "us").freeze
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

The key insight: every data point passes through the converter, which knows both what units the source provides and what units the customer requested.

### Source Adapters

Each external API gets an adapter class inheriting from a base:

```ruby
class Api::Sources::Base
  # Sentinel values for documentation
  DATA_MISSING = nil  # Data unavailable from this source
  DATA_PENDING = nil  # Available but not yet implemented

  def source_units
    Api::Units.build(:us)  # Default; override in subclasses
  end

  def currently
    Api::Currently.new  # Override to return actual data
  end

  def preload(output)
    # Override to eager-load data in parallel
  end
end
```

A concrete adapter maps the external API's structure to the internal shape:

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

### Source Units vs Return Units

This is where the pattern shines. Each source declares what units it returns:

```ruby
# Source A: returns Fahrenheit, mph, percentages as integers
def source_units
  Api::Units.build(:us, percentage: "integer")
end

# Source B: returns Celsius, m/s, precip in mm
def source_units
  Api::Units.build(:si,
    precip_accumulation: "mm",
    precip_intensity: "mmhr"
  )
end

# Source C: returns mixed units (tricky!)
def source_units
  Api::Units.build(:us,
    moon_phase: "degrees",      # 0-360 not 0-1
    precip_accumulation: "cm",  # Metric precip
    precip_intensity: "mmhr",   # but US temps
    pressure: "mb"
  )
end
```

The units object supports overrides for individual measurements, handling real-world APIs that mix unit systems.

### The Converter

The converter does the math to transform between unit systems:

```ruby
class Api::Converter
  TEMPERATURE_CONVERSIONS = {
    ["c", "f"] => ->(v) { (v * 9.0/5.0) + 32 },
    ["f", "c"] => ->(v) { (v - 32) * 5.0/9.0 },
    # ... other conversions
  }.freeze

  def temperature(val)
    return nil if val.nil?

    # No conversion needed if units match
    return val if source_units.temperature == return_units.temperature

    converter = TEMPERATURE_CONVERSIONS[[source_units.temperature, return_units.temperature]]
    converter.call(val.to_f)
  end
end
```

The pattern repeats for each measurement type: wind speed, pressure, visibility, precipitation, etc.

### Output Serializers with Alba

Finally, [Alba](https://github.com/okuramasafumi/alba) serializers define the JSON structure:

```ruby
class Api::Outputs::Base
  include Alba::Resource

  transform_keys :lower_camel

  attribute :latitude, &:lat
  attribute :longitude, &:lon

  association :currently do
    attributes :temperature, :humidity, :wind_speed, :icon
  end

  association :hourly do
    association :data do
      attributes :temperature, :precip_probability, :time
    end
  end
end
```

Different output formats (minimal, full, custom) just inherit and add attributes:

```ruby
class Api::Outputs::Full < Api::Outputs::Base
  association :currently do
    attributes :aqi, :uv_index, :pressure_trend  # Additional fields
  end
end
```

## Data Flow

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

The beauty of this pattern: adding a new source is mechanical:

1. Create `Api::Sources::NewSource < Api::Sources::Base`
2. Implement data-fetching methods
3. Map external fields to internal shape
4. Declare `source_units` accurately
5. Add to the source registry

No changes needed to the converter, outputs, or existing sources.

## Lessons Learned

- **Declare source units explicitly** - Assuming units leads to subtle bugs. Each source adapter must declare what it actually returns.

- **Handle mixed units** - Real APIs are messy. A source might return Fahrenheit temperatures but millimeter precipitation. The units system must support per-field overrides.

- **Lazy loading saves costs** - External APIs charge per request. Only fetch data that's actually needed. The adapter pattern supports this naturally with memoization.

- **Converters should be pure** - Given the same inputs, conversions produce the same outputs. No side effects, easy to test.

- **Sentinels document intent** - Using `DATA_MISSING` vs `DATA_PENDING` vs `DATA_SKIPPED` makes it clear why a field is nil: unavailable, not-yet-implemented, or intentionally omitted.

---

## How This Post Was Made

**Prompt:** "create a post, use the post skill, and pr skill, do a writeup of the basics of the Api::Weather system, we have the standard interface / entrypoint, then source adapters, source_units, return_units, the converter, and outputs using alba. Review the readme carefully, especially the part about how to add new source adapters. Review Api::Weather and Api::Converter etc. provide concise examples to illustrate the system, but not complete code. We don't want to share all the 'secret sauce' here, but we want to provide the pattern for other people to consider for adaptation in their own projects. remember to save this prompt for the pr etc."

Generated by Claude using the blog-post-generator skill. Based on code review of a production weather API system with 10+ source adapters.
