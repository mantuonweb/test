```diff
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute, Router } from '@angular/router';
import { AdminLoginEventsService, EDAPAdminConfigurationsService, EDAPExtendSessionPopup, EDAPLocaleConvertorService, EDAPLocalizationConfigurationService, EDAPLocalizationService, EdapServerConfigManager, EdapServerURI, EDAPSessionManagementService, SeedLoginEventsService,EDAP_APP_FLOW_TYPE } from 'EdapWebComponents';
import * as WebFont from 'webfontloader';
import { ApplicationConfigurationService } from '../configuration/application.configuration';
import { ContextManagerService } from './context-manager/context-manager.service';
import { AppLocationService } from './service/app.state.service';
import { LoginStatusService } from './service/login-status.service';
+import { take } from 'rxjs/operators';
declare var $;
@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css']
})
export class AppComponent implements OnInit {
  title = 'EDAP Login';
  extentSession
  constructor(private _router: Router, private acRoute: ActivatedRoute, private appLocation: AppLocationService, private edapAdminConfig: EDAPAdminConfigurationsService, public translate: EDAPLocalizationService, public localization: EDAPLocalizationConfigurationService, private sessionManage: EDAPSessionManagementService, private edapLocaleConvertor: EDAPLocaleConvertorService, private edapServerConfigManager: EdapServerConfigManager, private loginStatus: LoginStatusService, private contextManager: ContextManagerService, private adminLoginEvents: AdminLoginEventsService, private seedLoginEvent: SeedLoginEventsService) {
    //App Service to access last location entered in the browser
    this.appLocation.setLastLocation(window.location.pathname);
    //Set WebComponent Library Parameters for Application context to access the images and setting the context
    this.edapServerConfigManager.setEdapServerURI(<EdapServerURI>{
      url: ApplicationConfigurationService.configSettings.apiServerConfig.uri,
      version: ApplicationConfigurationService.configSettings.apiServerConfig.version,
      deployUrl: this.contextManager.getContext()
    });

    this.loginStatus.getLoggedInNotifier().subscribe(() => {
      this.setSessionExpiredPopupDetails();
    });
    
+    /**
+     * Event/Subscriptions to notify the application need to load the required locale resources
+     * To Notify Locale is changed and application can load the required 
+    */
+    this.localization.getTranslationLoadRequired().subscribe((userLocale)=>{
+      this.localization.setLocaleTranslationConfiguration(userLocale, ...this.contextManager.getLocaleFiles(userLocale));
+    });
    /**
     * For TMO Users Only
     * Subscribe the event to get the User successfully logged into the application
     * As soon User Login process is completed and user is verified the event is triggered
     * with parameters @loginEvent which contains the session information for the user
     */
    this.adminLoginEvents.onLoginSuccess.subscribe((loginEvent) => {
      //Initialize the Localization Service
-      this.localization.setLocaleConfiguration({ ...loginEvent.userLocale }, ...this.contextManager.getLocaleFiles(loginEvent.userLocale)).pipe(take(1)).subscribe(() => {
+      this.localization.setLocaleConfiguration({ ...loginEvent.userLocale }, ...this.contextManager.getLocaleFiles(loginEvent.userLocale)).pipe(take(1)).subscribe(() => {
        //for application flow only to refresh the session object
        //cached the session information object
        this.loginStatus.setSessionInfo(loginEvent);
        //Triggered the logged in event to notify
        this.loginStatus.userLoggedIn();
        //Admin logged in event to notify admin is logged in
        this.appLocation.onAdminLoggedIn.emit(loginEvent);
      });
    });
    /**
     * For Seed Admin Users Only
     * Subscribe the event to get the User successfully logged into the application
     * As soon User Login process is completed and user is verified the event is triggered
     * with parameters @loginEvent which contains the session information for the user
     */
    this.seedLoginEvent.onLoginSuccess.subscribe((loginEvent) => {
      let sessionInfo = loginEvent;
+      if(this.isSeedAdminBasic() && !sessionInfo.needToLoadLocale){
+         //for application flow only to refresh the session object
+         this.loginStatus.setSessionInfo(sessionInfo);
+         //Triggered the logged in event to notify
+         this.loginStatus.userLoggedIn();
+         //Seed Admin logged in event to notify seed admin is logged in
+         this.appLocation.onSeedAdminLoggedIn.emit(loginEvent);
+         this.localization.setLocaleConfigurationMetrics(sessionInfo.userLocale);
+      }
+      else{
        //Initialize the Localization Service
-        this.localization.setLocaleConfiguration(sessionInfo.userLocale, ...this.contextManager.getLocaleFiles(loginEvent.userLocale)).subscribe(() =>
+        this.localization.setLocaleConfiguration(sessionInfo.userLocale, ...this.contextManager.getLocaleFiles(loginEvent.userLocale)).pipe(take(1)).subscribe(() => {
          //for application flow only to refresh the session object
          this.loginStatus.setSessionInfo(sessionInfo);
          //Triggered the logged in event to notify
          this.loginStatus.userLoggedIn();
          //Seed Admin logged in event to notify seed admin is logged in
          this.appLocation.onSeedAdminLoggedIn.emit(loginEvent);
+        });
      }
    });
+    this.seedLoginEvent.onLoginRefreshFailure.subscribe((error) => {
+      this.seedLoginEvent.triggerSessionTerminated(error.error);
+      sessionStorage.clear();
+    });
+    /**
+     * Seed Locale changes
+     */
+    this.seedLoginEvent.getLocaleChangeEvent().subscribe((localeEvent)=>{
+      this.localization.setLocaleTranslationConfiguration(localeEvent, ...this.contextManager.getLocaleFiles(localeEvent)).pipe(take(1)).subscribe((e) => {
+        console.log(e); //@todo
+      });
+    })
    /**
     * For Admin Users Only
     * Subscribe the event to get when the admin is logged out from the Session and 
     * Application can Take the Required Action if needed
     */
    this.adminLoginEvents.onLogout.subscribe(() => {
+    this.loginStatus.clearSessionInformation();
      /**
       * Clear the session management schedular
       */
      this.sessionManage.clear();
    });
+    /**
+     * Event to capture session refresh failure for Admin User
+     * In the case of if edap token changed on server and not matched with the browser one.
+     * This event is triggered when user presses F5 or refresh the browser
+     **/
+    this.adminLoginEvents.onLoginRefreshFailure.subscribe((error) => {
+      this.adminLoginEvents.triggerSessionTerminated(error.error);
+      sessionStorage.clear();
+    });
    /**
    * For Seed Admin Users Only
    * Subscribe the event to get when the seed admin is logged out from the Session and 
    * Application can Take the Required Action if needed
    */
    this.seedLoginEvent.onLogout.subscribe(() => {
+    this.loginStatus.clearSessionInformation();
      /**
       * Clear the session management schedular
       */
      this.sessionManage.clear();
    });
    if (WebFont && WebFont.load) {
      WebFont.load({
        google: {
          families: ['falcon', 'Roboto', 'Roboto-Light', 'Roboto-Medium', 'Roboto-Bold', 'Roboto-Black']
        }
      });
    }
  }
  /**
   * Set the following things for the Session popup
   * 1.Verbiage for the session timeout popup
   * 2.Set Parameters for the popup window e.g.(grace period,idle time)
   */
  setSessionExpiredPopupDetails() {
    // Setting keys after locale loaded successfully
    let sessionInfo = this.loginStatus.getSessionInfo();
    this.extentSession = <EDAPExtendSessionPopup>{
      title: 'sessionExtend.title',//key presents inside assets/i18n/<localize-file>.json
      yesText: 'sessionExtend.yesText',
      noText: 'sessionExtend.noText',
      expired: 'sessionExtend.expired',
      expiring: 'sessionExtend.expiring'
    };
    //Initialize the session management timeout period
    this.sessionManage.init({
      //Grace period
      gracePeriod: sessionInfo.sessionGracePeriod || 0,
      //Max idle time
      maxIdleTime: sessionInfo.sessionMaxIdleTime || 0,
      //Need to be set false only (Reserved for Future use)
      suppressTimeout: false,
      //Auth token
      accessToken: sessionInfo.accessToken,
      //Extends Session call frequency
      pingtime: sessionInfo.schedluerFrequency,
      //Session Type TMO,USER etc.
      sessionType: sessionInfo.sessionType
    });
  }
  /**
   * Convert query parameters into the JSON Object
   * @param url URL provided
   */
  queryStringToJSON(url) {
    var pairs = url.slice(1).split('&');
    var result = {};
    pairs.forEach(function (pair) {
      pair = pair.split('=');
      result[pair[0]] = decodeURIComponent(pair[1] || '');
    });
    return JSON.parse(JSON.stringify(result));
  }
  /**
   * Triggered when the session is expired
   * Either from the Terminate button is clicked or Default terminated session is initiated
   * @param res Expired information object
   * for User :if res.errorMessage is non empty and res.suppressLogout is true then the user must lands to the Error Page otherwise Lands to the login page
   * for TMO : needs to call triggerSessionExpired of AdminLoginEventsService  to notify admin flow
   * for Seed admin: Needs to call the triggerSessionExpired of SeedLoginEventsService to notify seed admin flow
   */
  onSessionExpired(res) {
+    this.loginStatus.clearSessionInformation();
    /**
     * for Admin users needs to trigger the triggerSessionExpired of AdminLoginEventsService  to notify admin flow
     */
    if (window.location.pathname.toLowerCase().includes("/edapsample/admin")) {
      this.adminLoginEvents.triggerSessionExpired(res);
    }
    /**
    * for Seed admin needs to trigger the triggerSessionExpired of SeedLoginEventsService  to notify seed admin flow
    */
    else if (window.location.pathname.toLowerCase().includes("edapsample/seedadminsso")||window.location.pathname.toLowerCase().includes("edapsample/seedadmin")) {
      this.seedLoginEvent.triggerSessionExpired(res);
    }
    /**
     * for User :if res.errorMessage is non empty and res.suppressLogout is true then the user must lands to the Error Page otherwise Lands to the login page
     */
    else {
      /**
       * If Error Message contains error and Needs to go to the error page in the case of the SSO Flow
       */
      if (res && res.suppressLogout === true && res.errorMessage) {
        this._router.navigate(['/edapsample/error'], { queryParams: { error: res.errorMessage } });
      }
      /**
       * If no error message and no suppress means go to the login page flow
       */
      else {
        this._router.navigate(['/edapsample/login']);
      }
    }
    sessionStorage.clear();
  }
  ngOnInit() {
    //Redirect to the admin login Page in the case of the Admin Flow
    if (window.location.pathname.toLowerCase().endsWith("/edapsample/admin")) {
      this._router.navigate(['/edapsample/admin/tenantadmin/adminlogin']);
    }
    /**
     * Redirect to the seed admin flow in the case of the seed admin flow
     */
    if (window.location.pathname.toLowerCase().endsWith("/edapsample/seedadmin") || window.location.pathname.toLowerCase().endsWith("/edapsample/seedadminsso")) {
      if (window.location.pathname.toLowerCase().endsWith("/edapsample/seedadminsso")) {
        let params = (window.location.search && this.queryStringToJSON(window.location.search)) || {};
        if(!$.isEmptyObject(params)){
          let queryParams = { queryParams: params };
          this._router.navigate(['/edapsample/seedadminsso/seed/seedlogin'],queryParams);
        }
        else{
          this._router.navigate(['/edapsample/seedadminsso/seed/seedlogin']);
        }
      }
      else {
        this._router.navigate(['/edapsample/seedadmin/seed/seedlogin']);
      }
    }
    /**
     * For the User flow if the user is not admin and not seed
     */
    else if (!(window.location.pathname.toLowerCase().includes("/edapsample/admin") || window.location.pathname.toLowerCase().includes("/edapsample/seedadmin"))) {
      /**
       * Persist the Query parameters of the URL
       */
      let params = (window.location.search && this.queryStringToJSON(window.location.search)) || {};
      if ($.isEmptyObject(params)) {
        /**
         * Without query params if no query params presents
         */
        this._router.navigate(['/edapsample/login']);
      }
      else {
        let queryParams = { queryParams: params }
        /**
         * With the query params if found
         */
        this._router.navigate(['/edapsample/login'], queryParams);
      }
    }
    /**
     * Subscribe the Session terminate notifier in the case of the Session terminated from the Server itself
     */
    this.sessionManage.getSessionTerminatedNotifier().subscribe((err) => {
      this.onSessionTerminated(err);
    });
  }
  /**
   * Called when Session is terminated from the server
   * @param err Error Description object
   * for User :User must lands to the Error Page
   * for TMO : Needs to call triggerSessionTerminated of AdminLoginEventsService  to notify admin flow
   * for Seed Admin: Needs to call the triggerSessionTerminated of SeedLoginEventsService to notify seed admin flow
   */
  onSessionTerminated(err) {
+     this.loginStatus.clearSessionInformation();
    /**
     * for TMO : Needs to call triggerSessionTerminated of AdminLoginEventsService  to notify admin flow
     */
    if (window.location.pathname.toLowerCase().includes("/edapsample/admin")) {
      this.adminLoginEvents.triggerSessionTerminated(err);
    }
    /**
     * for Seed Admin: Needs to call the triggerSessionTerminated of SeedLoginEventsService to notify seed admin flow
     */
    else if (window.location.pathname.toLowerCase().includes("edapsample/seedadminsso")){
      this.seedLoginEvent.triggerSessionTerminated(err);
    }
    /**
     * for User :User must lands to the Error Page
     */
    else {
      //Clear the session storage
      sessionStorage.clear();
-      this._router.navigate(['/edapsample/error'], { queryParams: { error: err.errorMessage } });
+      if(sessionInfo && sessionInfo.userLoginMode === EDAP_APP_FLOW_TYPE.BASIC){
+        this._router.navigate(['/edapsample/login']);
+      }
+      else{
+        this._router.navigate(['/edapsample/error'], { queryParams: { error: err.errorMessage } });
+      }
  }

+  isSeedAdminBasic(){
+    return window.location.pathname.toLowerCase().includes("edapsample/seedadmin") && (!window.location.pathname.toLowerCase().includes("edapsample/seedadminsso"));
+  }
}

```

