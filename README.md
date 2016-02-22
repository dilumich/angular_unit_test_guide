# angular_unit_test_guide

This is the AngularJS unit test guide written TypeScript which is used internally at PernixData.

## Framework

Here is a list of main frameworks or tools used for unit testing purposes.

- [Jasmine](http://jasmine.github.io/2.0/introduction.html): Jasmine is a behavior-driven development framework for testing JavaScript code. It does not depend on any other JavaScript frameworks. It does not require a DOM. And it has a clean, obvious syntax so that you can easily write tests.
- [Karma](http://karma-runner.github.io/0.13/intro/how-it-works.html): Karma is essentially a tool which spawns a web server that executes source code against test code for each of the browsers connected. The results for each test against each browser are examined and displayed via the command line to the developer such that they can see which browsers and tests passed or failed.
- [ngMock](https://docs.angularjs.org/api/ngMock): The ngMock module provides support to inject and mock Angular services into unit tests. In addition, ngMock also extends various core ng services such that they can be inspected and controlled in a synchronous manner within test code.
- [Jasmine-jquery](https://github.com/velesin/jasmine-jquery): jasmine-jquery provides two extensions for the Jasmine JavaScript Testing Framework: a set of custom matchers for jQuery framework and an API for handling HTML, CSS, and JSON fixtures in your specs.
- [karma-ng-html2js-preprocessor](https://github.com/karma-runner/karma-ng-html2js-preprocessor): This preprocessor converts HTML files into JS strings and generates Angular modules. These modules, when loaded, puts these HTML files into the $templateCache and therefore Angular won't try to fetch them from the server.
- [istanbul](https://github.com/gotwarlost/istanbul): istanbul is a JS code coverage tool that computes statement, line, function and branch coverage with module loader hooks to transparently add coverage when running tests. Supports all JS coverage use cases including unit tests, server side functional tests and browser tests. Built for scale.
- [remap-istanbul](https://github.com/SitePen/remap-istanbul) remap-istanbul is a tool for remapping Istanbul coverage via Source Maps and it can be used to remap js coverage to ts coverage.

Basically, Jasmine provides a basic unit testing framework and Karma executes test code in the configured environment. ngMock provides a way to load AngularJS modules, inject dependencies, and mock dependencies. Jasmine-jquery has additional jquery testing framework to help us work with Jquery. karma-ng-html2js-preprocessor is a tool to pre-load all templates which avoids runtime HTTP requests.

Some other tools are also used for additional functionalities but they are not important. For a full list of dependencies, please see dev dependencies in px-uilibs package.json file.

## Configurations 

### Build and Run Tests
'build-test' and 'test' are two tasks defined in px-uilibs/gulpfile.js. 

1. `gulp build-test` builds all typescript source code and test code
2. `gulp test` builds all source code first and runs all unit tests

### Testing Environment Configurations
Testing environment configuration is defined in px-uilibs/test/test-unit.conf.js file. See comments in this file for more infornmation.

### Add External Dependencies
1. add them to package.json
2. run `npm install`
3. include path to required files in `files` array in px-uilibs/test/test-unit.conf.js file.

### Debug in Browser
1. run `node_modules/karma/bin/karma start test/test-unit.conf.js`
2. click 'debug' button in the opened browser
3. open developer tools
4. refresh browser to rerun tests

## Test Coverage
### Generate code coverage
1. run `gulp test-coverage` (it builds and runs all unit tests and generates covearge report).
2. open px-uilibs/coverage/index.html in browser

### How it works
1. karma invoke istanbul coverage tool to generate coverage info in .json format (px-uilibs/coverage/coverage-final.json)
2. remap-istanbul remaps javascript coverage info to typescript coverage info in .json format (px-uilibs/coverage/coverage.json)
3. istanbul generates coverage report in html using remapped coverage info.

### Archieve 100% coverage
Ideally, all px-uilibs code should be tested. However, Some branches in JS code are typically hard, if not impossible to test. Use istanbul ignore syntax to enforce istanbul to ignore these branches. (see [this](https://github.com/gotwarlost/istanbul/blob/master/ignoring-code-for-coverage.md) for details)


## Testing Guide
### Genaral Strucuture
```typescript
/// <reference path="../../../../../typings/test/all.d.ts" />

module App {
    // service, filter, or directive name to be tested
    describe("UILibs_Test_Service", () => {

        // store variable shared for all test cases here
        let rootScope: IRootScope;
        let uilibs_Test_Service: UILibs_Test_Service;

        // load module to be tested here. 
        beforeEach(module('uilibs.service.test'));

        // inject dependencies here.
        // angular dependency names should be wrapped with _
        beforeEach(() => {
            inject((_$rootScope_: IRootScope, UILibs_Test_Service: UILibs_Test_Service) => {
                uilibs_Test_Service = UILibs_Test_Service;
                rootScope = _$rootScope_;
            });
        });
        
        it('method name: test case descriptipn', () => {
            // test goes here
        });
    })
}
```

### Monitor or Mock a function
Jasmine `spyOn` provides a way to monitor and mock function calls. See http://jasmine.github.io/2.0/introduction.html `spyOn` section for more details. Some special mocking examples are shown below:
```typescript
// mock jquery function
spyOn($.fn, height).and.callFake(() => {
    return 100;
});

// mock angular service function
// since angular services are maintained as singleton, just inject service and spyOn it. 
spyOn(http, 'get').and.callFake((url: string): Object => {
    return {
        success: (callback: Function): void => {
            callback('<div>{{content}}</div>');
        }
    };
});
```

### Mock an AngularJS Service
`ngMock` provides `$provide` service to mock an AngularJS service.
```typescript
// only provide methods or variables used in tested service
let serviceObj;

beforeEach(angular.mock.module(($provide: angular.auto.IProvideService) => {
    $provide.value('serviceName', serviceObj);
}));

// load tested service after
beforeEach(module('uilibs.service.name'));
```

### Work with Jquery
Since normally html elements are not inserted into DOM in unit test, global Jquery seletors don't work. Fortunately, jasmine-jquery provides fixture concept to fix it. Setting element as jquery fixture makes jquery element accessible for global 
```typescript
let element: Jquery = $(compile('<div id="adc">')(scope));

// doesn't work
$($('#abc');

// appendSetFixtures is needed when you need to set multiple fixtures at the same time.
appendSetFixtures(element);

// works now 
$($('#abc');

```
Also, jasmine-jquery provides a lot of jquery matchers. Check https://github.com/velesin/jasmine-jquery for more detials.


### Directive Templates
When `templateUrl` is used, module `uilibs.test.templates` must be loaded to avoid HTTP request to template.
```typescript
beforeEach(module('uilibs.directives.name', 'uilibs.test.templates'));
```

### Directive Element
Since unit test doesn't have a DOM envrionment, directive elements cannot be created normally (that is, angular loads and compiles html files). Thus manual compilation is required here.
```typescript
$(compile('html')(scope));
// call digest to fire all the watches
scope.$digest();
```

### Trigger Event Programmatically
Use `triggerHandler` method to trigger event programmatically
```typescript
element.triggerHandler('eventName');
```

### Work with `$timeout` and `$interval`
Test exits before code inside `$timeout` or `$interval` executes. To fix this, `ngMock` provides a method called `timeout.flush()` or `$interva.flush()`. Remember to call this, if you have to work with `$timeout` or `$interva`.

### Spy on Global Objects
Spied global objects need to be restored to original value, otherwise other tests are affected. Use `setupSpyOn` function defined in test/unit/ts/init.ts
```typescript
let jquerySpy: IPxSpy = setupSpyOn(window, '$');
...
jquerySpy.unSpy();
```

### Work with Promise Returned by `$q.defer().promise`
Call `scope.$apply()`, otherwise, test exits before then() is called.

### Test Private Methods
If this method is simple enough, it does not need to tested separately and can be tested just along with 'public' methods. Otherwise, cast service to type of `any` and call private method directly. Basically, we just need to explictly tell typescript compiler to supress type checking by coverting the type of service to 'any'.
```typescript
expect((<any>uilibs_Service).privateMethod()).toEqual(exptectValue);
```
