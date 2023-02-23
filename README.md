# build

```bash
# install: https://jekyllrb.com/docs/installation/

# change ruby source:
gem sources --remove https://rubygems.org/
gem sources -a https://gems.ruby-china.com/
# install jekyll
gem install jekyll bundler
bundle install

# serve
bundle exec jekyll serve
# gh-pages
jekyll build
echo "" > _site/.nojekyll
cd _site

# local gem
bundle update jekyll-artisync
```