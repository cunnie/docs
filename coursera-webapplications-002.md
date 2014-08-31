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

### Module 3 / Quizzes / Assignment

To tie objects together with foreign keys, you need to add the following to the model:

* one-to-one:  `has_one`, `belongs_to`
* many-to-one: `has_many`, `belongs_to`
* many-to-many: `has_and_belongs_to_many` (both)

#### Configure a Many-to-One Relationship
For example, to express a many-to-one relationship between comments and posts, do the following:

* edit `app/models/post`

```
class Post < ActiveRecord::Base
  has_many :comments, dependent: :destroy
end
```
(by the way, `dependent: destroy` is an associative hash that's passed as an argument to `has_many`)

* edit `app/models/comment`

```
class Comment < ActiveRecord::Base
  belongs_to :post
end
```

#### Nest Comments under Posts
To nest the comments *under* the posts, edit `config/routes.rb`:

```
Rails.application.routes.draw do
  resources :comments

  resources :posts do
    resources :comments
  end
```

#### Validate Data
We validate data in the models. For example, to make sure that every post has a title and a body, we edit `app/models/post.rb`:

```
class Post < ActiveRecord::Base
  has_many :comments, dependent: :destroy
  validates_presence_of :title, :body
end
```
Similarly for `app/models/comment.rb`:

```
class Comment < ActiveRecord::Base
  belongs_to :post
  validates_presence_of :post_id, :body
end
```

#### Submit Assignment
Submit assignments here: [http://fdisk.co/mooc](http://fdisk.co/mooc).

#### Observations
He consistently confuses forward-slash ("/") and backward-slash ("\").  Look at the screen&mdash;do **not** listen to him&mdash;when he uses the word "slash".

I will try do the exercises along with him the first time I listen to a lecture because I'll probably have to do them along with him later when I work on the assignment.

Module 3's assignment is Assignment 2, **not** Assignment 3.
