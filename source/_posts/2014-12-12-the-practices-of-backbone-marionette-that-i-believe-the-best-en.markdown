---
layout: post
title: "The Practices of Backbone.Marionette That I believe The Best"
date: 2014-12-12 14:08
comments: true
categories:
- en
- javascript
---

# Part1: Design

## Use sub-module
You can create sub-modules as many as you want inside one Marionette app. I higly enoucurage to take the advantage of sub-modules.

This is the how to define your sub-modules.

```js
var MyApp = new Backbone.Marionette.Application();
MyApp.module("ModuleA", function(ModuleA, MyApp, Backbone, Marionette, $, _){
  // your module code goes here //
})
```

You may wonder about parameters in callback function. This is automatically passed in this order:

- The module itself
- The Application object
- Backbone
- Backbone.Marionette
- jQuery
- Underscore

They are defined in local scope and only accessbile from the callback function.

Now, let's see the benifits of using sub-modules.

### Encapsulation

You can define public and private methods easily inside a module.

```js
var MyApp = new Backbone.Marionette.Application();

MyApp.module("ModuleA", function(ModuleA, Demotape, Backbone, Marionette, $, _){
  // Defining public method 
  ModuleA.publicMethod = function() {
    return privateMethod();
  }
  
  // Defining private method 
  var privateMethod = function() {
    return "some value"
  }
  
})
```

Now other sub-modules can access public method but not private one.

```js
MyApp.ModuleA.publicMethod() // => "some value"
MyApp.ModuleA.privateMethod() // => "Uncaught TypeError: undefined is not a function"
```

### Starting and Stopping Modules

You can start and stop your sub-modules individually. By default, sub-modules is automatically started when Marionette parent app is started. You can change this behavior by `startWithParent` option.

```js
MyApp.module("ModuleA", function(ModuleA, Demotape, Backbone, Marionette, $, _){
    this.startWithParent = false;
})

MyApp.start() // this doesn't start ModuleA
MyApp.ModuleA.start() // you have to start by yourself

```

Why is this important? Because sometimes you want to execute some codes before starting other modules. Let's say you need to load some resources from a remote server when user enters your app and other modules are dependent on that resources.

```js
MyApp.on("start", function() {
    // Let's say deferredFetchihg grubs some data from remote server
    $.when(deferredFetching()).done(function(data) {
        // Set data to Module1
        MyApp.Module1.setData(data);
        
        // Now Module1 is ready to start
        MyApp.Module1.start()
    });
})
```

How about `stop()`? I don't think you need to use `stop()` unless your app is huge so that you need to pay careful attention to memory usage.

However, `stop()` is very useful when it comes to testing. I will cover this in the following post.

## File Hierarchy and Naming Convention

There is no canonical way to organize your files in Backbone.Marionette. However, it is good to agree on a convetion for how to name and organize files in project if you are working in a team. This is the pattern that works for me.

```
project_root/
    |
    |---application.js
    |
    |---small_module/
    |      |
    |      |---small_module.js
    |      |
    |      |---small_module_controller.js
    |      |
    |      |---small_module_model.js
    |      |
    |      |---small_module_view.js
    |      |
    |      |---templates/
    |              |
    |              |---template1.hbs
    |      
    |---big_module/
           |
           |---big_module.js
           |
           |---big_module_controller.js
           |
           |---big_module_model.js
           |
           |---new
           |    |
           |    |---big_module_new_view.js
           |    |
           |    |---templates/
           |            |
           |            |---template1.hbs
           |            |---template2.hbs
           |---edit
           |    |
           |    |---big_module_edit_view.js
           |    |
           |    |---templates/
           |            |
           |            |---template1.hbs
           |            |---template2.hbs
```

`application.js` is where you want to decalre global things of your app. This includes 

- App initialization code
- App starting code (Ex: `App.start()`)
- Route definition and initilalization
- Useful global helper (Ex: `App.currentUser()`, `App.showSuccessNotification()`) 


I like module-based file hierarchy.

In module based organization, files other than application.js goes under each module directory. If the module is big one, they are further broken down into seperate directories by action.

The role of each file under module direclty is self-explanatory except `<module_name>.js`.

`<module_name>.js` is used to define basic things for the entier module. These includes:

- module option (Ex. `this.startWithParent = false`)
- module initilaization (Ex. `addInitializer({})` )
- event listener (Ex. `Module1.on("event", function({}))`)
    
Maybe it is fair to explain `<module_name>_controller.js` since it does not exist in Backbone as well.

`<module_name>_controller.js` holds methods that redpond to user entry to your app. One example of such method is something like `showItem()`. What `showItem()` does is that instantiate a view, fetch model, pass it to view, and call `render()` of view to display html. It is similar to what Rails ActionController does.

