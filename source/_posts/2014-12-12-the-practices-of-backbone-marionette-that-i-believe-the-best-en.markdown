---
layout: post
title: "The Practices of Backbone.Marionette That I believe The Best"
date: 2014-12-12 14:08
comments: true
categories:
- en
- javascript
---

# About Modularity

## Modularize App By Using Sub-Module
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

MyApp.module("ModuleA", function(ModuleA, MyApp, Backbone, Marionette, $, _){
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
MyApp.module("ModuleA", function(ModuleA, MyApp, Backbone, Marionette, $, _){
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

# About View

## Organizing DOM with ui

This pattern is inspired by [the blog post](http://lxyuma.hatenablog.com/entry/2014/01/23/002644).

[ui](https://github.com/marionettejs/backbone.marionette/blob/master/docs/marionette.itemview.md#organizing-ui-elements) is simple yet very powerful feature of Marionette. You can organize view's DOM by using `ui`. Here is simple example. Say your have a template.

```html
<div id="edit_form">
  <input class="name_input">Name
  <input class="email_input">Name
  <button type="submit" class="js-submit">Submit</button>
</div>
```

And your view code looks like this.

```js
MyView = Marionette.ItemView.extend({
    template: "#edit_form",

    ui: {
        nameInput: "input.name_input",
        emailInput: "input.email_input",
        submitButton: "button.js-submit"
    }
});
```

If you don't have ui, you can access name input by using jQuery like this.

```js
var myView = new MyView()
var input = myView.$el.find("input.name_input");
```

By using `ui`, you can access this way.

```js
var myView = new MyView()
var input = myView.ui.nameInput;
```

The difference is subtle, but later one is more maintainable. Here is why.

Html markups are the subject of frequent changes. This is not an issue if the DOM is referenced from a single place. However, when multiple places look at a single DOM, it becomes difficult to maintain.

Suppose one person changes the class name of a input field from `name_input` to `name_field`. Also suppose that many codes refer the DOM by using jQuery: `myView.$el.find("input.name_input")`. Now, you have to change every codes that uses jQuery to  access the DOM. If the DOM is referenced from tests codes, these tests all suddenly break  and updating every places is the nightmare.

`ui` solves the issue. If your codes refer the DOM by using `myView.ui.nameInput`, then the person who changes the html only needs to change `ui` object.

```js
ui: {
    nameInput: "input.name_field",
    // edited for brevity //
}
```

And the rest of codes stay the same. Huge improvement.

So here is the pattern. **Avoid jQuery and always use ui to access view's DOM**

## Using LayoutView to create nested sub-views

This pattern is inspired by [the blog post](http://lostechies.com/derickbailey/2012/03/22/managing-layouts-and-nested-views-with-backbone-marionette/).

I saw different people use different ways to create nested views in vanilla Backbone app. Marionette however provides us a nice pattern to archive this.

We will use Marionette [LayoutView](https://github.com/marionettejs/backbone.marionette/blob/master/docs/marionette.layoutview.md#marionettelayoutview) and [Region](https://github.com/marionettejs/backbone.marionette/blob/master/docs/marionette.region.md#marionetteregion).

Let's take a example of creating shopping car page. The page contains two sub-views: item list view and pricing view.

![](/images/shopping_cart1.png)

Here is our code that creates this shopping cart page. First we create a layout html and layout view.

**shopping_cart_layout.html**
```html
<div id="shopping_cart_layout">
  <div id="itemlist_region"></div>
  <div id="price_region"></div>
</div>
```

**shopping_cart_layout_view.js**
```js
ShoppingCartLayoutView = Marionette.LayoutView.extend({
    template: "#shopping_cart_layout",

    regions: {
        itemListRegion: "#itemlist_region",
        priceListRegion: "#price_region",
    }
});
```

Let's talk briefly about what `LayoutView` is. According to [official doc](https://github.com/marionettejs/backbone.marionette/blob/master/docs/marionette.layoutview.md#marionettelayoutview),

> A LayoutView is a hybrid of an ItemView and a collection of Region objects.

So, it is simply extended from `ItemView` and add `Region` objects. Since it is extended from `ItemView`, you can attach template where you can speficy region container DOM. Layout can hold as many region objects as you want so they are suitable to create a parent view.

Now, let's quickly create our sub-views. The template for these views are not important at this subject so let's imagine we have templates for them.

**item_list_view.js**
```js
ItemView = Marionette.ItemView.extend({
    template: ItemTpl,
});

ItemListView = Marionette.CollectionView.extend({
    childView: ItemView
});
```

**price_view.js**
```js
PriceListView = Marionette.ItemView.extend({
    template: PriceTpl,
});
```

Now we are ready to put all things together. Here is the code that renders the shopping cart page. Again, model and collection are not important at this subject, so let's assume we have `items` collections already.

```js
// Instantiate views
layoutView = new ShoppingCartLayoutView()
itemListView = new ItemListView({collections: items}) // We assume that items is the collection
priceListView = new PriceListView()

// Put them into layout regions
layoutView.itemListRegion.show(itemListView);
layoutView.priceListRegion.show(priceview);
```

That's it. Marionette region provides a convenient method called `show()` where you can pass any views to be rendered. `el` property of passed views are automatically provided by region. This is the reason why you don't see `el` object often when using Marionette.

Creating further nested sub-view is easy. Let's say you want to divide `PriceListView` into `TotalPriceListView` and `SubTotalPriceListView`.

![](/images/shopping_cart2.png)

What you have to do is extending `PriceListView` from `LayoutView` instead of `ItemView` and add region objects.

**price_view.js**
```js
PriceListView = Marionette.Layout.extend({
    // Let's assume the template has #total_price_region and #sub_total_price_region
    template: PriceTpl,

    regions: {
        totalRegion: "#total_price_region",
        subTotalRegion: "#sub_total_price_region",
    }
});

totalPriceListView = Marionette.ItemView.extend({
    template: totalPriceTpl,
});

subTotalPriceListView = Marionette.ItemView.extend({
    template: subTotalPriceTpl,
});
```

And put them together.

```js
priceListView = new PriceListView()
totalPriceListView = new TotalPriceListView()
subTotalPriceListView = new SubTotalPriceListView()

priceListView.totalRegion.show(totalPriceListView)
priceListView.subTotalRegion.show(subTotalPriceListView)
```

It's a piece of cake.

Before closing this section, let me introduce how to access views rendered inside region because you will often use it.

Let's say you want to access `<div id="total_price">$10000</div>` DOM to get the total price of the shopping cart from your code. In this case, you will use `currentView()` API. The code looks like this:

```js
layoutView.priceListRegion.currentView.totalRegion.currentView.ui.totalPrice
```

As you can see, it's easy to access the value of nested sub-views.

So here is the pattern statement. **Use LayoutView to create sub-views over simple ItemView.**

# About Controller

If you come from vanilla Backbone, you are not sure what the role of Controller in Marionette. Unforunately, official doc doesn't help you either.

Quoted from [here](https://github.com/marionettejs/backbone.marionette/blob/master/docs/marionette.controller.md)
> Its name can be a cause for confusion, as it actually has nothing to do with the popular MVC architectural pattern. Instead, it's better to think of the Controller as a base object from which you can build.
>
> Controllers should be used when you have a task that you would like an object to be responsible for, but none of the other Marionette Classes quite make sense to do it. It's a base object for you to use to create a new Class altogether.

This makes you further puzzled. How should I use Controller?

Here is how I use Controller.

## Use Controller as integrator
This pattern is inspired by [the book](https://leanpub.com/marionette-gentle-introduction).

The role of Controller is to instantiate objects, access backend, and build complex vies: performs everything required to render a complete page in a browser.

Let's imagine that we want to render previous shopping cart page.

![](/images/shopping_cart1.png)

To render the page, several things has to be done.

- Instantiate views (layout views, item list views, etc)
- Retrieve items and prices (ajax call to backend server)
- Render views (call show() method of views)
- Error handling (when ajax call fail )

I will perform these things in a single method `showShoppingCart()` and this method goes to our imaginary ShoppingCart module.

```js
MyApp.module("ShoppingCart", function(ShoppingCart, MyApp, Backbone, Marionette, $, _){
    ShoppingCart.Controller = {
        showShoppingCart: function() {
            // implementation of this methods
        }
    }
})
```

Inside the method, you write codes that perform things that I mentioned above.

```js
MyApp.module("ShoppingCart", function(ShoppingCart, MyApp, Backbone, Marionette, $, _){
    ShoppingCart.Controller = Marionette.Controller.extend({
        showShoppingCart: function() {
            // Let's suppose that items and its prices are saved in backend.
            // So, we need to retrive them from backend server asynchronously.
            // Imaginary MyApp.request("items") and MyApp.request("price")
            // returns promise object that fetches items and prices respectively.
            var fetchingItems = MyApp.request("items");
            var fetchingPrice = MyApp.request("prices");

            // Instantiate a layout view that provides regions
            var layoutView = new ShoppingCartLayoutView();

            $.when(fetchingItems, fetchingPrice).done(function(items, prices){
                // Add callback method when ajax call is successfully done.
                // First, we will build collections
                var itemList = new ItemList([items]);
                var priceList = new PriceList([prices]);

                // Now we can instantiate views and pass colletion to them
                var itemListView = new ItemListView({collection: itemList});
                var priceListView = new PriceListView({collection: priceList});

                // At this point we are ready to render a complete paga for user
                layoutView.itemListRegion.show(itemListView);
                layoutView.priceListRegion.show(priceListView);
            });

            // We don't want user to see nasty error, so let's add error handling
            $.when(fetchingItems, fetchingPrice).fail(function() {
                // OopsView will show error message to user
                var oopsView = new OopsView({error: "oops, something went wrong"});
                layoutView.render(oopsView);
            });
        }
    });
})
```

The example above is pseudo code so it doesn't work, but should be easy enough to demonstrate my idea.

As you can see, my `showShoppingCart()` method does everthing including error handling required to show a shopping cart page to user. Now, somebody must call the method. Who will it be?

It is the responsibility of `Marionette.AppRouter`. Router is the first one that responds when user enters your app. It's job is to look at url and call Controller's methods. Here is how you can pass your `ShoppingCart.Controller` to router.

```js
// First we need to define router
MyApp.Router = Marionette.AppRouter.extend({
    appRoutes: {
        "shopping_cart": "showShoppingCart",
    }
});

// Instantiate controller object
var shoppingCartController = new ShoppingCat.Controller();

// Register controller
new MyApp.Router({
    controller: shoppingCartController
});
```

Now, if user accesses `/shopping_cart` page, router will call `showShoppingCart()` method of `ShoppingCat.Controller`.

There is one implicit thing involved here. The object you pass to `controller` property of router must implement the method that you define in `appRoutes` objects.

In this case, since you define `"shopping_cart": "showShoppingCart"` in routing, `shoppingCartController` object must implement `showShoppingCart()` method, which is exactly how we implemented our controller earlier.

If you have experiences with server side MVC frameworks, like RoR, then you may notice that I am using the Marionette Controller like the way I use ActionController of Rails: called by router and does everything needed to render a view. Isn't this contradictory to what [official doc]([here](https://github.com/marionettejs/backbone.marionette/blob/master/docs/marionette.controller.md) says?

***Yes and no***. Yes, because it is used by router and both performs similar tasks. No, because the role of server side Controller is providing HTTP terminal point whereas client side Controller does not do this.

Anyway, I am currently satisfied with the way I use Controller.

So here is the pattern statement. **Use Controller to provide public methods to router that integrates different componetns**

## Add event listener on view inside Controller
I often ask this question myself: *Is it ok that my View does ajax call?* Can you imagine what I am talking about? The View that makes ajax call looks like this.

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

So, the code above does:

- Listens on `click` event and call `submitForm` method
- Makes ajax call to save form data to backend server
- Execute callbacks for ajax call

This is perfectly valid code in Backbone. However, having Controller in Marionette now, I'd rather want to push this task to Controller and let View simply listening and triggering event.

```js
FormView = Marionette.ItemView.extend({
    events: {
        'click button.submit': 'submitForm'
    },

    submitForm: function() {
        this.trigger("form:submit");
    }
});

FormController = Marionette.Controller.extend({

    showForm: function(options){
        var formView = FormView.new();

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

        MyApp.region.show(formView);
    },
});
```

So, I simply move codes from View to Controller. What's the benefits of doing this? One thing is that View code gets slimed and it can focus on the mapping of DOM and event.

Our Controller instead gets messy but that's the tradeoff. After all, that's how Controller is designed to be used: ***put things that fit nowhere else***.

It also makes sense to put codes to Controller because Controller has more accesses to other components than a view. Imagine a case where you have to access other views inside the event listener.
You can't access other views directly from a view. Instead, you need to use App level event publishing to archive this.

On the other hand, Controller has an access to everything needed to render the page so it can easily talk to other views.

Before, finishing this section, let me slightly imporve the code above. We will use [View.triggers](https://github.com/marionettejs/backbone.marionette/blob/master/docs/marionette.view.md#viewtriggers).

Here is the new version of View:

```js
FormView = Marionette.ItemView.extend({
    triggers: {
        'click button.submit': 'form:submit'
    }
});
```

We removed `submitForm` method and `event` objects. Instead, we use `trigger` object that does the two thing at the same time: *listen on events* and *trigger events*. Now, our view is much cleaner than before!!

So here is the pattern. **Add event listener on views inside Controller**

<!--
## Responsibility of MVC Components

Marionette brings consistency to your Backbone app. However, this is not enough if you are working in a team. Marionette still allows developrs to write codes whatever they want which rapidly makes your project spaghetti. It's important to understand your codes must go where. To understand this, let's clarify the role of MVC components in Marionette.

### M
The responsibility of model is clearer than other components, so I will go quickly.

#### Business logic
Just like model of server side, this is the place where you write code for business logic.

**Ex.** Calculate and return total price by adding tax to sub-total.

#### Ajax ####
When modle is mapped to external resource, it fetches resource from external servers.

#### Validation ####
Before modle is saved into exteranl database, it needs to validate data. When validation fails, it must notifies by using event. View must respond to validation event and notifies user.

### V
This is the place which easily becomes chaotic because vanilla Backbone puts much burden to View.

#### Render html
When you define View, you will speficy which template to use. Template value is fetched by a model that View holds and rendered as html document.

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
>
> Controllers should be used when you have a task that you would like an object to be responsible for, but none of the other Marionette Classes quite make sense to do it. It's a base object for you to use to create a new Class altogether.

So, as the doc says, Controller is the place where you can put things that fit nowhere else.

But, what exactly are they?

#### Executing action for router
When user enters your app, router will invoke controller actions. Let's assume you have controller like so:

```js
MyApp.module("ModuleA", function(ModuleA, MyApp, Backbone, Marionette, $, _){
    ModuleA.Controller = Marionette.Controller.extend({
        listItem: function() {
            // Some codes to show item here
        }
    })

    return ModuleA
})
```

Now you can pass your controller to `controller` property of router.
```js
MyApp.module("ModuleA", function(ModuleA, MyApp, Backbone, Marionette, $, _){
    var controller = new ModuleA.Controller();

    ModuleA.Router = Marionette.AppRouter.extend({
        controller: controller,
        appRoutes: {
            "items": "listItem"
        },
    });
})

```
Then when user enters your app from `/items`, router will execute `controller.listItem()` method which is defined at `ModuleA.Controller`.

So, as you can see, I am using Controller pretty much like the way I use Rails ActionController. I don't see no reason why this is wrong even if doc says it is nothing to do with server side MVC.

#### Assemble things together
You saw how router invokes Controller methods. Now, let's look at `listItem()` method. The method looks something like this:

```js
listItem: function() {
    var deferredFetch = MyApp.request("items")
    var collectionView = new ItemCollectionView()

    // Let's assume you have Spinner class.
    // This will show loading spinner until items are fetched from server.
    MyApp.mainRegion.show(new Spinner())

    $.when(deferredFetch).done(function(items) {
        var collectionView = new ItemCollectionView({collection: items});

        // Once fetching is done, show items
        MyApp.mainRegion.show(collectionView)
    })/

    $.when(deferredFetch).fail(function() {
        // Let's assume you have OopsView.
        // This will show "oops, something went wrong" when fethcing fails
        var oopsView = new OopsView()
    });
```

As you can see, `listItem()` method interacts many things. It also takes care of showing loading spinner as well as error handling. This is the main responsibility of Controller. It assemble many parts defined in different modules and control the flow of action.

-->
