# Webpack Migration Walkthrough

## General Procedure
1) Add import/export portions to javascript files
2) Add necessary import to less files (if present)
3) Create index file to combine folder files
4) Continue down folder structure repeating the above steps until you reach leaf folders
5) Walk back up tree and import subfolder index files into parent folder index file
6) (optional) Clean up messy areas with es6/es7 syntax.

# Step 1: Define Imports/Exports

## Summary
In es6, modules are defined similar the node `require` api, and it'show you go about scaling javascript architecture in a modern, es6 environment. Unfortunately, since many browsers (inside of our support range) don't support es6 modules, we use build tools to transpile modern javascript to javascript that old browsers can run. The tools we're specifically using are **Babel.js** and **Webpack**.

Babel is responsible for transpiling our modern javascript into backwards compatible javascript. Webpack is responsible for walking through our architecture and creating a bundle that represents everything a user will need to run the application in their browser. It's, essentially, a elegant way of doing what grunt has always done for us. It also uses es6 syntax to determing what the bundle should look like, which means that eventually, once browsers catch up, webpack can be entirely removed from the equation and browsers can handle bundling our code all by themselves. Or at least, we can dream, right?

An important thing to understand here is that when using webpack, its a given that es6 is being used. We can decide _when_ and _when not_ to take advantage of new es6 features, but even the `import` and `export` api is, itself, an es6 spec. So, throughout this migration, we won't be deciding _if_ we should use es6, we'll be deciding _when_ we should use es6.

## Import/Export overview

_(If you already understand the import/export api, you can skip this)_

When we're performing this step, we basically isolate the essential "thing" that any given javascript file is creating or providing to the module and we denote it as an `export`. That can then be listed as an `import` in other files. For example `import component from './component.js'`, which is essentially saying "Make sure that my sibling file `component.js` is included in the production bundle, and let me access it in this file through the variable `component`". Similarly, you can mark mutliple things as `export` in any given file, and import multiple things from that file with the syntax `import {a, b} from './component.js`. Which is saying, "I expect component.js to have marked two named variables, `a` and `b`, as exported variables, and I want to access them here."

### Small example

Given the file below.

**constants.js**
```
export default  {
    THIS_IS_A_CONSTANT: 1,
    THIS_IS_ANOTHER_CONSTANT: 2
}

export let a = 24;

export let b = 78;
```

Another file could say `import constantObject from './constants.js'`, and this would import the object listed at the top of `constants.js`. This is because that object was specified as the default export. If the importing file instead said `import {a, b} from './constants.js'`, then the other variales would be imported instead. The `{}` "brace" syntax is a way to import specific, named exports. Whereas the "braceless" syntax says, "give me whatever the default export is and name it this variable." You can also import a specific, named export and rename it like so: `import {a as hoursInADay, b as threeDaysPlus4Hours} from 'constants.js'`. This is the equivalent of the previous export, but the import is assigned to a different variable when used in the importing file.

You can also group all of a folders import/exports together by specifying an `index.js` file in any given folder. If I have a folder named `component`, I can create an `index.js` file that imports the contents of the folder and rexports them as a "grouped" object. Then, any other file can simply use the folder name in an import statement (`import './component'`) to access the `index.js` file. This is a good way of establishing code architecture hierarchy and organizing things in a way that can be accessed by other pieces of code in sensible ways.

## Import/Exporting angular code

Since angular wasn't written in a es6-based era, there's not a clear-cut defacto approach to composing angular apps with es6-module architecture. That being said, several reasonable approaches have immerged from the dev community, so here's an example of each angular "type" (service, component, etc) using es6 imports/exports.

### In Depth Example

For a good walkthrough, we'll take a real example from the `pubs` module and we'll walk through migrating it over to webpack/es6. We'll use `pubPublicationResultsView` (found under (`core` > `reusableComponents` > `misc`) as our specimen, because it has a subfolder and several different kinds of angular code, so it should work well as an example.
 
#### File Structure
```| pubPulicationResultsView
   |--| pubPublicationResultsFilters
     |--> pub-publication-results-filter.less
     |--> pubFilteringService.js
     |--> pubPublicationResultsFilters.js
     |--> pubPublicationResultsFilters.tpl.html
   |--> pub-publication-results-view.less
   |--> pub-publication-results-view.mobile.less
   |--> pubHasFeaturedInterface.js
   |--> pubPublicationResultsView.js
   |--> pubPublicationResultsView.tpl.html
   |--> pubPublicationResultsViewHelper.js
