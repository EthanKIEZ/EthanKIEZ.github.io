source 'https://rubygems.org'

# Jekyll 与 GitHub Pages 主题依赖
gem "jekyll", "~> 3.9"
gem "minimal-mistakes-jekyll"

# Ruby 3.4+ 默认 gems 兼容
gem "base64"
gem "bigdecimal"
gem "csv"
gem "drb"

# GitHub Pages 兼容插件
group :jekyll_plugins do
  gem "kramdown-parser-gfm"
  gem "jekyll-paginate"
  gem "jekyll-sitemap"
  gem "jekyll-gist"
  gem "jekyll-feed"
  gem "jekyll-include-cache"
  gem "jekyll-remote-theme"
end

# Windows 平台相关依赖
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

# Performance-booster for watching directories on Windows
gem "wdm", "~> 0.1", :platforms => [:mingw, :x64_mingw, :mswin]

# Lock `http_parser.rb` gem to `v0.6.x` on JRuby builds
gem "http_parser.rb", "~> 0.6.0", :platforms => [:jruby]

# Webrick for Ruby 3.x local serving
gem "webrick", "~> 1.8"
