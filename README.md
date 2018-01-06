[![Stories in Ready](https://badge.waffle.io/mkowalzik/spring-typescript-services.png?label=ready&title=Ready)](https://waffle.io/mkowalzik/spring-typescript-services)
[![Build Status][travisbadge img]][travisbadge]
[![Coverity Scan Build Status][coveritybadge img]][coveritybadge]
[![codecov][codecov img]][codecov]
[![Passing Tests][sonar tests img]][sonar tests]
[![Quality Gate][sonar quality img]][sonar quality]
[![Tech Debt][sonar tech img]][sonar tech]
[![Maven Status][mavenbadge img]][mavenbadge]
[![Dependencies status][versioneye img]][versioneye]
[![Codacy Badge][codacy img]][codacy]
[![license][license img]][license]

# spring-typescript-services
Generate typescript services and type interfaces from spring annotated @RestControllers.

Get strongly typed interfaces for your spring-boot microservices in no time.

# Release 0.3.0 is here, What's new?
* Support for Generics besides Collections and Maps and improved handling of nested Type-Parameters.
* In particular, ResponseEntity<..> and Optional<..> can now be utilized as wrapping Type.
* Optional<Type> as Parameter now generates optional Functionparameters.
```typescript
public setFooPost(requiredParameter: number, optionalParamer?: string) ...
```
* @RequestBody is optional now and can be omitted or substituted by a MultipartFile,
which generates a FormData parameter.
```java
@RequestMapping(value = "/upload/{id}", method = POST, consumes = APPLICATION_JSON_VALUE, produces = APPLICATION_JSON_VALUE)
public ResponseEntity<UploadResponse> fileUpload(
    @PathVariable("id") long id,
    @RequestParam("uploadfile") MultipartFile uploadfile,
    @Context HttpServletRequest request) {...}
```
```typescript
public fileUploadPost(uploadfile: FormData, id: number) ...
```
* Controllers can now be configured through ServiceConfig to enabled debug mode or set the context root in other words a
prefix for all request URLs.

# What is it?
A Java annotation processor to generate a service and TypeScript types to access your spring @RestControllers.

# How does it work?
It generates a service.ts file for every with @TypeScriptEndpoint annotated class and includes functions 
for every enclosed public Java Method with a @RequestMapping producing JSON.
The TypeScript files are generated by populating [<#FREEMARKER>][freemarker]-template files. 
You can specify your own template files or use the bundled defaults.

# Getting started
Just specify the dependency in your maven based build.

```xml
<dependency>
    <groupId>org.leandreck.endpoints</groupId>
    <artifactId>annotations</artifactId>
    <scope>provided</scope> <!-- the annotations and processor are only needed at compile time -->
    <optional>true</optional> <!-- they need not to be transitively included in dependent artifacts -->
</dependency>
<!-- * because of spring-boot dependency handling they nevertheless get included in fat jars -->
```

# Example
The following snippet will produce an Angular Module.
```java
import org.leandreck.endpoints.annotations.TypeScriptEndpoint;
import org.springframework.web.bind.annotation.*;

import static org.springframework.http.MediaType.APPLICATION_JSON_VALUE;

@RestController
@TypeScriptEndpoint
public class Controller {

    @RequestMapping(value = "/api/get", method = RequestMethod.GET, produces = APPLICATION_JSON_VALUE)
    public ReturnType get(@RequestParam String someValue) {
        return new ReturnType("method: get");
    }
    // ...
}
```
and the produced TypeScript files from the default templates look like:

**controller.generated.ts:**
```typescript
import { HttpClient, HttpParams, HttpRequest } from '@angular/common/http';
import { Injectable } from '@angular/core';

import { Observable } from 'rxjs/Observable';
import { ErrorObservable } from 'rxjs/observable/ErrorObservable';
import 'rxjs/add/operator/catch';
import 'rxjs/add/observable/throw';
import 'rxjs/add/operator/map';

import { ReturnType } from './returntype.model.generated';
import { ServiceConfig } from './api.module';

@Injectable()
export class Controller {
    private get serviceBaseURL(): string {
        return this.serviceConfig.context + '';
    }
    private get onError(): (error: Response) => ErrorObservable {
        return this.serviceConfig.onError || this.handleError.bind(this);
    }
    constructor(private httpClient: HttpClient, private serviceConfig: ServiceConfig) { }
    /* GET */
    public getGet(someValue: string): Observable<ReturnType> {
        const url = this.serviceBaseURL + '/api/get';
        const params = this.createHttpParams({
            someValue: someValue
        });

        return this.httpClient.get<ReturnType>(url, {params: params})
            .catch((error: Response) => this.onError(error));
    }
    
    /* .. */

    private createHttpParams(values: { [index: string]: any }): HttpParams {
        let params: HttpParams = new HttpParams();

        Object.keys(values).forEach((key: string) => {
            const value: any = values[key];
            if (value != undefined) { // Check for null AND undefined
                params = params.set(key, String(value));
            }
        });

        return params;
    }

    private handleError(error: Response): ErrorObservable {
        // in a real world app, we may send the error to some remote logging infrastructure
        // instead of just logging it to the console
        this.log('error', error);

        return Observable.throw(error);
    }

    private log(level: string, message: any) {
        if (this.serviceConfig.debug) {
            console[level](message);
        }
    }

}
```
**returntype.model.generated.ts:**
```typescript
export interface ReturnType {
    method: string;
}
```

**api.module.ts:**
```typescript
import { NgModule, ModuleWithProviders, Injectable } from '@angular/core';
import { Observable } from 'rxjs/Observable';
import { Controller } from './controller.generated';

@Injectable()
export abstract class ServiceConfig {
    context?: string;
    debug?: boolean;
    onError?(): Observable<any>;
}

@NgModule({})
export class APIModule {
    static forRoot(serviceConfig: ServiceConfig = {context: ''}): ModuleWithProviders {
        return {
            ngModule: APIModule,
            providers: [
                {provide: ServiceConfig, useValue: serviceConfig},
                Controller
            ]
        };
    }
}
```
**index.ts:**
```typescript
export { BodyType } from './bodytype.model.generated';
export { ReturnType } from './returntype.model.generated';
export { Controller } from './controller.generated';
export { APIModule } from './api.module';
```

Then, in your root module, import the APIModule using forRoot()
```typescript
import { NgModule } from '@angular/core';
import { APIModule } from '...';

@NgModule({
  declarations: [...],
  imports: [APIModule.forRoot(), ...],  
  bootstrap: [...]
})
export class AppModule {}
```

[freemarker]: http://freemarker.org/

[travisbadge]:https://travis-ci.org/leandreck/spring-typescript-services
[travisbadge img]:https://travis-ci.org/leandreck/spring-typescript-services.svg?branch=master

[coveritybadge]:https://scan.coverity.com/projects/mkowalzik-spring-typescript-services
[coveritybadge img]:https://scan.coverity.com/projects/10040/badge.svg

[sonar quality]:https://sonarcloud.io/dashboard?id=org.leandreck.endpoints%3Aparent
[sonar quality img]:https://sonarcloud.io/api/badges/gate?key=org.leandreck.endpoints:parent

[sonar tech]:https://sonarqube.com/overview?id=org.leandreck.endpoints%3Aparent
[sonar tech img]:https://img.shields.io/sonar/http/sonarqube.com/org.leandreck.endpoints:parent/tech_debt.svg?label=tech%20debt

[sonar tests]:https://sonarqube.com/component_measures/metric/tests/list?id=org.leandreck.endpoints%3Aparent
[sonar tests img]:https://img.shields.io/sonar/http/sonarqube.com/org.leandreck.endpoints:parent/test_success_density.svg?label=passing%20tests%20%

[mavenbadge]:http://search.maven.org/#search%7Cga%7C1%7Cg%3A%22org.leandreck.endpoints%22%20AND%20a%3A%22annotations%22
[mavenbadge img]:https://maven-badges.herokuapp.com/maven-central/org.leandreck.endpoints/annotations/badge.svg

[versioneye]:https://www.versioneye.com/user/projects/589100fa6a0b7c004577c5a4
[versioneye img]:https://www.versioneye.com/user/projects/589100fa6a0b7c004577c5a4/badge.svg

[license]:LICENSE
[license img]:https://img.shields.io/badge/License-Apache%202-blue.svg

[codecov]:https://codecov.io/gh/leandreck/spring-typescript-services
[codecov img]:https://codecov.io/gh/leandreck/spring-typescript-services/branch/master/graph/badge.svg

[codacy]:https://www.codacy.com/app/leandreck/spring-typescript-services?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=leandreck/spring-typescript-services&amp;utm_campaign=Badge_Grade
[codacy img]:https://api.codacy.com/project/badge/Grade/fac6b09d290845d7bb1ef1f03cf3b95b