source "https://rubygems.org"

# Jekyll and default Minima theme
gem "jekyll", "~> 4.3.4"
gem "minima", "~> 2.5"

# Plugins
group :jekyll_plugins do
  gem "jekyll-feed", "~> 0.17"
  gem "jekyll-sitemap", "~> 1.4"
  gem "jekyll-scholar", "~> 7.1"
end

# Kramdown GFM parser (required for GFM in Jekyll 4 / Kramdown 2)
gem "kramdown-parser-gfm"

# Webrick was removed in Ruby 3.0 standard library, needed for local 'jekyll serve'
gem "webrick", "~> 1.8"

# Windows-specific gems (declared with platforms: so they are safely ignored on macOS and Linux)
gem "tzinfo-data", platforms: [:mingw, :mswin, :x64_mingw, :jruby, :ucrt]
gem "wdm", ">= 0.1.0", platforms: [:mingw, :mswin, :x64_mingw, :ucrt]
