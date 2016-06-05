# blog.hilzu.moe

```sh
# Ensure recent version of ruby
$ ruby --version
ruby 2.3.1p112 (2016-04-26 revision 54768) [x86_64-darwin15]

# Update bundler
$ gem update bundler

# Update dependecies
$ bundle update

# Serve locally
$ bundle exec jekyll serve

# Deploy
$ ansible-playbook -K ansible/deploy.yml
```
