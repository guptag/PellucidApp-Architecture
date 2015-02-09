Front End Architecture of the Pellucid Analytics's Web/ipad Application

## Background

Our front-end is a complex single-page web-application (SPA) supported in all major browsers and also packaged as a native application on the iPad (using Cordova). Some of the core requirements of our application include:

* iPad app must feel and respond like a native app.
* Flexible way to share code between desktop and mobile and also should be easy to diverge when needed.
* Application should be easily extensible to quickly add new views and features.

These days, to build a SPA, there are lot of JavaScript MVC frameworks out there. Two years ago, our choices were limited. Also, considering the emphasis on our iPad App, we decided to build a custom light-weight UI framework (LUI) with promises at its heart. In this blog post, I would like to provide some technical details of our front-end architecture. I hope some of these ideas might help others trying to solve similar problems in their applications.

## UI Tree and Modular Views

LUI's main emphasis is views. It composes the entire application as a tree of panels (or views). There are two types of panels, Container panels and Leaf panels. Container panels, as the name suggests, holds other panels and won't render any other UI elements within them. Whereas, Leaf panels are responsible for getting the data from models and rendering it using a template library. They also take care of binding/unbinding the UI elements with various event handlers. The core principle here is the panel that created the DOM element is responsible for listening any events and updating it (whenever needed). This approach nicely scopes the responsibility of each view to the part of the UI tree that it rendered. Scoping the DOM mutations only to the view that it originally rendered greatly helped us to quickly debug and fix any UI issues with confidence.

