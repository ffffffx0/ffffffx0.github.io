---
layout: post
title: "Jekyll的本地环境配置及主题更换(rvm管理ruby)"
keywords: ["jekyll", "rvm","ruby","jekyll-bootstrap"]
description: "md语法测试"
category: "jekyll"
tags: ["jekyll", "rvm","ruby","jekyll-bootstrap"]
---

```
186  curl -sSL https://rvm.io/mpapis.asc | gpg2 --import -
187  curl -L get.rvm.io | bash -s stable
188  sed -i -e 's/ftp\.ruby-lang\.org\/pub\/ruby/ruby\.taobao\.org\/mirrors\/ruby/g' ~/.rvm/config/db
189  sed -i 's!cache.ruby-lang.org/pub/ruby!ruby.taobao.org/mirrors/ruby!'  /usr/local/rvm/config/db 
190  cat /etc/profile.d/rvm.sh
191  source /etc/profile.d/rvm.sh
192  rvm requirements
193  rvm install ruby 2.0.0
194  gem
195  gem -v
196  gem install jekyll
197  rvm use 2.0.0 --default
198  rvm use 2.0.0
199  rvm gemset create rails416
200  rvm use 2.0.0
201  rvm use 2.0.0@rails416
202  gem install rails -v 4.1.6 --no-rdoc --no-ri
203  gem install jekyll
204  gem update --system 
205  vi /etc/hosts
206  history
207  gem install jekyll
208  history
209  vi /etc/hosts
210  gem install rails -v 4.1.6 --no-rdoc --no-ri
211  gem install jekyll
212  gem sources --remove https://rubygems.org/
213  gem sources -a https://ruby.taobao.org/
214  gem sources -l   
215  gem update --system  
216  gem install jekyll
217  gem install rdiscount
218  ls
219  jekyll serve
220  gem install therubyracer
221  jekyll serve
222  ls
223  service iptablse status
224  service iptables status
225  jekyll serve
226  ls
227  cd /data/solr-5.2.1/jekyll/jekyll-bootstrap
228  ls
229  jekyll serve
230  /etc/init.d/iptables status
231  curl http://172.16.82.186:4000
232  curl http://127.0.0.1:4000
233  jekyll serve
234  ls
235  cat _config.yml 
236  cat _config.yml |grep local
237  jekyll serve
238  rake
239  ls
240  rake theme:install git="git://github.com/sodabrew/theme-dinky.git"
241  rake theme:install git="git://github.com/sodabrew/theme-dinky.git
242  rake theme:install git="git://github.com/sodabrew/theme-dinky.git"
243  lsd
244  ls
245  cd ../
246  ls
247  mv jekyll-bootstrap jekyll-bootstrapbak
248  cd jekyll-bootstrapbak/
249  ls
250  ls _theme_packages/
251  ls _theme_packages/dinky/
252  ls _theme_packages/dinky/_includes/
253  ls _theme_packages/dinky/_includes/themes/
254  ls _theme_packages/dinky/_includes/themes/dinky/
255  ls _theme_packages/dinky/assets/
256  ls _theme_packages/dinky/assets/themes/dinky/css/
257  ls _theme_packages/dinky/assets/themes/dinky/css/styles.css 
258  cat  _theme_packages/dinky/assets/themes/dinky/css/styles.css
```