```

First, lets go through the files in the root folder. Starting with the javascript. The component, specifically. The order I typically migate files in is:

1) Component
2) HelperService
3) Other miscelaneous Angular files (constants or services or filters)
4) Less files.

#### Sample file

**pubPublicationResultsView.js**
```js
(function () {
    'use strict';

    angular
        .module('pub')
        .component('pubPublicationResultsView', {
            template: function ($templateCache) {
                "ngInject";
                return $templateCache.get('pubPublicationResultsView.tpl.html');
            },
            controller: pubPublicationResultsViewCtrl,
            bindings: {...},
            require: {...}
        });

    function pubPublicationResultsViewCtrl($element, $timeout, $filter, ...) {
    }
})();
```

First thing we can do is drop the immediately invoked function. That was just a method we were using to avoid accidentally putting variables on the global window object. Since es6 modules are inherently isolated in scope, we don't have to worry about that. So, let's drop it.

```diff
-(function () {
-    'use strict';
angular
    .module('pub')
    .component('pubPublicationResultsView', {
        template: function ($templateCache) {
            "ngInject";
            return $templateCache.get('pubPublicationResultsView.tpl.html';
        },
        controller: pubPublicationResultsViewCtrl,
        bindings: {...},
        require: {...}
    });

function pubPublicationResultsViewCtrl($element, $timeout, $filter, ...) {
}
- })();
```

Next we're going to remove the code registering the component definition with the angular module. We're going to move the work of tying the different pieces of code to the angular framework into the `index.js` file, so that we can leave clutter out of our main code files. We're also going to mark the component definition object as our `export default`.

```diff
-angular
-    .module('pub')
-    .component('pubPublicationResultsView', {
+ export default {
        template: function ($templateCache) {
            "ngInject";
            return $templateCache.get('pubPublicationResultsView.tpl.html';
        },
        controller: pubPublicationResultsViewCtrl,
        bindings: {...},
        require: {...}
+};
-   });

function pubPublicationResultsViewCtrl($element, $timeout, $filter, ...) {}
```

Next, we can ditch the complex template definition and just use `import` to import the template.

```diff
+import template from './pubPublicationResultsView.tpl.html';
export default {
+        template,
-        template: function ($templateCache) {
-           "ngInject";
-            return $templateCache.get('pubPublicationResultsView.tpl.html';
-        },
        controller: pubPublicationResultsViewCtrl,
        bindings: {...},
        require: {...}
+};
-   });

function pubPublicationResultsViewCtrl($element, $timeout, $filter, ...) {}
```

Notice that we didn't give `template` any assignment in the object expression. That's because, in es6, `{template}` is short hand for `{template: template}`. Since we've imported our template under the variable name `template`, we can write it in this cleaner, shorthand syntax. In fact, we're going to take that a step further and rename our controller function to just be `controller`. The javascript scope is isolated by es6 modules, so we don't have to worry about any naming collisions, and it will make our component definition cleaner.

```diff
import template from './pubPublicationResultsView.tpl.html';
export default {
+       template, controller
-       template,
-       controller: pubPublicationResultsViewCtrl,
        bindings: {...},
        require: {...}
+};
-   });

+function controller($element, $timeout, $filter, ...) {}
-function pubPublicationResultsViewCtrl($element, $timeout, $filter, ...) {}
```  

Aww yeah. Das real nice. Let's move on to the helper now!

**pubPublicationResultsViewHelper.js**
```js
(function() {
    'use strict';

    angular
        .module('pub')
        .service('pubPublicationResultsViewHelper', pubPublicationResultsViewHelper);

    /*@ngInject*/
    function pubPublicationResultsViewHelper(utils, utilityService, communicationTypeService, pubFilteringService, pubSearchService, pubWorkflowStages, pubAdvancedSearchCriteria) { ... }
})();
```

Procedure here is very similar, except the main export of the file won't be an object, because this is a service, which angular expects to be a function. So our service ends up looking like this:

```diff
-(function() {
-   'use strict';
-
-   angular
-        .module('pub')
-        .service('pubPublicationResultsViewHelper', pubPublicationResultsViewHelper);
-
-   /*@ngInject*/
-    function pubPublicationResultsViewHelper(utils, utilityService...) { ... }
+/*@ngInject*/
+export default function pubPublicationResultsViewHelper(utils, utilityService...) { ... }
-})();
```

Again, don't worry about registering the service on the actual angular module yet, we'll do that in the `index.js` file. For now just worry about exporting the function. Also, for some reason we started adding `return service;` to the end of our services at some point. That's not supposed to be there. Angular factories (different from services) are supposed to return an object, which may be where the confusion began. Angular services, however, are constructor function. They add things to the `this` instance, but they don't return anything. So, if you see `return service;` at the end of any services, remove it.

Moving on to the directive!

**pubHasFeaturedInterface.js**
```js
angular
    .module('pub')
    .directive('pubHasFeaturedInterface', pubHasFeaturedInterface);

