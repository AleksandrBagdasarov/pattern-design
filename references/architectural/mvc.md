# Model-View-Controller (MVC)

**Category:** Architectural · **Also known as:** Model 2 (web MVC), MTV (Django's naming) · **Related:** MVVM, MVP, Front Controller, Page Controller, Layered, Observer

## Intent
Separate an interactive application into the **Model** (data and business rules), the **View** (presentation) and the **Controller** (input handling that updates the model and selects a view), so each can change independently.

## Problem
- Templates contain SQL or business rules; request handlers build HTML strings.
- The same data must be shown in several representations (HTML page, CSV export, JSON).
- Designers cannot change markup without touching logic.
- Scenario: a server-rendered admin portal for orders. One `orders.php`/`views.py` file reads `request.form`, queries the DB, applies discounts and prints HTML. Adding a CSV export means copying half of it.

## Solution and structure
```
   user input ──► Controller ──updates──► Model
                     │                      │
                  selects                (classic MVC: notifies)
                     ▼                      ▼
                    View  ◄────reads─────  Model
```
- **Model:** domain objects plus the services/repositories behind them. Knows nothing about HTTP or HTML.
- **View:** renders model data; minimal logic (loops, conditionals, formatting via helpers).
- **Controller:** parses input, calls the model, chooses the view and passes it data.
- **Classic (Smalltalk) MVC:** views observe the model and refresh on change. **Web MVC (Model 2):** request/response; the controller pushes a view model to a template, no observation.

## When to use
- Server-rendered web apps (Django, Rails, Spring MVC, Laravel, ASP.NET MVC, Express + templates).
- Any UI where input handling and presentation should be testable/changeable separately.
- As the internal structure of the presentation layer in a Layered app.

## When NOT to use
- Rich client SPAs with two-way data binding: MVVM matches the framework better (Angular, Vue).
- JSON-only APIs: "views" are just serializers; think "router → handler → service", do not invent a View layer.
- Tiny tools: one handler function rendering one template is fine.
- Smell: "fat controllers" holding business rules, or "Massive View Controller" (iOS) — move rules to the model/service layer.

## Example — Python
```python
# Django-style MTV: model (models.py), template (view), view function (controller)
from decimal import Decimal
from django.db import models
from django.shortcuts import get_object_or_404, render, redirect
from django.http import HttpRequest, HttpResponse

class Order(models.Model):                                  # Model
    customer = models.CharField(max_length=100)
    total = models.DecimalField(max_digits=10, decimal_places=2)
    status = models.CharField(max_length=20, default="new")

    def apply_discount(self, pct: Decimal) -> None:
        if not (0 < pct <= 50):
            raise ValueError("discount must be 0-50%")
        self.total = (self.total * (100 - pct) / 100).quantize(Decimal("0.01"))

def order_detail(request: HttpRequest, pk: int) -> HttpResponse:   # Controller
    order = get_object_or_404(Order, pk=pk)
    if request.method == "POST":
        try:
            order.apply_discount(Decimal(request.POST["pct"]))
            order.save()
            return redirect("order-detail", pk=pk)
        except (ValueError, KeyError) as exc:
            return render(request, "orders/detail.html", {"order": order, "error": str(exc)}, status=400)
    return render(request, "orders/detail.html", {"order": order})  # selects View

# templates/orders/detail.html (View)
# <h1>Order {{ order.id }} – {{ order.customer }}</h1>
# <p>Total: {{ order.total }}</p>{% if error %}<p class="err">{{ error }}</p>{% endif %}
```
Django calls the controller a "view" and the view a "template"; the roles are the same.

## Example — TypeScript
```ts
// Express + a template engine: router as controller, service as model, EJS/Pug as view
import express, { Request, Response } from 'express';

interface Order { id: number; customer: string; total: number }
class OrderModel {                                           // Model
  private orders = new Map<number, Order>([[1, { id: 1, customer: 'Acme', total: 120 }]]);
  find(id: number): Order | undefined { return this.orders.get(id); }
  applyDiscount(id: number, pct: number): Order {
    const o = this.find(id);
    if (!o) throw new Error('not found');
    if (pct <= 0 || pct > 50) throw new RangeError('discount must be 0-50%');
    o.total = Math.round(o.total * (100 - pct)) / 100;
    return o;
  }
}

const model = new OrderModel();
const app = express();
app.set('view engine', 'ejs');
app.use(express.urlencoded({ extended: false }));

app.get('/orders/:id', (req: Request, res: Response) => {     // Controller
  const order = model.find(Number(req.params.id));
  if (!order) return res.status(404).render('404');
  res.render('orders/detail', { order });                    // View
});
app.post('/orders/:id/discount', (req: Request, res: Response) => {
  try { model.applyDiscount(Number(req.params.id), Number(req.body.pct)); res.redirect(`/orders/${req.params.id}`); }
  catch (e) { res.status(400).render('orders/detail', { order: model.find(Number(req.params.id)), error: String(e) }); }
});
```
Angular is **not** MVC in this sense: components bind to their own state (MVVM).

## In the wild
- Django (MTV), Rails, Laravel, Spring MVC, ASP.NET Core MVC, NestJS controllers with `@Render`.
- FastAPI + Jinja2Templates is web MVC when you render HTML.
- Smalltalk-80 (origin), Cocoa/UIKit (Apple's variant, with controllers mediating model and view).

## Trade-offs
| Pros | Cons |
|---|---|
| Clear separation of presentation and logic | Controllers tend to bloat |
| Multiple views over one model | Meaning varies between frameworks, causing confusion |
| Views editable by designers | Classic MVC's observer links get tangled in big UIs |
| Controllers testable with request fakes | Not a full architecture: says nothing about persistence or domain |

## Pitfalls
- **Fat controller:** validation, rules and queries in the handler. Move rules to model methods or a service layer.
- **Logic in templates:** computed totals or permission checks in the template. Precompute in the controller or a View Helper.
- **Model = ORM table only:** "model" means the whole domain layer, not just active-record classes.
- **Passing ORM objects with lazy relations to templates:** N+1 queries during render. Prefetch (`select_related`, `prefetch_related`, `selectinload`).

## Refactoring path
- **Introduce it:** in a script-like handler, first move the HTML into a template, then move business rules into model methods or a service, leaving the handler as parse → call → render.
- **Remove it / simplify:** for a JSON API, drop the view layer and keep handler → service; for a single-page widget, a single component (MVVM) replaces controller + template.

## Related patterns
- [MVVM](../architectural/mvvm.md) — the view binds to a view model; no controller per request.
- [MVP](../architectural/mvp.md) — presenter fully drives a passive view.
- [Front Controller](../ui/front-controller.md) — the router that dispatches to controllers.
- [Page Controller](../ui/page-controller.md) — one controller per page variant.
- [View Helper](../ui/view-helper.md) — keeps formatting out of templates.
- [Layered](../architectural/layered.md) — MVC usually lives inside its presentation layer.
