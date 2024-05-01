source "https://rubygems.org"

gem "jekyll"

# gem "minimal"
gem "minimal-mistakes-jekyll"

gem "rake"

group :jekyll_plugins do
  gem "jekyll-feed", "~> 0.12"
  gem "jekyll-archives", "~> 2.2.1"
  
  gem "jekyll-sitemap"
  gem "jekyll-gist"
  gem "jekyll-include-cache"
end

# Windows and JRuby does not include zoneinfo files, so bundle the tzinfo-data gem
# and associated library.
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", "~> 1.2"
  gem "tzinfo-data"
end

gem "faraday-retry"

# Performance-booster for watching directories on Windows
gem "wdm", "~> 0.1.1", :platforms => [:mingw, :x64_mingw, :mswin]

# Lock `http_parser.rb` gem to `v0.6.x` on JRuby builds since newer versions of the gem
# do not have a Java counterpart.
gem "http_parser.rb", "~> 0.6.0", :platforms => [:jruby]

gem "webrick", group: :development
