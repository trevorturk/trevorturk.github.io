---
layout: post
title: "One Rails App, Two Products"
date: 2026-02-27 10:00:00 -0600
summary: "One Rails app serves both Hello Weather and WeatherMachine. A routing constraint reads the request's host and picks the product, so there's one deployment and one weather data pipeline, and splitting them later is a config change."
tags: [rails, architecture, routing]
model: "Claude Opus 4.5"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

Two products on two domains share one weather data pipeline:

- **[Hello Weather](https://helloweather.com)** - Consumer iOS app with API, marketing site, and blog
- **[WeatherMachine](https://weathermachine.io)** - B2B weather API with dashboard, docs, and marketing

Both products need the same weather models, the same fetching code, and the same API. Two Rails apps would mean two deployments, and every change to the shared code would have to be copied across by hand. We're a small team and didn't want to carry that.

## The Solution

We run one Rails app, and it reads the request's host to decide which product to show. A request to `helloweather.com` gets the consumer site. A request to `weathermachine.io` gets the B2B site. Underneath, the models, fetching, and caching are one copy that both products share. Only the pages and the styling differ.

### The Routing

```ruby
# config/routes.rb

class ApiConstraint
  def matches?(request)
    ENV["API"] == "true" || request.host =~ /weathermachine/
  end
end

Rails.application.routes.draw do
  # WeatherMachine routes (when host matches)
  constraints(ApiConstraint.new) do
    scope module: :dashboard do
      get "/", to: "marketing#index"
      get "docs", to: "marketing#docs"
      # ...
    end
  end

  # Hello Weather routes (default)
  get "/", to: "v4/marketing#index", as: :root
  # ...
end
```

`ApiConstraint` says yes when the host contains `weathermachine`. It also says yes when `ENV["API"]` is set, which is how we work on the [WeatherMachine](https://weathermachine.io) side in development. The routes inside the `constraints` block only match when it says yes. Only WeatherMachine's marketing pages live in that block. The signed-in dashboard, the API, and the Hello Weather pages answer on any host.

### Directory Structure

Each product gets its own controller namespace, which in Rails means a folder and a module prefix. There are three: `dashboard/` for [WeatherMachine](https://weathermachine.io), `v4/` for the current [Hello Weather](https://helloweather.com) site, and `api/` for the forecast endpoints both products call. Views and layouts are split the same way.

### Product-Specific Behavior

No view checks the host. The routing constraint picks the controller namespace, and each namespace names its own layout:

```ruby
# app/controllers/dashboard/base_controller.rb
class Dashboard::BaseController < ApplicationController
  layout "dashboard"
end

# app/controllers/v4/marketing_controller.rb
class V4::MarketingController < ApplicationController
  layout "v4"
end
```

The layout pulls in that product's stylesheet, script, and navigation:

```erb
<%# app/views/layouts/dashboard.html.erb %>
<%= stylesheet_link_tag "dashboard", media: "all" %>
<%= javascript_include_tag "dashboard" %>
<%= render "dashboard/nav" %>

<%# app/views/layouts/v4.html.erb %>
<%= stylesheet_link_tag "v4", media: "all" %>
<%= javascript_include_tag "marketing" %>
<%= render "v4/marketing/nav" %>
```

### Assets

Each product has its own stylesheet and its own script bundle. A `shared/` folder holds the SCSS both stylesheets import:

```
app/assets/
├── stylesheets/
│   ├── dashboard.scss      # WeatherMachine
│   ├── dashboard/
│   ├── v4.scss             # Hello Weather
│   ├── marketing/
│   └── shared/             # mixins, colors, utilities
└── javascripts/
    ├── dashboard.js        # WeatherMachine
    └── marketing.js        # Hello Weather
```

## Why This Works for Small Teams

We run one of everything. One Heroku app, one deployment pipeline, one Postgres database that also holds the cache and the job queue, one set of environment variables, one error tracker, one log stream. `git push production main` deploys both products at once. A bug fix in shared code reaches both with the same deploy. One team can look after all of it.

### Trade-offs

| Benefit | Cost |
|---------|------|
| Shared infrastructure | Products can't scale independently |
| One codebase | Namespace discipline required |
| Single deployment | Changes affect both products |
| Code reuse | Coupling between products |

For us the benefits win. We'd think again if the API needed to scale on its own, or if different teams owned the two products.

## The Bigger Picture

We started with one app because splitting it later is easy, and merging two apps back into one is not.

The routing constraint already makes the split cheap. If [WeatherMachine](https://weathermachine.io) grows and needs its own deployment, we can:

1. Deploy the same codebase to a new Heroku app
2. Set `ENV["API"] = "true"` on that app
3. Point `weathermachine.io` DNS to the new app

That takes no code changes, because `ApiConstraint` treats `ENV["API"]` the same as a matching host.

## Lessons Learned

- **Namespace by product from the first controller.** `dashboard/`, `v4/`, and `api/` keep the two products out of each other's way in one `app/` tree.
- **Share the pipeline, not the presentation.** Models, jobs, and services are one copy. Stylesheets are one per product.
- **Test both hosts.** Our integration test base class sets `host!` to the Hello Weather domain, and a dashboard subclass sets it to the WeatherMachine one. Without the second class, the constrained routes never get tested.
- **Group and comment routes by product.** One routes file for two products gets confusing without it.