function pubHasFeaturedInterface() {
    return {
        restrict         : 'A',
        controller: pubHasFeaturedInterfaceController,
        bindToController : true
    };
}

function pubHasFeaturedInterfaceController($q, componentHelperService) { ... }
```

Same charade here, except that a directive is a function which returns a factory that produces a object. A.k.a, it's a function that returns a function that returns an object. So the export looks like this.

```diff
-angular
-    .module('pub')
-    .directive('pubHasFeaturedInterface', pubHasFeaturedInterface);
-
-function pubHasFeaturedInterface() {
-    return {
-        restrict         : 'A',
-        controller: pubHasFeaturedInterfaceController,
-        bindToController : true
-    };
-}
+
+export default () => () => ({
+    controller,
+    restrict : 'A',
+    bindToController : true
+});
-function pubHasFeaturedInterfaceController($q, componentHelperService) { ... }
+function controller($q, componentHelperService) { ... }
```

#### Less files
The last thing that needs to be dealt with in this folder is less files. Variables and mixins are no longer global, so if a file uses color variables, it needs to add `@import '~@pub/styles/variables';`, and if it uses mixins, then it should import `@import '~@pub/styles/mixins';`. Additionally, anything that uses a font-awesome icon needs `@import '~font-awesome/less/font-awesome';`, and icmoon icons need `@import '~@icomoon';`.

So, in our case, `pub-publication-results-view.less` needs `'~@pub/styles/variables';` and `@import '~@icomoon';`. And `pub-publication-results-view.mobile.less` needs `@import '~@pub/styles/mixins';`.

#### Handling Subfolders
In the case of nested components, this same process needs to be performed recursively down the folder structure. So we'll need to do everything we just did above, but in `pubPublicationResultsFilters`. Then we need to use an `index.js` to register all folder contents to the angular module, and make the folder accessible to other areas of the app.

Here's what the `index.js` file for `pubPublicationResultsFilters` (the child folder) looks like:

**pubPublicationResultsFilter/index.js**
```
import './pub-publication-results-filters.less';
import service from './pubFilteringService';
import filter from './pubPublicationResultsFilters';

export default module => {
    module.service('pubFilteringService', service)
        .filter('pubPublicationResultsFilters', filter);
}
```

This exports a function that, when provided with an angular module as an argument, will register `pubFilteringService` and `pubPublicationResultsFilters` onto that module. This was why we didn't worry about registering anything to the angular module previously. We do all the angular work here, and keep the actual code files as simple as possible.

Small note: if no extension is provided to an import, `.js` is assumed.

Now we need to handle the parent folder `index.js` file

**pubPublicationResultsView/index.js**
```
// import main component
import 'pub-publication-results-view.less';
import 'pub-publication-results-view.mobile.less';
import featuredInterface from './pubHasFeaturedInterface';
import component from './pubPublicationResultsView';
import helper from './pubPublicationResultsViewHelper';

// import subcomponent
import registerPublicationResultsFilters from './pubPublicationResultsFilters';

