[Netzke](http://netzke.org) is a set of Ruby gems that helps you build client-server components, represented in the browser with the help of [Sencha Ext JS](http://www.sencha.com/products/extjs/), and on the server are powered by Ruby on Rails. It's most useful for creating complex data-rich applications that have to provide the UI to a lot of different data - think about banking, logistics, call centers, and other ERP systems.

In this walk-through I'll show you how little time it takes to set up a simple application with 2 interacting grids that provide the UI to 2 associated (one-to-many) Rails models. As both grids will be powered by the [Netzke::Basepack::Grid](http://www.rubydoc.info/github/netzke/netzke-basepack/master/Netzke/Basepack/Grid) component, you may be surprised with all the features like sorting, pagination, search, multi-line editing, etc that this app will provide out of the box.

So, let's get started.

## Setting things up

Create a new Rails application:

    $ rails new netzke_bookshelf && cd netzke_bookshelf

Add Netzke gem to your Gemfile:

```ruby
gem 'netzke'
```

Install the gems:

    $ bundle

Symlink or copy the Ext JS files ([direct download link](http://cdn.sencha.com/ext/gpl/ext-5.1.0-gpl.zip)) and, optionally, the [FamFamFam silk icons](http://www.famfamfam.com/lab/icons/silk/), into `public/extjs` and `public/images/icons` respectively. For example:

    $ echo MOST PROBABLY NO COPY-PASTING HERE
    $ ln -s ~/code/extjs/ext-5.1.0 public/extjs
    $ mkdir public/images
    $ ln -s ~/assets/famfamfam-silk public/images/icons

Declare Netzke routes:

```ruby
# config/routes.rb
NetzkeTaskManager::Application.routes.draw do
  netzke
end
```

The `netzke` method sets up routes needed for communication between client and server side of every Netzke component.

## Creating Rails models

We'll build a little personal bookshelf app, let's start with generating the 2 models for it:

    $ rails g model author name
    $ rails g model book author_id:integer title exemplars:integer completed:boolean
    
Run the migrations:

    $ rake db:migrate

Let's set the 1-to-many relations between the models, and add basic validations:

```ruby
# app/models/author.rb
class Author < ActiveRecord::Base
  validates :name, presence: true
  has_many :books, dependent: :delete_all
end

# app/models/book.rb
class Book < ActiveRecord::Base
  validates :title, presence: true
  belongs_to :author
end
```

Now let's get to the fun part.

## Creating the grids

Netzke components are Ruby classes, so, let's create one in the `app/components/authors.rb` (you'll need to add the `app/components` directory first):

```ruby
# app/components/authors.rb
class Authors < Netzke::Basepack::Grid
  def configure(c)
    super
    c.model = "Author"
  end
end
```

The minimum configuration that `Netzke::Basepack::Grid` requires is the name of the model, and that's what we provided. Now let's see what this code is capable of.

The netzke-testing gem, which is a part of the Netzke bundle, provides convenient routes for testing components in isolation, so, we'll make use of it. Fire up your Rails app:

    $ rails s
    
... and point your browser to http://localhost:3000/netzke/components/Authors

Here's what you should see:

![Authors grid](//imgur.com/zTutMNX.png)

It's a full-featured grid view: try adding, editing, deleting, sorting and searching records in it, or rearranging the columns:

However, a more interesting example to play with would be our books grid, as it will have more columns of different types:

```ruby
# app/components/books.rb
class Books < Netzke::Basepack::Grid
  def configure(c)
    super
    c.model = "Book"
  end
end
```

Navigate to http://localhost:3000/netzke/components/Books in order to see it. I've added a couple of books before sharing the screenshot with you:

![Books grid](//imgur.com/giyMAC1.png)

Note that the grid has picked up the Author association without any configuration (yes, it honors some conventions).

As you may expect, our grids are highly configurable, and many configuration options can be found in the [documentation](http://www.rubydoc.info/github/netzke/netzke-basepack/master/Netzke/Basepack/Grid). Let us, for instance, make a few changes into the Books grid, customizing the columns and the button bar:

```ruby
class Books < Netzke::Basepack::Grid
  def configure(c)
    c.columns = [
      # you may configure columns inline like this:
      { name: :author__name, text: "Author" },

      :title,
      :exemplars,
      :completed
    ]

    c.model = "Book"

    # which buttons to show in the bottom toolbar
    c.bbar = [:add, :edit, :del, '->', :apply]

    super
  end

  # you may also use DSL to configure columns individually
  column :completed do |c|
    c.width = 100
  end
end
```

Here's how it'll look like:

![Books grid customized](//imgur.com/4gFOrVe.png)

## Making the composite component

Because our Netzke components are Ruby classes, inheritance and mixins will work just fine, and Netzke will even take care of the client-side inheritance, too. Yes, the client side of a Netzke components is, in its turn, an Ext JS component. This architecture makes it possible to build arbitrary component hierarchies, which is indespensible when building complex applications.

Further in the text I'll be using the terms "server side" and "client side" (of a component) a lot, meaning that "server side" is the Ruby code (powered by Rails) that executes on the server, and "client side" is that Javascript code (powered by Ext JS) in the browser.

> Have you asked yourself where that `bbar` property in the previous example comes from? That's a property known to any [Ext.panel.Panel](http://docs.sencha.com/extjs/5.1/5.1.0-apidocs/#!/api/Ext.panel.Panel-cfg-bbar), which the client-side part of our grid, in fact, inherits from.

However, inheritance is slightly out of the scope of this introduction. Instead, I'll show you another important trait of Netzke: composability (aren't those things called "components" after all?) Ext JS is of great help here, because it was designed with composability in mind from the day 1.

Let's start with creating the container component (we'll call it AuthorsAndBooks):

```ruby
class AuthorsAndBooks < Netzke::Basepack::Viewport
  def configure(c)
    super
    c.items = [:authors, :books]
    c.title = "Book shelf"
  end

  component :authors
  component :books
end
```

> We're inheriting from `Netzke::Basepack::Viewport` in order for the new component to automatically occupy the complete window space of the browser; this is not important for this tutorial, and subclassing directly from `Netzke::Base`, which is the top-component in Netzke class hierarchy, would do just as fine.

This basic configuration is enough for the new component to nest 2 grids together:

![AuthorsAndBooks](//imgur.com/9PApyeh.png)

What we need to do though is to fix the layout a little bit, using some [help from Ext JS](http://docs.sencha.com/extjs/5.1/5.1.0-apidocs/#!/api/Ext.layout.container.VBox).

```ruby
class AuthorsAndBooks < Netzke::Basepack::Viewport
  def configure(c)
    super
    c.items = [:authors, :books]
    c.layout = {type: :hbox, align: :stretch}
  end

  component :authors do |c|
    c.flex = 1
  end

  component :books do |c|
    c.columns = [:title, :exemplars, :completed]
    c.flex = 1
  end
end
```

![AuthorsAndBooks layed out](//imgur.com/fw8lD4B.png)

This piece also demonstrates how we can override the configuration of our child components: we're getting rid of the "Author" column in the Books grid, because on the next step we'll glue our components together in such a way, that clicking on an author in the Authors component will update the Books grid with books written by that author. So, the Author column in the Books grid becomes redundant.

Let me first describe with words what we want to achieve. Say, the user clicks on an author. It's responsibility of the AuthorsAndBooks component to process that click event, and then update the Books grid with books filtered by that author. In order to achieve this, we'll do the following:

* Set the row click event in the client (Javascript) code of the AuthorsAndBooks component.
* The event handler will inform the server (Ruby) side of AuthorsAndBooks that the currently selected author has been changed
* After, the event handler commands the books grid to reload the data.

> Due to multiplexing of server requests 2 last things will happen in one request

How will the Books grid know it has to load the data scoped out to a specific author? Well, its scope will be set by our AuthorsAndBooks Ruby class, similarly to how we configure the columns, and that scope we will be set to what comes from that 1st call from the row click handler.

Well, sometimes code is worth a thousand words (especially when it's abundantly commented):

```ruby
# app/components/authors_and_books.rb
class AuthorsAndBooks < Netzke::Basepack::Viewport
  js_configure do |c|
    # For the client-side of our component to include the Javascript code (see below)
    c.mixin
  end

  def configure(c)
    super
    c.items = [:authors, :books]
    c.layout = {type: :hbox, align: :stretch}
  end

  component :authors do |c|
    c.flex = 1
  end

  component :books do |c|
    c.columns = [:title, :exemplars, :completed]
    c.flex = 1

    # Filter books data by author (component_session is simply the session store scope out to the AuthorsAndBooks component, and is set in the endpoint call below)
    c.scope = {author_id: component_session[:current_author_id]}
  end

  # This "gets called" by the client side
  endpoint :server_update_author do |params, this|
    # params[:author_id] comes from the client side of the component (see the Javascript code below)
    component_session[:current_author_id] = params[:author_id]
  end
end
```

```javascript
// app/components/authors_and_books/javascripts/authors_and_books.js
{
  initComponent: function () {
    this.callParent(); // Ext JS requires this

    this.netzkeGetComponent('authors').on('rowclick', function(grid, record) {
      // call the "server_update_author" endpoint of the server side
      this.serverSetAuthor({author_id: record.getId()});

      // inform the client-side of the Books grid to reload itself
      this.netzkeGetComponent('books').getStore().load();
    }, this);
  }
}
```

What's going on here? I suggest to start with the Javascript piece. First, we extend our client-side code of the component by mixing in an override for the Ext JS `initComponent` function, which gets called during component initialization. We access our nested authors grid with the help of the `netzkeGetComponent` method, in order to set the row click event there. Where does the method `serverSetAuthor` come from? That's what Netzke generates for us when we declare an equally named _endpoint_ in the Ruby class. And that's how we pass the selected author id to the server side. After this the handler informs the client side of the Books grid to reload.

That `endpoint` block in our Ruby class will get executed with the params passed from the client-side call. In it we store the current author's id in the component's session, in order to pass it as a configuration option to the Books grid. This way, when the request for the books records comes from the client-side, that grid will be instantiated with the updated configuration.

![Authors and books final](//imgur.com/srX9kd8.png)

## Wrapping it up

I won't go here into the somewhat boring details about how to embed the components that we created into Rails views, for this you may turn to the last part of the [Hello World](https://github.com/netzke/netzke-core#helloworld-component) section in the Netzke Core README. It's dead simple anyway. Instead I'll point you to the [final source code]() for this app, which will embed all 3 components in the root page of the Rails app.

And if you feel hooked, check out these 2 demo apps that will be insightful for you should you decide to start using Netzke in your projects:

* [Yanit](http://yanit.netzke.org) is an issue tracking app that I initially created in just a couple of hours for a conference talk. It's not usable in real life, I guess, but it's impressive anyway, and is good source of example code.

* [Official demo](http://demo.netzke.org) will show off a few individual components.

Both apps have links to their respective source code on GitHub.

Thanks, and give me some feedback on [Twitter](http://twitter.com/mxgrn)!
