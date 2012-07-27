# Uprating your Ruby Rails 2.3 application

This will help migrating to Rails 3.x and Ruby 1.9.

## Bundler

See the [official unofficial method](http://gembundler.com/rails23.html)

## Thin

Sick of WEBrick, better than Mongrel and above all, works on both 1.8 and 1.9.

Add thin to the Gemfile.

```ruby
# config/boot.rb
# ---8<---
# Thin by default
if $0 == "script/server"
  require 'rack'
  module Rack::Handler
    puts "Patching Rack for Thin as default"
    class << self
      alias_method :get_without_thin_as_default, :get
    end

    def self.get(server)
      self.get_without_thin_as_default(server) or self.get_without_thin_as_default("thin")
    end
  end unless Rack::Handler.respond_to? :get_without_thin_as_default
end
```

## Sprockets

Served from {vendor,app}/assets/{javascripts/stylesheets} and whatnot.

### Install gems

```ruby
# Gemfile
# ---8<---
gem 'yui-compressor'
gem 'sprockets', '~> 2.0'

group :assets do
    # pick your evil
    gem 'sass'
    gem 'coffee-script'
    gem 'haml'
end

gropo development, test do
    gem 'sprockets-sass'
end
```

### Initialize Sprockets

```ruby
# config/initializers/sprockets.rb
module Sprockets
  def self.env
    @env ||= begin
      sprockets = Sprockets::Environment.new
      sprockets.append_path 'app/assets/javascripts'
      sprockets.append_path 'vendor/assets/javascripts'
      sprockets.append_path 'app/assets/stylesheets'
      sprockets.append_path 'vendor/assets/stylesheets'
      begin
        sprockets.css_compressor  = YUI::CssCompressor.new
        sprockets.js_compressor   = YUI::JavaScriptCompressor.new
      rescue NameError
      end unless Rails.env.development?
      sprockets
    end
  end

  def self.manifest
    @manifest ||= Sprockets::Manifest.new(env, Rails.root.join("public", "assets", "manifest.json"))
  end
end

# This controls whether or not to use "debug" mode with sprockets.  In debug mode,
# asset tag helpers (like javascript_include_tag) will output the non-compiled
# version of assets, instead of the compiled version.  For example, if you had
# application.js as follows:
# 
# = require jquery
# = require event_bindings
# 
# javascript_include_tag "application" would output:
# <script src="/assets/jquery.js?body=1"></script>
# <script src="/assets/event_bindings.js?body=1"></script>
# 
# If debug mode were turned off, you'd just get
# <script src="/assets/application.js"></script>
# 
# Here we turn it on for all environments but Production
Rails.configuration.action_view.debug_sprockets = true unless Rails.env.production?
```

### Start Sprockets with Rack

```ruby
# config.ru
# we need to protect against multiple includes of the Rails environment (trust me)
require File.dirname(__FILE__) + '/config/environment' if !defined?(Rails) || !Rails.initialized?
require 'sprockets'

unless Rails.env.production?
  map '/assets' do
    sprockets = Sprockets.env
    run sprockets
  end
end

map '/' do
  run ActionController::Dispatcher.new
end
```


## Enhance Rails with Sprockets-aware helpers

```ruby
# config/initializers/sprockets_asset_tag_helper.rb
# Asset Tag Helper that uses Sprockets
module ActionView
  module Helpers
    module AssetTagHelper

      # Overwrite the javascript_path method to use the 'assets' directory
      # instead of the default 'javascripts' (Sprockets will figure it out)
      def javascript_path(source, options)
        path = compute_public_path(source, 'assets', options.merge(:ext => 'js'))
        options[:body] ? path + "?body=1" : path
      end
      alias_method :path_to_javascript, :javascript_path # aliased to avoid conflicts with a javascript_path named route


      # Overwrite the stylesheet_path method to use the 'assets' directory
      # instead of the default 'stylesheets' (Sprockets will figure it out)
      def stylesheet_path(source, options)
        path = compute_public_path(source, 'assets', options.merge(:ext => 'css'))
        options[:body] ? path + "?body=1" : path
      end
      alias_method :path_to_stylesheet, :stylesheet_path # aliased to avoid conflicts with a stylesheet_path named route

      # Overwrite the stylesheet_link_tag method to expand sprockets files if 
      # debug mode is turned on.  Never cache files (like the default Rails 2.3 does).
      # 
      def stylesheet_link_tag(*sources)
        options = sources.extract_options!.stringify_keys
        debug   = options.key?(:debug) ? options.delete(:debug) : debug_assets?

        sources.map do |source|
          if debug && !(digest_available?(source, 'css')) && (asset = asset_for(source, 'css'))
            asset.to_a.map { |dep| stylesheet_tag(dep.logical_path, { :body => true }.merge(options)) }
          else
            sources.map { |source| stylesheet_tag(source, options) }
          end
        end.uniq.join("\n").html_safe
      end

      # Overwrite the javascript_include_tag method to expand sprockets files if 
      # debug mode is turned on.  Never cache files (like the default Rails 2.3 does).
      #
      def javascript_include_tag(*sources)
        options = sources.extract_options!.stringify_keys
        debug   = options.key?(:debug) ? options.delete(:debug) : debug_assets?

        sources.map do |source|
          if debug && !(digest_available?(source, 'js')) && (asset = asset_for(source, 'js'))
            asset.to_a.map { |dep| javascript_src_tag(dep.logical_path, { :body => true }.merge(options)) }
          else
            sources.map { |source| javascript_src_tag(source.to_s, options) }
          end
        end.uniq.join("\n").html_safe
      end

      private

      def javascript_src_tag(source, options)
        body = options.has_key?(:body) ? options.delete(:body) : false
        content_tag("script", "", { "type" => Mime::JS, "src" => path_to_javascript(source, :body => body) }.merge(options))
      end

      def stylesheet_tag(source, options)
        body = options.has_key?(:body) ? options.delete(:body) : false
        tag("link", { "rel" => "stylesheet", "type" => Mime::CSS, "media" => "screen", "href" => html_escape(path_to_stylesheet(source, :body => body)) }.merge(options), false, false)
      end

      def debug_assets?
        Rails.configuration.action_view.debug_sprockets || false
      end

      # Add the the extension +ext+ if not present. Return full URLs otherwise untouched.
      # Prefix with <tt>/dir/</tt> if lacking a leading +/+. Account for relative URL
      # roots. Rewrite the asset path for cache-busting asset ids. Include
      # asset host, if configured, with the correct request protocol.
      def compute_public_path(source, dir, options = {})
        source = source.to_s
        return source if is_uri?(source)

        source = rewrite_extension(source, options[:ext]) if options[:ext]
        source = rewrite_asset_path(source, dir, options)
        source = rewrite_relative_url_root(source, ActionController::Base.relative_url_root)
        source = rewrite_host_and_protocol(source, options[:protocol])
        source
      end

      def rewrite_relative_url_root(source, relative_root_url)
        relative_root_url && !(source =~ Regexp.new("^" + relative_root_url + "/")) ? relative_root_url + source : source
      end

      def has_request?
        @controller.respond_to?(:request)
      end

      def rewrite_host_and_protocol(source, porotocol = nil)
        host = compute_asset_host(source)
        if has_request? && !host.blank? && !is_uri?(host)
          host = @controller.request.protocol + host
        end
        host ? host + source : source
      end

      # Check for a sprockets version of the asset, otherwise use the default rails behaviour.
      def rewrite_asset_path(source, dir, options = {})
        if source[0] == ?/
          source
        else
          source = digest_for(source.to_s)
          source = Pathname.new("/").join(dir, source).to_s
          source
        end
      end

      def digest_available?(logical_path, ext)
        (manifest = Sprockets.manifest) && (manifest.assets["#{logical_path}.#{ext}"])
      end

      def digest_for(logical_path)
        if (manifest = Sprockets.manifest) && (digest = manifest.assets[logical_path])
          digest
        else
          logical_path
        end
      end

      def rewrite_extension(source, ext)
        if ext && File.extname(source) != ".#{ext}"
          "#{source}.#{ext}"
        else
          "#{source}"
        end
      end

      def is_uri?(path)
        path =~ %r{^[-a-z]+://|^(?:cid|data):|^//}
      end

      def asset_for(source, ext)
        source = source.to_s
        return nil if is_uri?(source)
        source = rewrite_extension(source, ext)
        Sprockets.env[source]
      rescue Sprockets::FileOutsidePaths
        nil
      end
    end
  end
end
```

### Rake tasks

Let's have some `rake assets:precompile` fun.

```ruby
# lib/tasks/assets.rake
require "fileutils"
require 'pathname'

namespace :assets do
  def ruby_rake_task(task, fork = true)
    env    = ENV['RAILS_ENV'] || 'production'
    groups = ENV['RAILS_GROUPS'] || 'assets'
    args   = [$0, task, "RAILS_ENV=" + env, "RAILS_GROUPS=" + groups]
    args << "--trace" if Rake.application.options.trace
    if $0 =~ /rake\.bat\Z/i
      Kernel.exec $0, *args
    else
      fork ? ruby(*args) : Kernel.exec(FileUtils::RUBY, *args)
    end
  end

  # We are currently running with no explicit bundler group
  # and/or no explicit environment - we have to reinvoke rake to
  # execute this task.
  def invoke_or_reboot_rake_task(task)
    if ENV['RAILS_GROUPS'].to_s.empty? || ENV['RAILS_ENV'].to_s.empty?
      ruby_rake_task task
    else
      Rake::Task[task].invoke
    end
  end

  desc "Compile all the assets named in config.assets.precompile"
  task :precompile do
    invoke_or_reboot_rake_task "assets:precompile:all"
  end

  namespace :precompile do
    def internal_precompile(digest = nil)

      # Ensure that action view is loaded and the appropriate
      # sprockets hooks get executed
      _ = ActionView::Base

      sprockets = Sprockets.env
      manifest_path = Pathname.new(Rails.public_path).join("assets", "manifest.json")

      manifest = Sprockets.manifest
      manifest.compile
    end

    task :all do
      Rake::Task["assets:precompile:primary"].invoke
    end

    task :primary => ["assets:environment", "tmp:cache:clear"] do
      internal_precompile
    end

    task :nondigest => ["assets:environment", "tmp:cache:clear"] do
      internal_precompile(false)
    end
  end

  desc "Remove compiled assets"
  task :clean do
    invoke_or_reboot_rake_task "assets:clean:all"
  end

  namespace :clean do
    task :all => ["assets:environment", "tmp:cache:clear"] do
      public_asset_path = Pathname.new(Rails.public_path).join("assets")
      rm_rf public_asset_path, :secure => true
    end
  end

  task :environment do
    Rake::Task["environment"].invoke
  end
end
```

[0] http://jaredonline.github.com/blog/2012/05/16/sprockets-2-with-rails-2-dot-3/