export default module => {
    // register main component
    module.component('pubPublicationResultsView', component)
        .service('pubPublicationResultsViewHelper', helper)
        .directive('pubHasFeaturedInterface', featuredInterface);

    // register subcomponent
    registerPublicationResultsFilter(module);
}
```

Like the other index file, we export a function that, when provided with an angular module, registers local angular components on to the provided module. It also imports the function exported by `pubPublicationResultsFilter` and provides it with the same module it was provided with. Now anyone who imports `pubPubliationResultsView` and calls its registration function with their angular module can use all the filters, services, and components within. They can also choose to _only_ import the `pubPublicationResultsFilters` folder instead, if they like.

### Verifying Your Work

Its good to run `npm run webpack` whenever you finished a large chunk of work. You can't really know it completely works, but you should ensure you don't get compilation errors. Every time I've run webpack after a large chunk of work it's thrown about 25 errors based on typos or imports I've missed.

Also ensure that there's actually an import path from the root module file to the code you've just migrated, because if no file imports the code you just migrated is actually in the import tree that webpack walks through, it won't touch that code and, therefor, won't give you any relevant compilation errors about it.

### Miscellaneous Changes For Cleaner Code
The next step I typically take is completely optional, but I've found has cleaned up/standardized/decreased the line count of our codebase _substantially_.

#### Standardizing our usage of ngInject

We're pretty inconsistent in the way we use our `ngInject` directive, which handles angular dependency injection. Sometimes we us

```
function(serviceName) {
    "ngInject";
    ...
}
```

Other times, we've used:

```
/* @ngInject */
function (serviceName) {}
```

So one small thing I did, just for consistency, was convert everything to the latter syntax.

#### `{componentName}Helper` name clutter

We have a common design pattern to move complex business logic into a helper service named `{componentName}Helper`, that only contains no-side-effect functions. This has been a good and usefule design pattern, but sometimes the service name becomes _really_ long, in the example we used in this guide, the helper name is `pubPublicationResultsViewHelper`. WHEW! That's a pain, and it hurts readability and makes line wrap much more common than is necessary through the file. So another thing I've done in every file I've touched is alias the helper service simply as the variable `helper` in each controller.

A.k.a

```js
function controller(pubReallyLongNameForComponentHelper) {
    let helper = pubReallyLongNameForComponentHelper;
}
```

And then find and replace every instance of `pubReallyLongNameForComponentHelper.{methodName}` with `helper.{methodName}`.

#### Standardize lifecycle hook formats

Another area of inconsistency that is pretty easy to standardize, now that we're touching every file. Component lifecycle hooks currently are kind of varied in the way they're executed.

Sometimes they look like this:

```
    $ctrl.$onInit   = componentHelperService.onInit($ctrl, function() {
        // some initialization code,

        // some other code
    });
    $ctrl.$onChanges = componentHelperService.onChanges($ctrl, function() {
        // some update code
    });
    $ctrl.$postLink = componentHelperService.postLink($element);
    $ctrl.$onDestroy = componentHelperService.onDestroy(function () {
        // some cleanup code
    });
```

Other times it looks like this

```
    $ctrl.$onInit   = componentHelperService.onInit($ctrl,  function() {
        // some initialization code,

        // some other code
        updateThisComponent()
    });
    $ctrl.$onChanges = componentHelperService.onChanges($ctrl, updateThisComponent);
    $ctrl.$postLink = componentHelperService.postLink($element);
    $ctrl.$onDestroy = componentHelperService.onDestroy(cleanup);

    // ...later in the code under *private functions*

    function update this component() {
        // some update code
    }

    function cleanup() {
        // clean up code
    }
```

I've been standardizing the way we do lifecycle hookes as I've touched files. All lifecycle hooks are named functions that go under private functions. If there is an initialzation function, its name is `initialize`. If there is an `update` function, it's named update. If there is a destroy function, it's named `cleanup`. If there is a postLink function, it's named `link`. If there is an update function but no initialization logic, for some odd reason, create an `initialize` function anyways and have it simply call `update()`. That way, if initialization logic is added later there will be an obvious place to put it.

So it should always look like this:
```js
    $ctrl.$onInit    = componentHelperService.onInit($ctrl, initialize);
    $ctrl.$onChanges = componentHelperService.onChanges($ctrl);
    $ctrl.$postLink  = componentHelperService.postLink($element);
    $ctrl.$onDestroy = componentHelperService.onDestroy();

    // ...later in the code under *private functions*

    function initialize() {
        // some initialization code
    }
```

Or this:

```js
    $ctrl.$onInit    = componentHelperService.onInit($ctrl, initialize);
    $ctrl.$onChanges = componentHelperService.onChanges($ctrl, update);
    $ctrl.$postLink  = componentHelperService.postLink($element);
    $ctrl.$onDestroy = componentHelperService.onDestroy();

    // ...later in the code under *private functions*

    function initialize() {
        // some initialization code
        update();
    }

    function update() {
        // some update code
    }
