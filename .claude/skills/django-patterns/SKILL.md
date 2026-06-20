---
name: django-patterns
description: Structuring Django projects — service/selector layers, ORM tuning, DRF serializers, signals, and middleware
---

# Django patterns

## Project layout

Lay a project out so apps, shared utilities, and configuration each have their own clear home.

```
project/
  config/
    settings/
      base.py
      local.py
      production.py
    urls.py
    wsgi.py
  apps/
    users/
      models.py
      serializers.py
      views.py
      services.py
      selectors.py
      urls.py
      tests/
    orders/
      ...
  common/
    models.py
    permissions.py
    pagination.py
```

Put write logic in `services.py` and read logic in `selectors.py`, and let views stay thin.

## Tuning the ORM

```python
# select_related for ForeignKey / OneToOne (SQL JOIN)
orders = Order.objects.select_related("customer", "customer__profile").all()

# prefetch_related for ManyToMany / reverse FK (separate query)
authors = Author.objects.prefetch_related(
    Prefetch("books", queryset=Book.objects.filter(published=True))
).all()

# Defer fields you don't need
posts = Post.objects.defer("body", "metadata").filter(status="published")

# Use .only() when you need just a few columns
emails = User.objects.only("id", "email").filter(is_active=True)

# Bulk operations
Product.objects.bulk_create(products, batch_size=1000)
Product.objects.bulk_update(products, ["price", "stock"], batch_size=1000)
```

Keep an eye on what's actually firing with `django-debug-toolbar`, or assert on `connection.queries` in tests.

## DRF serializers

```python
class OrderSerializer(serializers.ModelSerializer):
    customer_name = serializers.CharField(source="customer.full_name", read_only=True)
    items = OrderItemSerializer(many=True, read_only=True)
    total = serializers.SerializerMethodField()

    class Meta:
        model = Order
        fields = ["id", "customer_name", "items", "total", "created_at"]
        read_only_fields = ["id", "created_at"]

    def get_total(self, obj):
        return sum(item.price * item.quantity for item in obj.items.all())

    def validate(self, data):
        if data.get("start_date") and data.get("end_date"):
            if data["start_date"] >= data["end_date"]:
                raise serializers.ValidationError("end_date must be after start_date")
        return data
```

## Signals

```python
from django.db.models.signals import post_save
from django.dispatch import receiver

@receiver(post_save, sender=Order)
def order_created_handler(sender, instance, created, **kwargs):
    if created:
        send_order_confirmation.delay(instance.id)
        update_inventory.delay(instance.id)
```

Save signals for cross-app side effects; when the logic lives in the same app, just call the service directly.

## Custom middleware

```python
import time
import logging

logger = logging.getLogger(__name__)

class RequestTimingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        start = time.monotonic()
        response = self.get_response(request)
        duration = time.monotonic() - start
        logger.info(f"{request.method} {request.path} {response.status_code} {duration:.3f}s")
        return response
```

## What to avoid

- Stuffing business logic into views or serializers instead of a service layer
- `Model.objects.all()` with no pagination on a list endpoint
- N+1 queries from a missing `select_related` / `prefetch_related`
- Overusing signals for same-app logic, which makes the flow hard to follow
- Secrets sitting in `settings.py` rather than environment variables
- Raw SQL that isn't parameterized

## Before you ship

- [ ] Business logic lives in services/selectors, not views
- [ ] List queries use `select_related` or `prefetch_related` where they need to
- [ ] Serializers validate input through custom `validate` methods
- [ ] Settings are split into base/local/production modules
- [ ] Migrations get reviewed before merge
- [ ] Batch inserts and updates use bulk operations
- [ ] Custom middleware follows the WSGI callable pattern
- [ ] Tests cover model constraints, serializer validation, and view permissions
