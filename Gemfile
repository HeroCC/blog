# If you do not have OpenSSL installed, change
# the following line to use 'http://'
source 'https://rubygems.org'

# For faster file watcher updates on Windows:
gem 'wdm', '~> 0.1.0', platforms: [:mswin, :mingw]

# Windows does not come with time zone data
gem 'tzinfo-data', platforms: [:mswin, :mingw, :jruby]

# Middleman Gems
gem "middleman", "~> 4.1"
gem "middleman-blog"
gem "middleman-livereload"
gem "middleman-syntax" # Code Highlighting

gem 'redcarpet', '~> 3.3', '>= 3.3.3'
gem 'nokogiri'

# For feed.xml.builder
gem "builder", "~> 3.0"

group :production do
  gem "middleman-sitemap" # Automatic Sitemap Generation
  gem "puma" # Heroku Webhosting
end

gem "rack-contrib"

# Actual Site Libs
gem 'font-awesome-sass', '~> 4.6.2'