```

Or this:

```js
    $ctrl.$onInit    = componentHelperService.onInit($ctrl, initialize);
    $ctrl.$onChanges = componentHelperService.onChanges($ctrl, update);
    $ctrl.$postLink  = componentHelperService.postLink($element);
    $ctrl.$onDestroy = componentHelperService.onDestroy(cleanup);

    // ...later in the code under *private functions*

    function initialize() {
        // some initialization code
        update();
    }

    function update() {
        // some update code
    }

    function cleanup() {
        // some clean up code
    }
```

NOT this:

```js
    $ctrl.$onInit    = componentHelperService.onInit($ctrl, update);
    $ctrl.$onChanges = componentHelperService.onChanges($ctrl, update);
    $ctrl.$postLink  = componentHelperService.postLink($element);
    $ctrl.$onDestroy = componentHelperService.onDestroy();

    // ...later in the code under *private functions*

    function update() {
        // some update code
    }

```

Or this:

```js
    $ctrl.$onInit    = componentHelperService.onInit($ctrl, initialize);
    $ctrl.$onChanges = componentHelperService.onChanges($ctrl, initialize);
    $ctrl.$postLink  = componentHelperService.postLink($element);
    $ctrl.$onDestroy = componentHelperService.onDestroy();

    // ...later in the code under *private functions*

    function initialize() {
        // some update code
    }

```

#### Cleaner callbacks

One of Es6's biggest new features is arrow functions, which dramatically cleanup and simplify anonymous functions. A few examples:

**es5 syntax**
```js
var someVar = null;
var someList= null;
somePromise().then(function(value) {
        someList = value;
        return someList.filter(function(listItem) {
            return listItem.someCondition === true;
        })
    }).then(filteredList) {
        somVar = filteredList[0]
    });
