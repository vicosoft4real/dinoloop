# Dino
Dino object is the main interface through which you will register controllers and middlewares.

### constructor(app: Express, baseUri: String)
Requires express instance and baseUri on dino app will be mounted.
```
const app = express();
const dino = new Dino(app, '/api');
```
### bind()
Binds dino to express, once you invoke `.bind()` it is done. You are not allowed to invoke `.bind()` twice which results `Error: dino.bind(): Already invoked.`
```
dino.bind();
```
* Invoke `bind()` only after everything is configured and registered.
### dependencyResolver<T>(injector: T, cb: (injector: T, type: any) => any)
Allows you to configure any dependency injection framework available in Typescript. We highly recommend [InversifyJs](https://github.com/inversify/InversifyJS/) which is just fantastic. We already have a [dino-inverisfy-starter](https://github.com/ParallelTask/dinoloop-inversify-starter) project adhere to SOLID principles, clone it and start developing your app :)
```
import { Container } from 'inversify';
import { AppContainer } from '/path/to/app.container'

dino.dependencyResolver<Container>
    (AppContainer, (injector, type) => {
        // Resolve objects from inversify container
        return injector.resolve(type);
    });
```
`Warning:` Please be sure when you choose DI framework, you do not have singleton instance of controller objects. If you have single instance lifescope for controller objects then you might enter into serious issues where one user request gets tampered with other users request.

#### Why does singleton controller create problems?
Dinoloop resolves objects from DI container on every request. Now If you get the same object reference on every hit to `HomeController` (*Assuming HomeController as singleton*) then you are having single object/request sharing across users which results lot of problems. *Technically It is not a webserver :)*.

`Note:` This is applicable only for controller instances.

#### Problems with [injection-js](https://github.com/mgechev/injection-js)

[injection-js](https://github.com/mgechev/injection-js) is one such DI framework which resolves everything as singleton. To have a quick fix, you can shallow clone the controller objects 
```
import { ReflectiveInjector } from 'injection-js';
import * as clone from 'lodash.clone';

dino.dependencyResolver<ReflectiveInjector>(Container,
    (injector, type) => {
        let i = injector.get(type);
        // clone it, so that you get different reference object
        // you don't need deepclone, lodash.clone works!
        if (i instanceof ApiController) {
            return clone(i);
        }
        return i;
    });
```
#### Middlewares can be singleton
You can have singleton middlewares, totally depends on your requirements.
### disableRouteNotFoundException()
Dinoloop internally uses router middleware (*created by dinoloop authors*) which handles every request to verify if the request matches a valid action-route, If the request does not match then dinloop throws **RouteNotFoundException**.

#### Why dinoloop uses router middleware?
Express fires the handlers based on the mounted path.
So the middlewares still gets fired even though the end action-route is not matched. For example:
``` 
router.get('/home', function (req, res) {
    res.send('hello, user!')
});

// This handler gets fired even for /admin/apple
// but we have configured /admin/home
app.use('/admin', (req, res, next) => {
    next();
});

app.use('/admin', router);

Request: GET /admin/apple
```
Even though end route is not matched, the middlewares are fired. To overcome this problem dinoloop introduced router-middleware. To make sure middlewares are executed only when the valid action-method is found.
This also increases response times and reduces unwanted handlers execution.

Invoking this method disables dinloop router middleware.

Highly recommended **NOT TO INVOKE** this method.

### enableUserIdentity()
Experimental feature for future use.
### registerApplicationError<T>(T)
Register controller that extends [ErrorController](https://github.com/ParallelTask/dinoloop/blob/master/wiki/controllers.md#errorcontroller).
```
dino.registerApplicationError<ApplicationErrorController>(ApplicationErrorController);
```
* Dinoloop allows only one error controller to register.

If you register multiple error controllers, the last controller gets registered with dino. All other error controllers are discarded.
### registerController<T>(T)
Register controller that extends [ApiController](https://github.com/ParallelTask/dinoloop/blob/master/wiki/controllers.md#apicontroller).
```
dino.registerController<HomeController>(HomeController);
```
### requestStart<T>(T)
Register request-start middlewares. These are the middlewares to handle request first in the chain. More on [RequestStart Middlewares](https://github.com/ParallelTask/dinoloop/blob/master/wiki/application_middlewares.md#requeststartmiddleware).
```
dino.requestStart<LogRequestStart>(LogRequestStart);
dino.requestStart<OtherRequestStart>(OtherRequestStart);
```
### requestEnd<T>(T)
Register request-end middlewares. These are the middlewares to handle request last in the chain. More on [RequestEnd Middlewares](https://github.com/ParallelTask/dinoloop/blob/master/wiki/application_middlewares.md#requestendmiddleware).
```
dino.requestEnd<LogRequestEnd>(LogRequestEnd);
dino.requestEnd<OtherRequestEnd>(OtherRequestEnd);
```
### serverError<T>(T)
Register server-error middlewares to handle uncaught exceptions/errors thrown by application. More on [ServerError Middlewares](https://github.com/ParallelTask/dinoloop/blob/master/wiki/application_middlewares.md#errormiddleware) .
```
dino.serverError<FormatExceptionr>(FormatException);
dino.serverError<MongoException>(MongoException);
```
### useRouter(cb: () => express.Router)
Register callback that returns new instance of `express.Router` on every invoke.
```
dino.useRouter(() => express.Router());
```
Make sure to attach express.Router() via `.useRouter()`, otherwise dino throws `Error: Express router is not registered with dino`.
### Important Points
* Multiple `requestStart`, `requestEnd` and `serverError` dinowares can be registered.
* Order of execution depends on the order of registration.
