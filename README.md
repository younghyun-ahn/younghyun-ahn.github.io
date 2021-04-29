## younghyun-ahn.github.io

### Init
```bash
> git clone git@github.com:younghyun-ahn/younghyun-ahn.github.io.git
> jekyll new . --force
> bundle exec jekyll serve --livereload
```

```ruby
Gemfile

# gem "jekyll", "~> 4.2.0"
gem "github-pages", group: :jekyll_plugins 
```

```bash
> bundle update
> bundle install
```

```yml
_config.yml

title: Lake log
email: younghyun.ahn@gmail.com

baseurl: "" # the subpath of your site, e.g. /blog
url: "https://younghyun-ahn.github.io" # the base hostname & protocol for your site, e.g. http://example.com
github_username: younghyun-ahn
```

```bash
> bundle exec jekyll build
> bundle exec jekyll serve
```

### Regular theme instead of Gem-based theme
open $(bundle info --path minima)
copy directory - _includes, _layouts, _sass, assets

```yml
_confit.yml

# theme: minima
```


```ruby
Gemfile

# gem "minima", "~> 2.5"
```

```bash
bundle update
```


## minima-reboot

```bash
sudo gem install bundler:1.12
bundle _1.12_ update
bundle _1.12_ install
bundle exec jekyll _1.12_ build
bundle exec jekyll serve
```

Gemfile
```ruby
gem "kramdown-parser-gfm"
```