```

**es6 syntax**
```js
let someVar = null;
let someList = null;
somePromise.then(value => {
    someList = value;
    return someList.filter(listItem => listItem.someCondition === true)
}).then(filteredList => someVar = filteredList[0]);
```

It definitely helps clean things up. Note: the format `var => var = value` is only valid with a _single_ param used in a _single_ command. Multiple commands should use the braces like `var => { someCommand(); someOtherCommand()  }`, and multiple variables should be surrounded by parens, like `(var1, var2) => var1 + var2;`.

Here's a real diff from a messy function I refactored to es6.

```diff
-    function setStepAttributes(commInfo) {
+    function setStepAttributes({finalReleaseId, step: {internalCode: stepCode}}) {
+       const { FINAL_RELEASE_SECTION } = communicationInternalCodes;
+       $ctrl.finalReleaseId = +finalReleaseId;
-       $ctrl.finalReleaseId = +commInfo.finalReleaseId;
-       return $ctrl.communication.getHeader()
-           .then(function (header) {
-               var currentReviewer,
-                   activeStepHeaderData = header
-                       .NavigationGroups.filter(function (s) {
-                       return s.Code === communicationInternalCodes.FINAL_RELEASE_SECTION;
-                   })[0]
-                       .Steps.filter(function (s) {
-                       return s.Code === commInfo.step.internalCode;
-                   })[0];
+       return $ctrl.communication.getHeader().then(({NavigationGroups: sections}) => {
+           let currentReviewer,
+               {Contributors} = sections
+                   .filter(({Code}) => Code === FINAL_RELEASE_SECTION)[0].Steps
+                   .filter(({Code}) => Code === stepCode)[0];
-           if (activeStepHeaderData.Contributors.length < 1) {
-               currentReviewer = null;
-           } else {
-               currentReviewer = activeStepHeaderData.Contributors[0];
-           }
+           currentReviewers = (Contributors.length < 1) ?  null : Contributors[0];
-           $ctrl.reviewersForStep = activeStepHeaderData.Contributors;
+           $ctrl.reviewersForStep = Contributors;
            return currentReviewer;
        });
    }
```

So the final before and after is:

**before**
```js
    function setStepAttributes(commInfo) {
       $ctrl.finalReleaseId = +commInfo.finalReleaseId;
       return $ctrl.communication.getHeader()
            .then(function (header) {
                var currentReviewer,
                    activeStepHeaderData = header
                        .NavigationGroups.filter(function (s) {
                        return s.Code === communicationInternalCodes.FINAL_RELEASE_SECTION;
                    })[0]
                        .Steps.filter(function (s) {
                        return s.Code === commInfo.step.internalCode;
                    })[0];
            if (activeStepHeaderData.Contributors.length < 1) {
                currentReviewer = null;
            } else {
                currentReviewer = activeStepHeaderData.Contributors[0];
            }
            $ctrl.reviewersForStep = activeStepHeaderData.Contributors;
            return currentReviewer;
        });
    }
```

**after**
```js
    function setStepAttributes({finalReleaseId, step: {internalCode: stepCode}}) {
       const { FINAL_RELEASE_SECTION } = communicationInternalCodes;
       $ctrl.finalReleaseId = +finalReleaseId;
       return $ctrl.communication.getHeader().then(({NavigationGroups: sections}) => {
           let currentReviewer,
               {Contributors} = sections
                   .filter(({Code}) => Code === FINAL_RELEASE_SECTION)[0].Steps
                   .filter(({Code}) => Code === stepCode)[0];
           currentReviewers = (Contributors.length < 1) ?  null : Contributors[0];
           $ctrl.reviewersForStep = Contributors;
            return currentReviewer;
        });
    }
```

Which, not only is cleaner and more readable, but it's also about 40% fewer lines. You may have also noticed some odd assignment and parameter syntax in that last example, which leads us to our next subject.

#### Destructuring

Destructuring assignments is another big es6 feature. Basically, its the idea of using a pattern to perform a bunch of variable assignments at once. Here are a few examples:

**Array destructuring**
```js
// es5 syntax
let firstItem = array[0],
    otherItem = array[4];

// es6 syntax
let [firstItem,,,otherItem] = array;
```

**Object Destructuring**
```js
//es5
var code = StepObj.Code,
    reviewers = StepObj.Contributors
    mainReviewer = reviewers[0];

// es6
let {Code: code, Contributors: reviewers} = StepObj,
    [mainReviewer] = reviewers;

// or if we only want mainReviewer
let {Contributors: [mainReviewer]} = StepObj
```

It's a very concise format and can be used to make a lot of our much more readable and a lot less busy work of assigning and reassigning fields from objects.

You can also do this with arguments, which is what I did in the previous example. We didn't care about the whole commInfo object that was given to the function, we only wanted the `finalReleaseId` and `step.internalCode`. So, I destructured the parameter in the argument list to be `{finalReleaseId, step: {internalCode: stepCode}}`. Which says, "I want `finalReleaseId` from the object argument, and I want it available as `finalReleaseId` in this function (because I didn't specify a different name), and I also want the field `internalCode` from the `step` field, and I want it available in this function under the variable `stepCode`.

Very useful and very concise. For more information, go [here](https://wesbos.com/destructuring-objects/).

#### Controllerless Components (Dumb Components)

Often times, we create a component purely for the purpose of binding a template with styles, and perhaps passing down some data or callbacks. The component might look something like this:

```js
import template from './pubReviewReassign.tpl.html';
export default {
    template, controller,
    bindings: { reassignReviewer: '&' },
    require: { reviewerHelper: '^^pubReviewerHelper' },
};

/* @ngInject */
function controller($element, componentHelperService) {
    const $ctrl = this;

    /****************************************
     *        Controller Attributes         *
     ****************************************/
    /****************************************
     *        Controller API                *
     ****************************************/
    /****************************************
     *        Lifecycle Hooks               *
     ****************************************/

    $ctrl.$onInit    = componentHelperService.onInit($ctrl);
    $ctrl.$onChanges = componentHelperService.onChanges($ctrl);
    $ctrl.$postLink  = componentHelperService.postLink($element);
    $ctrl.$onDestroy = componentHelperService.onDestroy();

    /****************************************
     *        API Functions                 *
     ****************************************/
    /****************************************
     *        Private Functions             *
     ****************************************/
}
```

In this case *the controller can actually be ommitted*. In angular 1.5 and greater, the controller argument of a component object defaults to `angular.noop`. So, it's not a required field. Even our lifecycle hooks are unnecessary here, because their purpose is to use defensive binding to avoid a component mutating data it doesn't own. But since the component has no methods, it isn't even mutating data (*you shouldn't ever be mutating data in a template*).

So, in this case, this component can actually be reduced to this:

```js
import template from './pubReviewReassign.tpl.html';
export default {
    template,
    bindings: { reassignReviewer: '&' },
    require: { reviewerHelper: '^^pubReviewerHelper' },
};
```

Craaaazzzzy, right? This particular change can _majorly_ cut down our network bundle size!
