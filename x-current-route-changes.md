# x-current-route Header – Changes Only

Goal: add the `x-current-route` header **only** to API calls going to the app's base URL
(`envBaseUrl` / `envMilitaryBaseUrl`). Assets and third-party calls are untouched. No request is blocked.

---

## FILE 1 – `services/route-tracking.service.ts` (NEW – skip if it already exists)

```ts
/**
 * Copyright GOSI. All Rights Reserved.
 * This software is the proprietary information of GOSI.
 * Use is subject to license terms.
 */
import { Injectable } from '@angular/core';
import { NavigationEnd, Router } from '@angular/router';
import { filter } from 'rxjs/operators';

@Injectable({
  providedIn: 'root'
})
export class RouteTrackingService {
  private currentRoute = '';

  constructor(private router: Router) {
    this.router.events
      .pipe(filter((event): event is NavigationEnd => event instanceof NavigationEnd))
      .subscribe((event: NavigationEnd) => {
        this.currentRoute = this.cleanRoute(event.urlAfterRedirects);
      });
  }

  /**
   * Returns the current screen route without query params or fragment.
   */
  getCurrentRoute(): string {
    return this.currentRoute || this.cleanRoute(this.router.url);
  }

  private cleanRoute(url: string): string {
    if (!url) {
      return '';
    }
    return url.split('?')[0].split('#')[0];
  }
}
```

---

## FILE 2 – `services/index.ts` (ADD ONE LINE – only if a barrel file exists)

```ts
export * from './route-tracking.service';
```

---

## FILE 3 – `interceptors/gosi-http.interceptor.ts` (4 CHANGES)

### Change 3a – Import (top of file)

```ts
import { RouteTrackingService } from '../services/route-tracking.service';
```

### Change 3b – Constructor: add as LAST parameter

```ts
    @Inject(RegistrationNoToken) private registrationNo: RegistrationNumber,
    private routeTrackingService: RouteTrackingService   // <-- ADD
  ) {
```

### Change 3c – `intercept()`: REMOVE from the top

Delete everything between the method signature and `let publicTenancy = '';`

```ts
  intercept(request: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    // ❌ DELETE any x-current-route / isAppApi code that was here

    let publicTenancy = '';
```

### Change 3d – `intercept()`: ADD after the SharePoint branch, BEFORE the `this.map` check

```ts
      } else if (request.url.indexOf('api/SharePoint') !== -1) {
        request = this.interceptForGPT(request);
      }

      // ✅ ADD START
      // Attach current screen route to the request header for analytics.
      // Placed after the base URL is applied, so only calls going to the
      // application's API base URL get it; assets and third-party calls are excluded.
      if (this.isAppApiRequest(request.url)) {
        const currentRoute = this.routeTrackingService.getCurrentRoute();
        if (currentRoute) {
          request = request.clone({
            setHeaders: {
              'x-current-route': currentRoute
            }
          });
        }
      }
      // ✅ ADD END

      if (!this.map.get(request.urlWithParams)) {
```

### Change 3e – New private method (bottom of the class, after `interceptForGPT`)

```ts
  /**
   * Checks whether the request is going to the application's own API base URL
   * @param url Request URL after base URL is applied
   */
  private isAppApiRequest(url: string): boolean {
    const apiBaseUrls = [this.envBaseUrl, this.envMilitaryBaseUrl].filter(
      (baseUrl): baseUrl is string => !!baseUrl && baseUrl.trim().length > 0
    );
    return apiBaseUrls.some(baseUrl => url.startsWith(baseUrl));
  }
```

---

## Before commit – checklist

- [ ] Remove every `console.log('[x-current-route]...')` line
- [ ] Network tab: API call (e.g. `/v1/lov`) → has `X-Current-Route` header
- [ ] Network tab: `en.json` / `.svg` → no header, loads normally
- [ ] Privileged user call (`gateway-e2e.gosi.ins`) → has header, returns 200
- [ ] Backend team confirms `x-current-route` is in `Access-Control-Allow-Headers`
      for `gateway-e2e.gosi.ins` and the production gateway
      (`gosiapidev.gosi.ins:9004` already allows it)
