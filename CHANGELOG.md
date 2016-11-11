### 0.5.0 - 2016-11-10

* enhancements
  * Convert railtie to Rails engine
  * Extend install generator to add offline page and starter icons for generated web app manifest
  * Install serviceworker and manifest as .erb files

### 0.4.0 - 2016-11-09

* enhancements
  * Add install generator to insert code snippets and add default serviceworker
    and companion scripts, web app manifest, and Rails initializer

### 0.3.1 - 2016-05-03

* enhancements
  * add CHANGELOG
* bug fixes
  * ensure railtie adds middleware to stack after config initializers are loaded

### 0.3.0 - 2016-05-02

* enhancements
  * new route matching behavior; similar to Rails-routing style and Rack::Router
  * add to Travis-CI builds

### 0.2.0 - 2016-04-25

* enhancements
  * routes are now configurable instead of a single-hardcoded path
  * support custom headers
  * extract Route and Router classes

### 0.1.0 - 2016-04-23

* initial release
