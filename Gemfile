source 'https://ruby.taobao.org/'

require 'json'
require 'open-uri'
versions = JSON.parse(open('https://pages.github.com/versions.json').read)

gem 'github-pages', versions['github-pages']
gem 'rake'

gem 'wdm', '>= 0.1.0' if Gem.win_platform?
#gem 'pygments', '0.5.0'
