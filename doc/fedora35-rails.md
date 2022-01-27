# Ruby On Rails on Fedora 35

## fedora35_00

```bash
# bare fedora 35 server
```

## fedora35_01
```bash
sudo dnf update

# install dependencies for Ruby.
sudo dnf install git git-core zlib zlib-devel gcc-c++ patch readline readline-devel libyaml-devel libffi-devel openssl-devel make bzip2 autoconf automake libtool bison curl sqlite-devel

# install rbenv
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
source ~/.bashrc
exec $SHELL

git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
echo 'export PATH="$HOME/.rbenv/plugins/ruby-build/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
exec $SHELL

# install ruby
rbenv install 3.0.1
rbenv global 3.0.1
ruby -v

# dont install the documentation for each package locally
echo "gem: --no-ri --no-rdoc" > ~/.gemrc

# install bundler
gem install bundler

# rehash after installing a new ruby version/gem (to load executables fo rbenv)
rbenv rehash

# install mariadb
sudo dnf install -y mariadb-server
sudo systemctl start mariadb
sudo systemctl status mariadb
sudo systemctl enable mariadb
sudo mysql_secure_installation

# install rails dependencies
sudo dnf install mysql-devel
sudo dnf install nodejs
sudo npm install --global yarn

# install tools
sudo dnf install vim
```

## fedora35_02

```bash
mkdir project && cd $_
echo "source 'https://rubygems.org'
ruby   '3.0.1'
gem    'rails', '~> 7.0.0'
" >> Gemfile

bundle install

bundle exec rails new . --force --database=mysql --css=bootstrap

bundle exec rails s

# http://localhost:3000/
```
