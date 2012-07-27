eval File.read('Gemfile.rails-2.3')

group :test do
  if RUBY_PLATFORM =~ /darwin/
    gem 'rb-fsevent'
    gem 'growl'
  end
end