Here is an illustration of how our views are structured. 
![screen shot 2014-12-04 at 12 23 13 am](https://cloud.githubusercontent.com/assets/8048953/5294049/da8040c6-7b4b-11e4-9e75-7db04f0366a5.png)


Our folder structure matches with the view hierarchy. Each leaf (view) folder contains a script (view code), style (CSS) and a handlebar template (HTML).

### Sample code for the container panel

``` javascript
var Q = require('q'),
    Lui = require('lui'),
    TopNav = require('./topnav'),
    Sidebar = require('./sidebar'),
    Content = require('./content');

function AppMain () {
    return Lui.Panel({
        name: "app-main",
        views: { /* container panel holds other views */
            topnav: TopNav(),
            sidebar: Sidebar(),
            content: Content()
        },
        layout: "app-main",
        init: function(context) {
            // initialization code
        },
        before: function () {
            // panel is becoming visible
            // show the child panels

            // return a promise that composes
            // the promises from each child panel
            return Q.all([
                this.makeVisible("topnav"),
                this.makeVisible("sidebar"),
                this.makeVisible("content")
            ]);
        },
        after: function () {
           // after panel becomes visible
        }
    });
}
module.exports = AppMain;
```


### Sample code for the leaf panel

``` javascript
var Lui       = require('lui'),
    Q         = require('q');

function TopNav () {

    return Lui.Panel({
        name: "topnav",
        template: tmpl,
        layout: "topnav",
        init: function( context ) {
            var self = this;

            this.router = context.router;
        },
        before: function () {
            var self = this;

            // render the template
            Lui.$.html(this.elt, this.template());

            // abstract each topnav DOM element into a Button instance
            this.navButtons =  _.chain(Lui.$("#topnav .nav-button"))
                                .map(function(el) {
                                    var luiButton = Lui.ui.Button({
                                        elt: el,
                                        options: {
                                            viewName: Lui.$.attr(el, "view")
                                        },
                                        ƒ: function () {
                                            // when top nav link is clicked, switch the app to that view
                                            return self.selectView(this.options.viewName);
                                        }
                                    });
                                })
                                .value();
            return Q.resolve(this.elt);
        },
        selectView: function (viewName) {
            // navigate to the view
            this.router.navigateTo(viewName);
        }
    });
}

module.exports = TopNav;
```

### App Initialization

When the application is loaded for the first time, the LUI framework starts with root panel and traverses down the entire tree recursively showing the descendent panels until it reaches the leaf panels (calls `before` method on each panel). This initialization step is optimized only to load the panels that are visible on the UI (for e.g the Dashboard is shown by default when the app starts, in that scenario company/deck/library panels will stay hidden and not loaded). When control reaches the Content panel (within app/main), it will initialize only one child i.e dashboard:

``` javascript
before: function () {
 return this.makeVisible("dashboard");
}
```

## Layouts

We have discussed how panels are organized in the code, now let's look into how they are rendered on the screen. Every panel in the system is an absolutely positioned "div" element with default `width`/`height` set to 100% and `top`/`left` set to 0. This applies for all of the top level panels (both container panels as well as leaf panels). The content within leaf panel is custom templated HTML and is styled locally within the view. Top level panels can acquire custom dimensions and positions via a property called "Layout".

Application maintains another tree ("Layout Tree") in which each node holds the information about position and dimension of a rectangle. The root node of this tree contains window dimensions. Each node's position and dimension is calculated according to their definition (which can be relative to the parent or root).  The idea here is, layout definitions are added based on the designed wireframes. The UI panel can pick a layout definition and inherit it's top/left/width/height properties from the layout. When that panel becomes visible, Lui will assign the layout dimensions to the panel.

When the window is resized,
  * Layout tree is traversed from the root and all dimensions are updated accordingly as per the node definitions.
  * UI tree is traversed and all visible panel's dimensions are updated as per their layout properties.

Maintaining a separate layout tree helps in two ways:
  * Finding panel dimensions on live DOM will force browser to recalculate the layout and it can lead to layout trashing.
  * Different panels can share the same layouts. The layout calculation can happen only once and the values can be assigned to the corresponding panels whenever they become visible.

``` javascript
// Layout definition
Lui.Layouts.add("topnav_layout", function (w /* parent layout's width */, h, W /* Window width */, H) {
        return {
            x : 10, /* margin-left the parent */
            y : 20, /* margin-top from the parent */
            width : 10 * w / 100, /* 10% of the parent's (app) width */
            height : h /* match the height of the parent */
        };
    }, "app" /* parent layout */);

```

``` javascript
return Lui.Panel({
        name: "topnav",
        template: tmpl,
        layout: "topnav_layout", /* This panel's width/height/top/left will always be in sync with "topnav" layout node */
        // ..
   });
```

![screen shot 2014-11-26 at 9 23 19 pm](https://cloud.githubusercontent.com/assets/8048953/5211531/87a1d2f0-75b2-11e4-83c8-1f0ae5a8de1d.png)

## Communication between views

In JavaScript world, it is a common practice to use pub/sub type of events to communicate between various components without the tight coupling. The approach works well for many scenarios especially when there are tens or hundreds of components involved in the system and they are dynamically changing. But pub/sub model makes the code more difficult to understand and harder to debug. Also, sequencing the execution of handlers cannot be easily done with this approach.

The other alternative is to use Promises. They provide the benefit of the loose coupling, makes it easy to understand the code flow with explicit calls to subscribers and provides a way to sequence the execution. This approach is more suitable when the components are more static. In our case, all our views and their hierarchy are very well predefined.

Here is a scenario:

Let's say we have a dashboard with a list of companies. Header and Left Nav shows the total count of companies in dashboard and need to be synced whenever a new company is added to the dashboard.

If we implement this in the pub/sub eventing model:

  * Dashboard publishes the "`company:added`" event.
  * Header listens for "`company:added`" event and it updates the count in its   view.
  * LeftNav listens for "`company:added`" event and it updates the count in its view.

For someone trying to debug the dashboard code, publishing "`company:added`" the event in dashboard.js is pretty much the end of the road. Now they need to search for "`company:added`" event across all files to understand the dependencies. Imagine debugging a code-base with ton of events floating around.

Now let's implement the same scenario using promises. The approach turns the above eventing flow upside-down:

  * We define two "deferreds" in a global hash (ignored namespaces for brevity)

``` javascript
     Header.updateCount = new Q.defer();
     LeftNav.updateCount = new Q.defer();
```

  * Header.js resolves `Header.updateCount` deferred with a method in its view

``` javascript
    // in init (resolve only once)
    Header.updateCount.resolve(this.updateCompanyCount.bind(this));

    this.updateCompanyCount: function (count) {
        // update DOM
    }
```

  * LeftNav.js resolves `LeftNav.updateCount` deferred with a method in its view

``` javascript
    // in init (resolve only once)
    LeftNav.updateCount.resolve(this.updateCompanyCount.bind(this));

    this.updateCompanyCount: function (count) {
        // update DOM
    }
```

  * Dashboard.js - When a new company is added, Dashboard will call both the promises when a new company is added (which basically executes the underlying methods in the other views)

``` javascript
    this.addCompanyToDashboard: function(company) {
          var self = this;

          // Makes backend call (which returns a promise)
          this.dashboardModel.addCompany(company)
                .then(function(companies) {
                      // upon successful save
                      // refresh dashboard, header and leftnav
                      self.refreshDahboardView(companies);
                      Header.updateCount.promise.fcall(companies.length);
                      LeftNav.updateCount.promise.fcall(companies.length);
                });
    }
```

While this involves more steps, the exclusive calls to the necessary updates (from dashboard) makes it very easy to understand and debug. It matches exactly with the mental model of the application flow.

The important point to note here is that deferreds are resolved with bounded methods (not many examples about promises show this useful feature). The execution of the underlying resolved methods happens asynchronously. Dashboard doesn't need to know how (and when) those deferreds are resolved; `Header` and `LeftNav` don't need to know about the components calling their exposed methods via deferreds. The updates can also be sequenced in an order if needed:

``` javascript
// Makes backend call (which returns a promise)
this.dashboardModel.addCompany(company)
    .then(function(companies) {
        // upon successful save
        // refresh dashboard, header and leftnav
        Q.fcall(function () {
           self.refreshDahboardView(companies);
        }).then(function () {
           Header.updateCount.promise.fcall(companies.length);
        }).then(function () {
           LeftNav.updateCount.promise.fcall(companies.length);
        }).done();
});
```

We still use regular broad-casted events in some specific scenarios but in the context of communication between views, we found promises are a lot more helpful.

## Conclusion

I think, we came far with the choices we've made and are pretty happy with the result so far. We are adding lots of new functionalities as per our business needs without much changes to the underlying architecture. You can sneak peek our application [here](http://www.finovate.com/spring14vid/pellucid.html).

If we had to rewrite the entire application again with the choices we have now, we might probably leverage a framework like [React](http://io.pellucid.com/video/fsb-building-user-interfaces-with-react).

Thank you for taking time and reading this far. Hope you've found this information useful.