## Interceptor changes for error handing and Session Event Notification
```diff
import { Injectable } from '@angular/core';
import {
    HttpInterceptor,
    HttpRequest,
    HttpResponse,
    HttpHandler,
    HttpEvent,
    HttpErrorResponse
} from '@angular/common/http';

import { Observable, throwError } from 'rxjs';
import { map, catchError } from 'rxjs/operators';
import { ApplicationConfigurationService } from 'src/configuration/application.configuration';
+import { LoginStatusService } from '../service/login-status.service';
-import { EDAPSessionManagementService } from 'EdapWebComponents'
+import { EDAPSessionManagementService,EDAP_COMMON_SESSION_ERRORS, EDAP_ADMIN_SESSION_AUTH_TOKEN, EDAP_SEED_ADMIN_SESSION_AUTH_TOKEN, AdminLoginEventsService, SeedLoginEventsService,EDAP_APP_FLOW_TYPE } from 'EdapWebComponents'
import { Router } from '@angular/router';

@Injectable()
export class HttpConfigInterceptor implements HttpInterceptor {
    apiContext;
    loggedIn = false;
    constructor(private loginStatus: LoginStatusService, public _router: Router, private sessionManage: EDAPSessionManagementService,private adminLoginEvents: AdminLoginEventsService, private seedLoginEvent: SeedLoginEventsService) {
        this.loginStatus.getLoggedInNotifier().subscribe(() => {
            this.loggedIn = true;
        });
    }
    isNotAdminAndSeed() {
        return (!window.location.href.includes("edapsample/seedadmin") && !window.location.href.includes("edapsample/admin"));
    }
    isAdmin() {
        return window.location.href.includes("edapsample/admin");
    }
    isSeedUser() {
        return window.location.href.includes("edapsample/seedadmin");
    }
    intercept(request: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {

        if (ApplicationConfigurationService && ApplicationConfigurationService.configSettings && ApplicationConfigurationService.configSettings.apiServerConfig && ApplicationConfigurationService.configSettings.apiServerConfig.uri) {
            this.apiContext = ApplicationConfigurationService.configSettings.apiServerConfig.uri.substring(ApplicationConfigurationService.configSettings.apiServerConfig.uri.lastIndexOf("/") + 1);
        }
        let isApiCall = request.url.indexOf(this.apiContext) >= 0;
        if (isApiCall && this.isNotAdminAndSeed()) {
            let token: string = sessionStorage.getItem('accessToken');
            if (token) {
                request = request.clone({ headers: request.headers.set('Authorization', token) });
            }
        }
        if (isApiCall && !this.isNotAdminAndSeed()) {
            if (window.location.href.includes("edapsample/seedadmin")) {
                let token: string = sessionStorage.getItem(EDAP_SEED_ADMIN_SESSION_AUTH_TOKEN.AUTH_TOKEN_STORAGE_KEY);
                if (token) {
                    request = request.clone({ headers: request.headers.set(EDAP_SEED_ADMIN_SESSION_AUTH_TOKEN.AUTH_TOKEN_HTTP_HEADER_KEY, token) });
                }
            }
            else {
                let token: string = sessionStorage.getItem(EDAP_ADMIN_SESSION_AUTH_TOKEN.AUTH_TOKEN_STORAGE_KEY);
                if (token) {
                    request = request.clone({ headers: request.headers.set(EDAP_ADMIN_SESSION_AUTH_TOKEN.AUTH_TOKEN_HTTP_HEADER_KEY, token) });
                }
            }
        }
        return next.handle(request).pipe(
            catchError((error: HttpErrorResponse) => {
                let notAdminAndNotSeed = this.isNotAdminAndSeed();
-                if (error && error.error && error.error.errorCode) {
+                if (error && error.error && error.error.errorCode && this.loginStatus.getLoggedInStatus()) {
                    //Error for the User needs to be handled
+                    let hasUserError = (EDAP_COMMON_SESSION_ERRORS.USER.indexOf(error.error.errorCode) >= 0);
+                    //Error for the Seed Admin needs to be handled
+                    let hasSeedAdminError = (EDAP_COMMON_SESSION_ERRORS.SEEDADMIN.indexOf(error.error.errorCode) >= 0);
+                    //Error for the Admin
+                    let hasAdminError = (EDAP_COMMON_SESSION_ERRORS.TMO.indexOf(error.error.errorCode) >= 0);
+                    //Catch exception for the User
+                     if (notAdminAndNotSeed && hasUserError) {
+                        this.loginStatus.clearSessionInformation();
+                        sessionStorage.clear();
+                        this.sessionManage.clear();
+                        let sessionInfo = this.loginStatus.getSessionInfo();
+                        if(sessionInfo && sessionInfo.userLoginMode === EDAP_APP_FLOW_TYPE.BASIC){
+                            this._router.navigate(['/edapsample/login']);
+                        }
+                        else{
                            this._router.navigate(['/edapsample/error'], { queryParams: { error: error.error.errorMessage } });
+                        }                      
+                    }
+                    //Catch exception for the Admin
                    else if (this.isAdmin() && hasAdminError) {
+                         this.loginStatus.clearSessionInformation();
                          this.adminLoginEvents.triggerSessionTerminated(error.error);
+                    }
+                    //catch Exception for the Seed Admin
+                    else if (this.isSeedUser() && hasSeedAdminError) {
+                        this.loginStatus.clearSessionInformation();
                         this.seedLoginEvent.triggerSessionTerminated(error.error);
+                    }
                }
                return throwError(error);
            })
        );
    }
}
```
