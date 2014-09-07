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

### Module 5 Assignment
Module 5's assignment is **Assignment 3**; don't let the inconsistent numbering scheme throw you off.

#### 1. List Post's Comments

edit `app/views/posts/show.html.erb` and add the following:

```
 <h2>Comments</h2>
   <% @post.comments.each do |comment| %>
     <%= div_for comment do %>
     <p>
       <strong>Posted <%= time_ago_in_words(comment.created_at) %></strong><br />
       <%= h(comment.body) %>
     </p>
   <% end %>
 <% end %>
```

#### 2. Create new comment form on Post's page

edit `app/views/posts/show.html.erb` and add the following:

```
<%= form_for([@post, Comment.new]) do |f| %>
  <p><%= f.label :body, "New Comment" %><br />
    <%= f.text_area :body %>
  </p>
  <p><%= f.submit "Add Comment" %></p>
<% end %>
```
We need to pass in the post's id so that the user isn't prompted to fill it out: edit `app/controllers/comments_controller.rb` and modify as follows:

```
def create
  @post = Post.find(params[:post_id])
  @comment = @post.comments.create(comment_params)
  
  respond_to do |format|
    if @comment.save
      format.html { redirect_to @post, notice: 'Comment was successfully created.' }
```

Now let's comment-out the [now unneeded] routes for comments. Edit `config/routes.rb` and comment-out the appropriate line:

```
Rails.application.routes.draw do
  #resources :comments
```
Remember to restart your rails server (you need to restart your rails server every time you change the routes).

#### 3. Authentication
We need to add a *before filter* to our post's controller. Let's modify `app/controllers/posts_controller.rb`:

```
class PostsController < ApplicationController
  before_action :set_post, only: [:show, :edit, :update, :destroy]
  before_action :authenticate, except: [:index, :show]
```
Let's add an authenticate method in the `private` methods section of the controller:

```
def authenticate
  authenticate_or_request_with_http_basic do |name, password|
    name == "admin" && password == "secret"
  end
end
```

#### 4. Test and Push
Test to make sure it works. Then push to your bitbucket repo:

```
git push origin head
```

#### 5. Submit Assignment
Submit assignments here: [http://fdisk.co/mooc](http://fdisk.co/mooc).
