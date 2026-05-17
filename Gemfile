source "https://rubygems.org"

# GitHub Pages built-in: pin to the version GitHub Pages currently serves.
# This locks Jekyll, minima, and all plugins to exactly what github.com/pages
# is running, so what builds locally matches production.
gem "github-pages", group: :jekyll_plugins

group :jekyll_plugins do
  gem "jekyll-feed"
  gem "jekyll-sitemap"
  gem "jekyll-seo-tag"
end

# Windows/JRuby compat (no-op on Linux but kept for portability)
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end
