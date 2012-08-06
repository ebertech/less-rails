# The dynamic stylesheet language for the Rails asset pipeline, backported.

This gem provides integration for Rails 2.3 projects using the Less stylesheet language in the Rails (Sprockets, really) asset pipeline.

## Important backport note

This backport is completely unofficial (whatever official may mean in a DVCS world). You may consider using [sprockets-less](https://github.com/lloeki/sprockets-less) instead anyway.

This backport is meant as to ease the transition for someone's application from 2.x to 3.x and up. The more work you do early, the less you have to do at once, and the easier the transition becomes. Being able to use the asset pipeline is one of those things, and being able to use Less right now is another. Sadly, less-rails 1.x requires the less 1.2 gem (which is old and in pure ruby), not less 2.x (which uses the shiny and much more featured less.js) Besides, this old less-rails does not integrate at all with Sprockets.

## Installing

Once you have set up Sprockets to behave on Rails 2.3 (see RAILS_UP for how to uprate your app), just bundle up less-rails in your Gemfile. This will pull in less as a runtime dependency too. This gem shall not be published at rubygems under this name, so be sure to set up both git and the branch correctly.

    gem 'less-rails', :git => https://github.com/lloeki/less-rails.git, :branch => 'rails-2.3-backport'



## Configuration

You have to setup this gem so that Sprockets knows about Less, and where to look for files. This was the role of the Railtie. The following `config/initializers/sprockets_less.rb` will do the trick:

```ruby
require 'less'
require 'less-rails'
Sprockets::Engines #force autoloading
Sprockets.register_engine '.less', Less::Rails::LessTemplate
Sprockets.env.register_preprocessor 'text/css', Less::Rails::ImportProcessor

module Less
  module Sprockets
    module LessContext
      attr_accessor :less_config
    end
  end
end

less = ActiveSupport::OrderedOptions.new
less.paths = []
less.compress = false

Sprockets.env.context_class.extend(Less::Sprockets::LessContext)
Sprockets.env.context_class.less_config = less
```

#### About Compression

Less has real basic compression and it is recommended that you set the rails `config.assets.css_compressor` to something more stronger like YUI.



## Import Hooks

Any `@import` to a `.less` file will automatically declare that file as a sprockets dependency to the file importing it. This means that you can edit imported framework files and see changes reflected in the parent durning development. So this:

```css
@import "frameworks/bootstrap/mixins";

#leftnav { .border-radius(5px); }
```

Will end up acting as if you had done this below:

```css
/*
 *= depend_on "frameworks/bootstrap/mixins.less"
*/

@import "frameworks/bootstrap/mixins";

#leftnav { .border-radius(5px); }
```



## Helpers

When referencing assets use the following helpers in LESS.

```css
asset-path(@relative-asset-path)  /* Returns a string to the asset. */
asset-path("rails.png")           /* Becomes: "/assets/rails.png" */

asset-url(@relative-asset-path)   /* Returns url reference to the asset. */
asset-url("rails.png")            /* Becomes: url(/assets/rails.png) */
```

As a convenience, for each of the following asset classes there are corresponding `-path` and `-url` helpers image, font, video, audio, javascript and stylesheet. The following examples only show the `-url` variants since you get the idea of the `-path` ones above.

```css
image-url("rails.png")            /* Becomes: url(/assets/rails.png) */
font-url("rails.ttf")             /* Becomes: url(/assets/rails.ttf) */
video-url("rails.mp4")            /* Becomes: url(/videos/rails.mp4) */
audio-url("rails.mp3")            /* Becomes: url(/audios/rails.mp3) */
javascript-url("rails.js")        /* Becomes: url(/assets/rails.js) */
stylesheet-url("rails.css")       /* Becomes: url(/assets/rails.css) */
```

Lastly, we provide a data url method for base64 encoding assets.

```css
asset-data-url("rails.png")       /* Becomes: url(data:image/png;base64,iVBORw0K...) */
```

Please note that these helpers are only available server-side, and something like ERB templates should be used if client-side rendering is desired.



## Generators

Generators are dropped in this backport.

## Testing

Testing is dropped in this backport.

## License

Less::Rails is Copyright (c) 2011 Ken Collins, <ken@metaskills.net> and is distributed under the MIT license.

