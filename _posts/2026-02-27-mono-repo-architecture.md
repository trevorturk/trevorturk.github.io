---
layout: post
title: "One Rails App, Two Products"
date: 2026-02-27 10:00:00 -0600
summary: "Host-based routing serves Hello Weather and WeatherMachine from one Rails app, one deployment, and one weather data pipeline, with the split into separate apps already planned for."
tags: [rails, architecture, routing]
model: "Claude Opus 4.5"
last_edited: 2026-09-01
last_edited_by: "Claude Fable 5.1"
---

## The Problem

Two products on two domains share one weather data pipeline:

- **[Hello Weather](https://helloweather.com)** - Consumer iOS app with API, marketing site, and blog
- **[WeatherMachine](https://weathermachine.io)** - B2B weather API with dashboard, docs, and marketing

Both need the same pipeline, the same API layer, and the same models and business logic. Separate Rails apps would mean duplicated infrastructure, shared code kept in sync by hand, and two deployments. For a small team, that is overhead we don't need.

## The Solution

One Rails app serves both products, and the request's host decides which one. A request to `helloweather.com` gets the consumer experience. A request to `weathermachine.io` gets the B2B experience. Same codebase, same deployment, different UX. The pipeline is one copy, with the same models, fetching logic, and caching behind both products. Only the presentation layer differs.

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

`ApiConstraint` sends a request to [WeatherMachine](https://weathermachine.io) when `ENV["API"]` is set (useful in development) or the host matches (production). Routes inside `constraints(ApiConstraint.new)` apply only then. Only WeatherMachine's marketing pages sit inside it; the signed-in dashboard, the API, and the Hello Weather pages answer on any host.

### Directory Structure

Controllers are namespaced by product: `dashboard/` for [WeatherMachine](https://weathermachine.io), `v4/` for the current [Hello Weather](https://helloweather.com) site, and `api/` for the forecast endpoints both products call. Views and layouts follow the same split.

### Product-Specific Behavior

Nothing in a view checks the host. The constraint picks the controller namespace, and each namespace declares its layout:

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

Each product has its own stylesheet and script bundle. A `shared/` directory holds the mixins and utilities both stylesheets import:

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

Everything operational is singular. One Heroku app, one deployment pipeline, one Postgres database that also backs the cache and the job queue, one set of environment variables, one error tracker, one logging pipeline. A single `git push production main` deploys both products. A bug fix in shared code reaches both immediately. One team can maintain everything.

### Trade-offs

| Benefit | Cost |
|---------|------|
| Shared infrastructure | Products can't scale independently |
| One codebase | Namespace discipline required |
| Single deployment | Changes affect both products |
| Code reuse | Coupling between products |

For a small team, the benefits outweigh the costs. If the API had to scale independently, or separate teams owned the two products, we would reconsider.

## The Bigger Picture

Start simple, split when necessary. You can always extract a service later. You can't easily un-extract one.

The routing constraint already leaves the door open. If [WeatherMachine](https://weathermachine.io) grows and needs its own deployment, we can:

1. Deploy the same codebase to a new Heroku app
2. Set `ENV["API"] = "true"` on that app
3. Point `weathermachine.io` DNS to the new app

No code changes required. The routing constraint already handles it.

## Lessons Learned

- **Namespace by product from the first controller.** `dashboard/`, `v4/`, and `api/` keep two products from colliding in one `app/` tree.
- **Share the pipeline, not the presentation.** Models, jobs, and services are one copy. Stylesheets are one per product.
- **Test both hosts.** The integration test base class sets `host!` to the Hello Weather domain, and a dashboard subclass sets it to the WeatherMachine one, or the constrained routes go untested.
- **Group and comment routes by product.** One routes file for two products gets confusing without it.
