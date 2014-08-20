## Coursera Web Applications

Course [home page](https://class.coursera.org/webapplications-002)

### Modules 1 - 2 / Quizzes / Assignment

```
gem install bundler
gem install rails
rails new blog
cd blog
bundle exec rails server # or `bundle exec rails s`
```
At this point you can browse to your new rails webserver [http://localhost:3000/](http://localhost:3000/).

Add scaffolding:

```
^C
bundle exec rails generate scaffold post title:string body:text
 # if the previous command hangs, look for a process `spring` and kill it
bundle exec rails generate scaffold comment post_id:integer body:text
bundle exec rails s
```
At this point you can browse to your new rails webserver [http://localhost:3000/](http://localhost:3000/), but you'll see *ActiveRecord::PendingMigrationError*

```
^C
bundle exec rake db:migrate RAILS_ENV=development
bundle exec rails s
```
Then it should look OK: [http://localhost:3000/](http://localhost:3000/)

#### Set up BitBucket Account and blog repo

```
git remote add origin git@bitbucket.org:brian_cunnie/blog.git
git push -u origin --all # pushes up the repo and its refs for the first timegit push -u origin --tags # pushes up any tags
```
