# Page Controller

**Category:** UI · **Also known as:** Per-page handler, route handler · **Related:** Front Controller, Application Controller, MVC, Service Layer, Transaction Script

## Intent
Give each page (or each route/action) its own controller object or function that handles the request for that page: read input, call the model, choose and render the view.

## Problem
- A server app has a handful of pages; you need a simple, obvious mapping "this URL → this code → this template".
- A single giant handler with `if path == ...` branches is unreadable.
- Generic dispatch frameworks feel heavy for a small admin tool.

Scenario: an internal admin site has `/orders`, `/orders/{id}`, `/orders/{id}/refund`. Each gets one handler function that parses its parameters, calls the order service, and renders its template. Nothing else knows about the other pages.

## Solution and structure
```
GET /orders        ──▶ list_orders()   ──▶ OrderService.search() ──▶ orders/list.html
GET /orders/{id}   ──▶ order_detail()  ──▶ OrderService.get()    ──▶ orders/detail.html
POST /orders/{id}/refund ──▶ refund()  ──▶ OrderService.refund() ──▶ redirect
```
- **Page controller**: one function/class per page or action; handles input parsing, invokes the model, selects the view.
- **Model / service layer**: business logic, shared across controllers.
- **View**: template or response model.

Original Fowler form: each page *is* the entry point (classic PHP/JSP/ASP files). Today the framework's router is a Front Controller that dispatches to per-route handlers — those handlers are Page Controllers. The two patterns combine in almost every modern web framework.

## When to use
- CRUD-style web apps and APIs where each page/endpoint has independent logic.
- Small-to-medium server-rendered apps (FastAPI + Jinja, Django views, Express routes, Next.js/Analog route handlers).
- When you want navigation to be obvious from the file/route layout.

## When NOT to use
- Flow logic spans pages (wizards, conditional next steps, "after step 2 go to 4 if…"): put that in an Application Controller instead of scattering redirects across page controllers.
- Many pages share cross-cutting pre-processing (auth, locale, CSRF, logging): use Front Controller middleware rather than repeating it in every handler.
- Controllers containing business logic: they become "fat controllers". Move rules into a service/domain layer.

## Example — Python
```python
from typing import Annotated
from fastapi import APIRouter, Depends, Form, Request, HTTPException
from fastapi.responses import HTMLResponse, RedirectResponse
from fastapi.templating import Jinja2Templates
from app.orders import OrderService, get_order_service, OrderNotRefundable

router = APIRouter(prefix="/orders")
templates = Jinja2Templates(directory="templates")
Service = Annotated[OrderService, Depends(get_order_service)]

@router.get("", response_class=HTMLResponse)
async def list_orders(request: Request, svc: Service, q: str = "", page: int = 1):
    result = await svc.search(q=q, page=page)
    return templates.TemplateResponse(request, "orders/list.html", {"result": result, "q": q})

@router.get("/{order_id}", response_class=HTMLResponse)
async def order_detail(request: Request, order_id: int, svc: Service):
    order = await svc.get(order_id)
    if order is None:
        raise HTTPException(404)
    return templates.TemplateResponse(request, "orders/detail.html", {"order": order})

@router.post("/{order_id}/refund")
async def refund(order_id: int, svc: Service, reason: Annotated[str, Form()]):
    try:
        await svc.refund(order_id, reason=reason)
    except OrderNotRefundable as e:
        raise HTTPException(409, str(e))
    return RedirectResponse(f"/orders/{order_id}", status_code=303)   # Post/Redirect/Get
```
Each handler is thin: parse, delegate, render. `Depends` injects the service, so tests can override it.

## Example — TypeScript
On the client, an Angular routed component plus its resolver acts as the page controller for that route:
```ts
import { Component, inject, input, computed } from '@angular/core';
import { CurrencyPipe } from '@angular/common';
import { Routes, ResolveFn, Router } from '@angular/router';
import { OrdersApi, Order } from './orders.api';

export const orderResolver: ResolveFn<Order> = (route) =>
  inject(OrdersApi).get(Number(route.paramMap.get('id')));

@Component({
  selector: 'app-order-detail-page',
  imports: [CurrencyPipe],
  template: `
    <h1>Order #{{ order().id }}</h1>
    <p>Total: {{ order().total | currency: 'EUR' }}</p>
    @if (canRefund()) { <button (click)="refund()">Refund</button> }`,
})
export class OrderDetailPage {
  readonly order = input.required<Order>();     // bound from resolver via withComponentInputBinding()
  private readonly api = inject(OrdersApi);
  private readonly router = inject(Router);
  readonly canRefund = computed(() => this.order().status === 'PAID');

  refund(): void {
    this.api.refund(this.order().id).subscribe(() => this.router.navigate(['/orders']));
  }
}

export const routes: Routes = [
  { path: 'orders/:id', component: OrderDetailPage, resolve: { order: orderResolver } },
];
```
Needs `provideRouter(routes, withComponentInputBinding())` so the resolved `order` is bound to the `order` input.

## In the wild
- FastAPI/Flask route functions, Django function/class-based views, Express/Fastify route handlers, NestJS controller methods, Rails/Laravel controllers (one action per page).
- Next.js / Analog / SvelteKit file-based routes: one file per page with its loader.
- Classic PHP files, ASP.NET Web Forms code-behind, JSP + servlet per page.

## Trade-offs
| Pros | Cons |
|---|---|
| Very simple, obvious mapping from URL to code | Cross-cutting logic repeated unless the router adds middleware |
| Pages evolve independently | Flow across pages scattered among controllers |
| Easy to test one handler with dependency overrides | Fat controllers if business logic creeps in |
| Matches file-based routing | Duplication of input parsing/validation between pages |

## Pitfalls
- **Fat controller:** SQL queries and pricing rules inside handlers. Extract to a Service Layer; handlers should be ~10 lines.
- **Missing Post/Redirect/Get:** returning HTML from a POST leads to double submissions on refresh. Redirect with 303.
- **Duplicated auth checks** in every handler: move to router-level dependencies (`APIRouter(dependencies=[...])`) or Angular route guards.
- **Angular component doing its own loading and navigation logic** for multi-step flows: move flow to an Application Controller service.

## Refactoring path
- **Introduce it:** split a branching mega-handler into one function per route; move shared logic into a service.
- **Remove it / simplify:** when many page controllers share flow logic, extract an Application Controller; when they share pre-processing, move it into Front Controller middleware.

## Related patterns
- [Front Controller](../ui/front-controller.md) — the single entry/router that dispatches to page controllers.
- [Application Controller](../ui/application-controller.md) — centralizes flow between pages.
- [MVC](../architectural/mvc.md) — page controllers are the C in web MVC.
- [Service Layer](../enterprise/service-layer.md) — where logic goes so controllers stay thin.
