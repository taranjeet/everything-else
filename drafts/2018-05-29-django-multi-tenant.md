
## Steps to achieve multi-tenancy in Django

Lets us consider an application named djangolibrary, hosted at djangolibrary.com. Any company(let's call it customer) can signup and host its library. Consider that a customer named `abc` wants to host its library. It will be hosted at `abc.djangolibrary.com`. Below are the steps to achieve multi-tenancy in Django for this application.

* First, let's add a model which contain the details about the customer. `short_id` will be the unique key that can be used to identify any customer.

```
# tenants/models.py
# consider that this app is named as tenants
# so this model will be available as tenants.models.Customer

class Customer(models.Model):

    short_id = models.CharField(max_length=255, unique=True, help_text='This is the unique id of the customer')
    full_name = models.CharField(max_length=512)

    class Meta:
        db_table = 'customer'

    def __str__(self):
        return '{}'.format(self.full_name)
```

* Now, let's add a middleware, which will find the subdomain form the request and then attach relevant customer instance to the request.

```
# tenants/middlewares.py
from tenants.models import Customer


class TenantMiddleware(object):

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):

        domain = request.META.get('HTTP_HOST') or request.META.get('SERVER_NAME')
        pieces = domain.split('.')
        subdomain = ".".join(pieces[:-2])

        try:
            customer = Customer.objects.get(short_id=subdomain)
        except Exception as e:     # noqa
            customer = None

        setattr(request, 'customer', customer)
        response = self.get_response(request)
        return response

```

This middleware can be enabled in settings by including in `MIDDLEWARE_CLASSES` like

```
MIDDLEWARE_CLASSES = [
    ...,
    'tenants.middlewares.TenantMiddleware',
    ...,
]
```

This middleware attaches the value of customer instance on request. It can be accessed by `request.customer`

* Now, we need to write a mixin which will attach the add `customer` as a foreign key in each model of the application. Each table will contain a column named `customer_id` which will be a reference to the `Customer` instance.

```
# tenants/models.py

from django.db import models

class CustomerMixin(models.Model):

    customer = models.ForeignKey(Customer, blank=True,
                                 null=True, db_index=True,
                                 on_delete=models.CASCADE)

    class Meta:
        abstract = True
```

Lets say, we are creating a `Team` model, which is used to denote team for a company. This model will be inherited(or used as mixin in case you are using some class other than models.Model) from `CustomerMixin`, like

```
from tenants.models import CustomerMixin

# way 1
class Team(CustomerMixin):
    name = models.CharField(max_length=100, unique=True)

# OR
# way 2
class Team(AuditModelMixin, CustomerMixin):
    name = models.CharField(max_length=100, unique=True)

```

Similarly, any other model can be extended.

* Once every model is extended from `CustomerMixin`, we need to add a generic support to fetch and save the value of customer in `customer` column. For this we need to write a custom queryset and manager.

```
from django.db import models

class CustomerQuerySet(models.query.QuerySet):

    def own_data(self, customer):
        return self.filter(customer=customer)


class CustomerManager(models.Manager):

    use_for_related_fields = True

    def get_queryset(self):
        return CustomerQuerySet(self.model, using=self._db)

    def own_data(self, customer):
        return self.get_queryset().own_data(customer)
```

Now this manager should be added to `CustomerMixin` so that any model, which is extended from it, can use this. So `CustomerMixin` will look like

```
class CustomerMixin(models.Model):

    customer = models.ForeignKey(Customer, blank=True,
                                 null=True, db_index=True,
                                 on_delete=models.CASCADE)

    objects = models.Manager()
    customers = CustomerManager()

    class Meta:
        abstract = True
```

Now to get objects of `Team` model, we can use something like `Team.customers.own_data(request.customer)`

* Till now, we have added support to automatically add where clause on subdomain, but if any new instance of model is saved, it will be saved without the subdomain reference. For saving reference to subdomain in a generic way, we will use `pre_save` signal

```
@receiver(pre_save)
def save_company_name(sender, instance, **kwargs):
    # as this runs for every model
    if not issubclass(sender, CompanyModel):
        return
    # how get_request provides request, this is explained below
    request = get_request()
    try:
        company = Company.objects.get(short_code=request.subdomain)
        instance.company = company
    except Exception as e:
        company = None
```

This signal runs before any model is saved. It checks if the sender is a subclass of `CompanyModel` and if it is, then it saves subdomain reference to the concerned model.

* This completes most of the part of multi-tenancy when models are concerned. One important thing that is required frequently will be to access current customer in the template. For this, a context processor can be added.

```
# tenants/context_processor.py

def get_customer_name(request):

    try:
        customer_name = request.customer.full_name
    except Exception as e:
        customer_name = None

    return {
        'customer_name': customer_name
    }
```

Use this context processor in the settings like

```
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [join(APP_DIR, 'templates'), ],
        'OPTIONS': {
            'context_processors': [
                ...
                'tenants.context_processor.get_customer_name',
                ...
            ],
        },
    },
```

Now to access customer value in the templates, we can use `{{customer_name}}`.

## Notes

* Earlier we were using global request package with the help of package like [django-contrib-requestprovider](https://pypi.org/project/django-contrib-requestprovider/), but this is not the case now. Now we are explicitly passing `request.customer` from the views to the queryset. This is considered as a best practice where the models should not be aware of the current request.


Todos:

* update info about how to save model with customer info
