# https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/creating-a-github-pages-site-with-jekyll

source "https://rubygems.org"

gem "github-pages", "~> 228", group: :jekyll_plugins
gem "minima", "~> 2.5.1"

group :jekyll_plugins do
  gem "jekyll-feed", "~> 0.15.1"
end

gem "webrick", "~> 1.8"

# https://jekyllrb.com/docs/installation/windows/

# Windows and JRuby does not include zoneinfo files, so bundle the tzinfo-data gem
# and associated library.
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

# Performance-booster for watching directories on Windows
gem 'wdm', '~> 0.1.1', :install_if => Gem.win_platform?