At last, you may think it is redudant to prefix every files with module name, but this makes it easy to search files by module name. Not must to have, but it is useful once your project becomes bigger where you have many `controller.js` or `view.js`.

## Responsibility of MVC Components

Marionette brings consistency to your Backbone app. However, this is not enough if you are working in a team. Marionette still allows developrs to write codes whatever they want which rapidly makes your project spaghetti. Let's make it clear what MVC components is responsbile for what. 

### M
The responsibility of model is clearer than other components, so I will go quickly.

#### Business logic 
Just like model of server side, this is the place where you write code for business logic.

**Ex.** Calculate and return total price by adding tax to sub-total.

#### Ajax ####
When modle is mapped to external resource, it fetches resource from external servers.

#### Validation ####
Before modle is saved into exteranl database, it needs to validate data. When validation fails, it must notifies by using event. View must respond to validation event and notifies user.

BTW, you should use [Backbone.Validation](https://github.com/thedersen/backbone.validation) plugin if your models needs validation. Never try to write validation code by youself unless you are super confident. It is not easy to write validation in a declarative way.

### V
This is the place which easily becomes chaotic because vanilla Backbone puts much burden to View. In Marionette, it is less loaded thanks to Controller, but still you want to be careful.

#### Render html
When you define View, you will speficy which template to use. Template value is fetched by a model that View holds and then rendered a html.

#### Define mapping of DOM and event
You can define what action of user for DOM elements do what in View. This is done by `events` propety. I don't want to go detail about this since implementation is not the the subject of post, but just briefly showing an example.

```js
var MyView = Marionette.ItemView.extend({
    ui: {
        paragraph: 'p',
        button: '.my-button'
    },
    
    events: {
        'click ui.button': 'clickedButton'
    },
    
    clickedButton: function() {
        console.log('I clicked the button!');
    }
});
```

View listens on click event of `ui.button` and trigger clickedButton() event. Notice `ui` propety. This does not exist in Backbone. You can pass a object that mappes properties and jQuery DOM. You can access the DOM like this.

```js
 MyView.ui.paragraph
```

This is simple yet very powerful feature. I will write more about this in subsequent post.

#### Listens on Model/Collection change event
Most of Views are passed model or collection. Whenever there are changes to model/collection (Ex. setting new value to the field of model / model is removed from collection ), View responds to the event, changes its $el, and modifies its own html.

### C
Controller is loosely defined even in official doc.

Quoted from [here](https://github.com/marionettejs/backbone.marionette/blob/master/docs/marionette.controller.md)
> Its name can be a cause for confusion, as it actually has nothing to do with the popular MVC architectural pattern. Instead, it's better to think of the Controller as a base object from which you can build.

So, as the doc says, Controller is the place where you can put things that fit nowhere else.

But, what exactly are they?

#### Providing entry point for user
When user hits url bar, the user now enters your app. First, router responds the user entry and decides which route to use. Then, router invokes Controller API. One example of Controller API looks something like this:

TODO: understand how routing automatically mappes controller method.
TODO: should controller is respnsible for #change event?
```js
MyApp.module("ModuleA", function(ModuleA, Demotape, Backbone, Marionette, $, _){
    ModuleA.Controller = Marionette.Controller.extend({
        showItem: function() {
            
            // Here you write code that "assembles" things to display an item
            var item = new Item()
            var itemView = new ItemView(model: item)
            
            itemView.render();
        }
    })
})
```

So, as you can see, I am using Controller pretty much like the way I use Rails ActionController. I don't see no reason why this is wrong even if doc says it is nothing to do with server side MVC.

#### Handling Ajax call for View
I often ask this quetion myself: `Is it ok that my View does ajax call?` Can you imagine what I am talking about? The View that makes ajax call looks like this.

```js
FormView = Marionette.ItemView.extend({
    events: {
        'click button.submit': 'submitForm'
    },
    
    submitForm: function() {
        // Get form data somehow//
        var data = getFormData();
        
        // Model.save fires ajax call to remote server
        this.model.save(data, {
            success: function(){
                console.log("form is submitted");
            },
            error: function(){
                console.log("error");
            }
        });  
    }
})
```

Or should I put this task to Marionette.Controller, instead?

```js
FormController = Marionette.Controller.extend({
    
    initialize: function(options){
        var model = new FormData()
        var formView = FormView.new(model: model)
        
        // Suppose formView triggers "form:submit"
        formView.on("form:submit", function(data) {
            model.save(data, {
                success: function() {
                    console.log("form is submitted");
                },
                
                error: function(){
                    console.log("error");
                }
            })
        })
    },
});
```

Both works. Probably you have no choise in vanilla Backbone but doing the first one. However, Marionette comes with Controller, so I want Controller to do the job. In this way, View can focus on the mapping of DOM and event.
