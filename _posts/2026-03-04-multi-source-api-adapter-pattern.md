---
layout: post
title: "Multi-Source API Adapter Pattern in Ruby"
date: 2026-03-04 09:00:00 -0600
summary: "Each source adapter declares the units it returns, a converter turns them into the units the customer asked for, and Alba serializers shape the response. Adding a vendor touches no existing code."
tags: [ruby, architecture, api, patterns]
---

## The Problem

One vendor returns Fahrenheit and miles per hour, with humidity as `85`. Another returns Celsius, meters per second, and precipitation in millimeters. A third mixes systems inside a single response: Fahrenheit temperatures next to precipitation in centimeters, pressure in millibars, and moon phase as a number from 0 to 360 rather than 0 to 1. Each wraps those values in its own JSON structure.

The application behind this post is a weather API backed by 10+ vendor sources, and its customers choose their own units. A request for SI has to come back in SI whether the data came from the Fahrenheit vendor or the Celsius one. With that many sources, adding the next one cannot mean editing the ones already in production.

## The Solution

Four parts, each with one job:

- **Source adapters** normalize external API data into a standard internal shape
- **Source units** declare what units each adapter returns
- **A converter** transforms data from source units to the requested return units
- **Output serializers** render the final response format

Adapters parse, the converter does math, and serializers format. None of them reaches into another's internals.

## Implementation

### The Main Interface

One class coordinates a request. It takes the source name, coordinates, and the units the customer wants, and every data point it exposes has already been through the converter:

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

The converter is built once per request from two inputs: the units the chosen source declares and the units the customer asked for. Nothing downstream of `currently` ever sees a raw vendor value.

### Source Adapters

Each external API gets one class inheriting from a base. The base fixes the interface: which units this source uses, how to return current conditions, and an optional hook for fetching in parallel.

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

A concrete adapter is mostly a field map from the vendor's structure to the internal one:

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

The adapter knows nothing about the customer's units. It reports its own, US with integer percentages here, and leaves conversion to the converter. The two sentinel constants are both `nil` at runtime; they exist so a field mapped to `DATA_MISSING` reads differently from one mapped to `DATA_PENDING`.

### Source Units vs Return Units

One unit system per source would not have been enough, because real vendors mix systems in one payload. So the units object takes a base system plus per-field overrides:

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

Source C is US for temperature and wind but metric for precipitation, and its declaration says so field by field. The converter trusts that declaration completely. It has no way to check it against the data.

### The Converter

The converter is a lookup table of pure functions keyed by `[from, to]` pairs, one table per measurement type:

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

When source and return units already match, the value passes through untouched. The same shape repeats for wind speed, pressure, visibility, precipitation, and the rest.

### Output Serializers with Alba

The last layer is formatting. [Alba](https://github.com/okuramasafumi/alba) serializers define the JSON structure, and the base output lists the fields every response carries:

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

Output variants inherit and add attributes rather than redefining the structure:

```ruby
class Api::Outputs::Full < Api::Outputs::Base
  association :currently do
    attributes :aqi, :uv_index, :pressure_trend  # Additional fields
  end
end
```

The serializer only ever sees converted values, so one class serves a request in any unit system. A minimal, full, or custom output is a subclass, not a branch in the coordinator.

## Data Flow

A request for source `foo` in SI units passes through the layers in order:

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

Adding a source is mechanical:

1. Create `Api::Sources::NewSource < Api::Sources::Base`
2. Implement data-fetching methods
3. Map external fields to internal shape
4. Declare `source_units` accurately
5. Add to the source registry

The converter, the outputs, and the existing sources do not change. Step 4 is where the pattern's risk sits. Declaring Celsius for a vendor that sends Fahrenheit produces plausible-looking wrong numbers, and nothing downstream can tell.

## Lessons Learned

- **Declare units per source and per field.** Assuming a vendor's units leads to subtle bugs, and a vendor can mix systems in one payload.
- **Fetch lazily, because external APIs charge per request.** Memoized accessors on the adapter mean only the data the output asks for is fetched.
- **Keep converters pure.** Same inputs, same outputs, no side effects, so each conversion is easy to test in isolation.
- **Use sentinels to say why a field is nil.** `DATA_MISSING`, `DATA_PENDING`, and `DATA_SKIPPED` all resolve to nil, but each records a different reason: unavailable, not yet implemented, or intentionally omitted.

---

## How This Post Was Made

**Prompt:** "create a post, use the post skill, and pr skill, do a writeup of the basics of the Api::Weather system, we have the standard interface / entrypoint, then source adapters, source_units, return_units, the converter, and outputs using alba. Review the readme carefully, especially the part about how to add new source adapters. Review Api::Weather and Api::Converter etc. provide concise examples to illustrate the system, but not complete code. We don't want to share all the 'secret sauce' here, but we want to provide the pattern for other people to consider for adaptation in their own projects. remember to save this prompt for the pr etc."

Generated by Claude using the blog-post-generator skill. Based on code review of a production weather API system with 10+ source adapters.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The Problem now opens on three vendors' mismatched units instead of a generic list of challenges, each Implementation section says why before the code and what the code cannot enforce after it, and Lessons Learned merged the two bullets that repeated the body. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.
