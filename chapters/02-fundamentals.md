# Fundamentals

Design patterns are proven solutions to common development problems that can help us improve the organization and structure of our applications. By using patterns, we benefit from the collective experience of skilled developers who have repeatedly solved similar problems. 

Historically, developers creating desktop and server-class applications have had a wealth of design patterns available for them to lean on, but it's only been in the past few years that such patterns have been applied to client-side development.

In this chapter, we're going to explore the evolution of the Model-View-Controller (MVC) design pattern and get our first look at how the Backbone.js framework enables application of this pattern to client-side development.

## MVC

MVC is an architectural design pattern that encourages improved application organization through a separation of concerns. It enforces the isolation of business data (Models) from user interfaces (Views), with a third component (Controllers) traditionally managing logic, user-input, and coordination of models and views. The pattern was originally designed by [Trygve Reenskaug](http://en.wikipedia.org/wiki/Trygve_Reenskaug) while working on Smalltalk-80 (1979), where it was initially called Model-View-Controller-Editor. MVC was described in depth in [“Design Patterns: Elements of Reusable Object-Oriented Software”](http://www.amazon.co.uk/Design-patterns-elements-reusable-object-oriented/dp/0201633612) (The "GoF" or “Gang of Four” book) in 1994, which played a role in popularizing its use.


### Smalltalk-80 MVC

It's important to understand the issues that the original MVC pattern was aiming to solve as it has changed quite heavily since the days of its origin. Back in the 70's, graphical user-interfaces were few and far between. An approach known as [Separated Presentation](http://martinfowler.com/eaaDev/uiArchs.html) began to be used as a means to make a clear division between domain objects which modeled concepts in the real world (e.g., a photo, a person) and the presentation objects which were rendered to the user's screen.

The Smalltalk-80 implementation of MVC took this concept further and had an objective of separating out the application logic from the user interface. The idea was that decoupling these parts of the application would also allow the reuse of models for other interfaces in the application. There are some interesting points worth noting about Smalltalk-80's MVC architecture:

* A Domain element was known as a Model and was ignorant of the user-interface (Views and Controllers)
* Presentation was taken care of by the View and the Controller, but there wasn't just a single view and controller. A View-Controller pair was required for each element being displayed on the screen and so there was no true separation between them
* The Controller's role in this pair was handling user input (such as key-presses and click events) and doing something sensible with them
* The Observer pattern was used to update the View whenever the Model changed

Developers are sometimes surprised when they learn that the Observer pattern (nowadays commonly implemented as a Publish/Subscribe system) was included as a part of MVC's architecture decades ago. In Smalltalk-80's MVC, the View and Controller both observe the Model: anytime the Model changes, the Views react. A simple example of this is an application backed by stock market data - for the application to show real-time information, any change to the data in its Model should result in the View being refreshed instantly.

Martin Fowler has done an excellent job of writing about the [origins](http://martinfowler.com/eaaDev/uiArchs.html) of MVC over the years and if you are interested in further historical information about Smalltalk-80's MVC, I recommend reading his work.

### MVC Applied To The Web

The web heavily relies on the HTTP protocol, which is stateless. This means that there is not a  constantly open connection between the browser and server; each request instantiates a new communication channel between the two. Once the request initiator (e.g., a browser) gets a response the connection is closed. This fact creates a completely different context when compared to the one of the operating systems on which many of the original MVC ideas were developed. The MVC implementation has to conform to the web context. 

A typical server-side MVC implementation implements the Front Controller design pattern. This pattern layers an MVC stack behind a single point of entry. This single point of entry means that all HTTP requests (e.g., `http://www.example.com`, `http://www.example.com/whichever-page/`, etc.) are routed by the server's configuration to the same handler, independent of the URI.

When the Front Controller receives an HTTP request it analyzes it and decides which class (Controller) and method (Action) to invoke.  The selected Controller Action takes over and interacts with the appropriate Model to fulfill the request. The Controller receives data back from the Model, loads an appropriate View, injects the Model data into it, and returns the response to the browser.

For example, let say we have our blog on `www.example.com` and we want to edit an article (with `id=43`) and request `http://www.example.com/article/edit/43`:

On the server side, the Front Controller would analyze the URL and invoke the Article Controller (corresponding to the `/article/` part of the URI) and its Edit Action (corresponding to the `/edit/` part of the URI). Within the Action there would be a call to, lets say, the Articles model and its `Articles::getEntry(43)` method (43 corresponding to the `/43` at the end of the URI). This would return the blog article data from the database for edit. The Article Controller would then load the (`article/edit`) View which would include logic for injecting the article's data into a form suitable for editing its content, title, and other (meta) data. Finally, the resulting HTML response would be returned to the browser.

As you can imagine, a similar flow is necessary with POST requests after we press a save button in a form. The POST action URI would look like `/article/save/43`. The request would go through the same Controller, but this time the Save Action would be invoked (due to the `/save/` URI chunk), the Articles Model would save the edited article to the database with `Articles::saveEntry(43)`, and the browser would be redirected to the `/article/edit/43` URI for further editing.

Finally, if the user requested `http://www.example.com/` the Front Controller would invoke the default Controller and Action; e.g., the Index Controller and its Index action. Within Index Action there would be a call to the Articles model and its `Articles::getLastEntries(10)` method which would return the last 10 blog posts. The Controller would load the blog/index View which would have basic logic for listing the blog posts.

The picture below shows this typical HTTP request/response lifecycle for server-side MVC:

![](img/webmvcflow_bacic.png)

The Server receives an HTTP request and routes it through a single entry point. At that entry point, the Front Controller analyzes the request and based on it invokes an Action of the appropriate Controller. This process is called routing. The Action Model is asked to return and/or save submitted data. The Model communicates with the data source (e.g., database or API). Once the Model completes its work it returns data to the Controller which then loads the appropriate View. The View executes presentation logic (loops through articles and prints titles, content, etc.) using the supplied data. In the end, an HTTP response is returned to the browser.

The need for fast, complex, and responsive Ajax-powered web applications demands replication of a lot of this logic on the client side, dramatically increasing the size and complexity of the code residing there. Eventually this has brought us to the point where we need MVC (or a similar architecture) implemented on the client side to better structure the code and make it easier to maintain and further extend during the application life-cycle.

And, of course, JavaScript and browsers constitute another context to which the traditional MVC paradigm must be adapted.

### MVC In The Browser

In complex JavaScript single-page web applications (SPA), all application responses (e.g., UI updates) to user inputs are done seamlessly on the client-side. Data fetching and persistence (e.g., saving to a database on a server) are done with Ajax in the background. For silky, slick, and smooth user experiences, the code powering these interactions needs to be well thought out.

**The problem**

A typical page in a SPA consists of small elements representing logical entities. These entities belong to specific data domains and need to be presented in particular ways on the page.

A good example is a basket in an e-commerce web application which can have items added to it. This basket might be presented to the user in a box in the top right corner of the page (see the picture below): 

![](img/wireframe_e_commerce.png)

The basket and its data are presented in HTML. The data and its associated view in HTML changes over time. There was a time when we used jQuery (or a similar DOM manipulation library) and a bunch of Ajax calls and callbacks to keep the two in sync. That often produced code that was not well-structured or easy to maintain. Bugs were frequent and perhaps even unavoidable.

Through evolution, trial and error, and a lot of spaghetti (and not so spaghetti-like) code, JavaScript developers have, in the end, harnessed the ideas of the traditional MVC paradigm. This has led to the development of a number of JavaScript MVC frameworks, including Ember.js, JavaScriptMVC, and of course Backbone.js.

### Client-Side MVC - Backbone Style

Let's take our first look at how Backbone.js brings the benefits of MVC to client-side development using a Todo application as our example. We will build on this example in the coming chapters when we explore Backbone's features but for now we will just focus on the core components' relationships to MVC.

Our example will need a div element to which it can attach a list of Todo's and an HTML template for each Todo item containing its title and a checkbox indicating if it has been completed. These are provided by the following HTML:

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title></title>
  <meta name="description" content="">
</head>
<body>
  <div id="todo">
  </div>
  <script type="text/template" id="item-template">
    <div>
      <input id="todo_complete" type="checkbox" <%= completed %>>
      <%= title %>
    </div>
  </script>
  <script src="underscore-min.js"></script>
  <script src="cranium.js"></script>
  <script src="example.js"></script>
</body>
</html>
```

In our Todo application, Backbone Model instances are used to hold the data for each Todo item:

```javascript
// Define a Todo Model
var Todo = Backbone.Model.extend({
  // Default todo attribute values
  defaults: {
    title: '',
    completed: false
  }
});

// Instantiate the Todo Model with a title, allowing completed attribute
// to default to false
var todo1 = new Todo({ 
  title: 'Check attributes property of the logged models in the console.'
});
```

Our Todo model extends Backbone.Model and simply defines default values for two data attributes. As you will discover in the upcoming chapters Backbone Models provide many more features, but this simple model illustrates that first and foremost a Model is a data container.

Each Todo instance will be rendered on the page by a TodoView:

```javascript
var TodoView = Backbone.View.extend({

  tagName:  'li',

  // Cache the template function for a single item.
  todoTpl: _.template( $('#item-template').html() ),

  events: {
    'dblclick label': 'edit',
    'keypress .edit': 'updateOnEnter',
    'blur .edit':   'close'
  },

  // Re-render the titles of the todo item.
  render: function() {
    this.$el.html( this.todoTpl( this.model.toJSON() ) );
    this.input = this.$('.edit');
    return this;
  },

  edit: function() {
    // executed when todo label is double clicked
  },

  close: function() {
    // executed when todo loses focus
  },

  updateOnEnter: function( e ) {
    // executed on each keypress when in todo edit mode, 
    // but we'll wait for enter to get in action
  }
});

// create a view for a todo
var todoView = new TodoView({model: todo});
```

TodoView is defined by extending Backbone.View and is instantiated with an associated model. In our example, the ```render()``` method uses a template to construct the HTML for the Todo item which is placed inside a li element. Each call to ```render()``` will replace the content of the li element using the current model data. Later we will see how a View can bind its ```render()``` method to model change events, causing the View to re-render whenever the model changes.

So far, we have seen that Backbone.Model implements the Model aspect of MVC and Backbone.View implements the View. However, as we noted earlier, Backbone departs from traditional MVC when it comes to Controllers - there is no Backbone.Controller!

Instead, the Controller responsibility is addressed within the View. Recall that Controllers respond to requests and perform appropriate actions which may result in changes to the Model and updates to the View. In a single-page application, rather than having requests in the traditional sense, we have events. Events can be traditional browser DOM events (e.g., clicks) or internal application events such as Model changes.

In our TodoView, the ```events``` attribute fulfills the role of the controller configuration, defining how events occurring within the view's DOM element are to be routed to event-handling methods defined in the view.

While in this instance events help us relate Backbone to the MVC pattern, we will see them playing a much larger role in our SPA applications. Backbone.Event is a fundamental Backbone component which is mixed into both Backbone.Model and Backbone.View, providing them with rich event management capabilities.

This completes our first encounter with Backbone.js. The remainder of this book will explore the many features of the framework which build on these simple constructs. Before moving on, let's take a look at common features of JavaScript MV* frameworks.

### Implementation Specifics

#### Models

* The built-in capabilities of models vary across frameworks; however, it's common for them to support validation of attributes, where attributes represent the properties of the model, such as a model identifier.

* When using models in real-world applications we generally also need a way of persisting models. Persistence allows us to edit and update models with the knowledge that their most recent states will be saved somewhere, for example in a web browser's localStorage data-store or synchronized with a database.

* A model may also have single or multiple views observing it. Depending on the requirements, a developer might create a single view displaying all Model attributes, or they might create separate views displaying different attributes. The important point is that the model doesn't care how these views are organized, it simply announces updates to its data as necessary through the framework's event system.

* It is not uncommon for modern MVC/MV* frameworks to provide a means of grouping models together. In Backbone, these groups are called "Collections." Managing models in groups allows us to write application logic based on notifications from the group when a model within the group changes. This avoids the need to manually observe individual model instances. We'll see this in action later in the book.

* If you read older texts on MVC, you may come across a description of models as also managing application "state." In JavaScript applications state has a specific meaning, typically referring to the current state of a view or sub-view on a user's screen at a fixed time. State is a topic which is regularly discussed when looking at Single-page applications, where the concept of state needs to be simulated.

#### Views

* Users interact with views, which usually means reading and editing model data. For example, in our todo application, todo model viewing happens in the user interface in the list of all todo items. Within it, each todo is rendered with its title and completed checkbox. Model editing is done through an "edit" view where a user who has selected a specific todo edits its title in a form.

* We define a ```render()``` utility within our view which is responsible for rendering the contents of the ```Model``` using a JavaScript templating engine (provided by Underscore.js) and updating the contents of our view, referenced by ```el```.

* We then add our ```render()``` callback as a Model subscriber, so the view can be triggered to update when the model changes.

* You may wonder where user interaction comes into play here. When users click on a todo element within the view, it's not the view's responsibility to know what to do next. A Controller makes this decision. In Backbone, this is achieved by adding an event listener to the todo element which delegates handling of the click to an event handler.

**Templating**

In the context of JavaScript frameworks that support MVC/MV*, it is worth looking more closely at JavaScript templating and its relationship to Views.

It has long been considered bad practice (and computationally expensive) to manually create large blocks of HTML markup in-memory through string concatenation. Developers using this technique often find themselves iterating through their data, wrapping it in nested divs and using outdated techniques such as ```document.write``` to inject the 'template' into the DOM. This approach often means keeping scripted markup inline with standard markup, which can quickly become difficult to read and maintain, especially when building large applications.

JavaScript templating libraries (such as Handlebars.js or Mustache) are often used to define templates for views as HTML markup containing template variables. These template blocks can be either stored externally or within script tags with a custom type (e.g 'text/template'). Variables are delimited using a variable syntax (e.g <%= title %>). Javascript template libraries typically accept data in JSON, and the grunt work of populating templates with data is taken care of by the framework itself. This has a several benefits, particularly when opting to store templates externally which enables applications to load templates dynamically on an as-needed basis.

Let's compare two examples of HTML templates. One is implemented using the popular Handlebars.js library, and the other uses Underscore's 'microtemplates'.

**Handlebars.js:**

```html
<div class="view">
  <input class="toggle" type="checkbox" {{#if completed}} "checked" {{/if}}>
  <label>{{title}}</label>
  <button class="destroy"></button>
</div>
<input class="edit" value="{{title}}">
```

**Underscore.js Microtemplates:**

```html
<div class="view">
  <input class="toggle" type="checkbox" <%= completed ? 'checked' : '' %>>
  <label><%= title %></label>
  <button class="destroy"></button>
</div>
<input class="edit" value="<%= title %>">
```

You may also use double curly brackets (i.e ```{{}}```) (or any other tag you feel comfortable with) in Microtemplates. In the case of curly brackets, this can be done by setting the Underscore ```templateSettings``` attribute as follows:

```javascript
_.templateSettings = { interpolate : /\{\{(.+?)\}\}/g };
```

**A note on navigation and state**

It is also worth noting that in classical web development, navigating between independent views required the use of a page refresh. In single-page JavaScript applications, however, once data is fetched from a server via Ajax, it can be dynamically rendered in a new view within the same page. Since this doesn't automatically update the URL, the role of navigation thus falls to a "router", which assists in managing application state (e.g., allowing users to bookmark a particular view they have navigated to). As routers are neither a part of MVC nor present in every MVC-like framework, I will not be going into them in greater detail in this section.

#### Controllers


In our Todo application, a controller would be responsible for handling changes the user made in the edit view for a particular todo, updating a specific todo model when a user has finished editing.

It's with controllers that most JavaScript MVC frameworks depart from the traditional interpretation of the MVC pattern. The reasons for this vary, but in my opinion, Javascript framework authors likely initially looked at server-side interpretations of MVC (such as Ruby on Rails), realized that that approach didn't translate 1:1 on the client-side, and so re-interpreted the C in MVC to solve their state management problem. This was a clever approach, but it can make it hard for developers coming to MVC for the first time to understand both the classical MVC pattern and the "proper" role of controllers in other JavaScript frameworks.

So does Backbone.js have Controllers? Not really. Backbone's Views typically contain "controller" logic, and Routers are used to help manage application state, but neither are true Controllers according to classical MVC.

In this respect, contrary to what might be mentioned in the official documentation or in blog posts, Backbone isn't truly an MVC framework. It's in fact better to see it a member of the MV* family which approaches architecture in its own way. There is of course nothing wrong with this, but it is important to distinguish between classical MVC and MV* should you be relying on discussions of MVC to help with your Backbone projects.

## What does MVC give us?

To summarize, the separation of concerns in MVC facilitates modularization of an application's functionality and enables:

* Easier overall maintenance. When updates need to be made to the application it is clear whether the changes are data-centric, meaning changes to models and possibly controllers, or merely visual, meaning changes to views.
* Decoupling models and views means that it's straight-forward to write unit tests for business logic
* Duplication of low-level model and controller code is eliminated across the application
* Depending on the size of the application and separation of roles, this modularity allows developers responsible for core logic and developers working on the user-interfaces to work simultaneously


### Delving Deeper into MVC

Right now, you likely have a basic understanding of what the MVC pattern provides, but for the curious, we'll explore it a little further.

The GoF (Gang of Four) do not refer to MVC as a design pattern, but rather consider it a "set of classes to build a user interface." In their view, it's actually a variation of three other classical design patterns: the Observer (Pub/Sub), Strategy, and Composite patterns. Depending on how MVC has been implemented in a framework, it may also use the Factory and Decorator patterns. I've covered some of these patterns in my other free book, "JavaScript Design Patterns For Beginners" if you would like to read about them further.

As we've discussed, models represent application data, while views handle what the user is presented on screen. As such, MVC relies on Publish/Subscribe for some of its core communication (something that surprisingly isn't covered in many articles about the MVC pattern). When a model is changed it "publishes" to the rest of the application that it has been updated. The "subscriber," generally a Controller, then updates the view accordingly. The observer-viewer nature of this relationship is what facilitates multiple views being attached to the same model.

For developers interested in knowing more about the decoupled nature of MVC (once again, depending on the implementation), one of the goals of the pattern is to help define one-to-many relationships between a topic and its observers. When a topic changes, its observers are updated. Views and controllers have a slightly different relationship. Controllers facilitate views' responses to different user input and are an example of the Strategy pattern.

### Summary

Having reviewed the classical MVC pattern, you should now understand how it allows developers to cleanly separate concerns in an application. You should also now appreciate how JavaScript MVC frameworks may differ in their interpretation of MVC, and how they share some of the fundamental concepts of the original pattern.

When reviewing a new JavaScript MVC/MV* framework, remember - it can be useful to step back and consider how it's opted to approach Models, Views, Controllers or other alternatives, as this can better help you understand how the framework is intended to be used.

### Further reading

If you are interested in learning more about the variation of MVC which Backbone.js is better categorized under, please see the MVP (Model-View-Presenter) section in the appendix.


## Fast facts

### Backbone.js

* Core components: Model, View, Collection, Router. Enforces its own flavor of MV*
* Used by large companies such as SoundCloud and Foursquare to build non-trivial applications
* Event-driven communication between views and models. As we'll see, it's relatively straight-forward to add event listeners to any attribute in a model, giving developers fine-grained control over what changes in the view
* Supports data bindings through manual events or a separate Key-value observing (KVO) library
* Support for RESTful interfaces out of the box, so models can be easily tied to a backend
* Extensive eventing system. It's [trivial](http://lostechies.com/derickbailey/2011/07/19/references-routing-and-the-event-aggregator-coordinating-views-in-backbone-js/) to add support for pub/sub in Backbone
* Prototypes are instantiated with the ```new``` keyword, which some developers prefer
* Agnostic about templating frameworks, however Underscore's micro-templating is available by default. Backbone works well with libraries like Handlebars
* Doesn't support deeply nested models, though there are Backbone plugins such as [Backbone-relational](https://github.com/PaulUithol/Backbone-relational) which can help
* Clear and flexible conventions for structuring applications. Backbone doesn't force usage of all of its components and can work with only those needed.