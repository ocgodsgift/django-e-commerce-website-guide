# Simple Django E-Commerce Website Tutorial

This tutorial will guide you through building a basic e-commerce site with Django. Perfect for students to learn:

- Product catalog
- Shopping cart functionality
- Basic checkout process
- Admin product management


1. Project Setup

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # Linux/Mac
venv\Scripts\activate   # Windows

# Install Django and Pillow
pip install django pillow

# Create Django project and app
django-admin startproject ecommerce
cd ecommerce
python manage.py startapp shop

# Migrations and superuser
python manage.py migrate
python manage.py createsuperuser
```

2. Update Settings (`ecommerce/settings.py`)
This file contains the configuration for your entire Django project.

```python
INSTALLED_APPS = [
    ...
    'shop',
]

TEMPLATES =[
      DIRS = [os.path.join(BASE_DIR, 'templates')],
]

# Add at the bottom
import os
MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')
```

3. Create Models (`shop/models.py`)
Models define your database schema. Each model maps to a database table.

```python
from django.db import models

class Category(models.Model):
    name = models.CharField(max_length=100)
    
    def __str__(self):
        return self.name

class Product(models.Model):
    category = models.ForeignKey(Category, on_delete=models.CASCADE)
    name = models.CharField(max_length=200)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    description = models.TextField()
    image = models.ImageField(upload_to='products/')
    
    def __str__(self):
        return self.name

class Order(models.Model):
    product = models.ForeignKey(Product, on_delete=models.CASCADE)
    quantity = models.IntegerField(default=1)
    completed = models.BooleanField(default=False)
    
    def total_price(self):
        return self.product.price * self.quantity
```

Run migrations:

```bash
python manage.py makemigrations
python manage.py migrate
```

4. Admin Setup (`shop/admin.py`)
It registers your models so you can manage them in the Django Admin panel.

```python
from django.contrib import admin
from .models import Category, Product, Order

admin.site.register(Category)
admin.site.register(Product)
admin.site.register(Order)
```


5. Create Views (`shop/views.py`)
Views control what the user sees. They fetch data from models and pass it to templates.

```python
from django.shortcuts import render, redirect
from .models import Product, Order

def product_list(request):
    products = Product.objects.all()
    return render(request, 'shop/product_list.html', {'products': products})

def add_to_cart(request, product_id):
    product = Product.objects.get(id=product_id)
    order, created = Order.objects.get_or_create(
        product=product,
        completed=False
    )
    if not created:
        order.quantity += 1
        order.save()
    return redirect('product_list')

def cart(request):
    orders = Order.objects.filter(completed=False)
    total = sum(order.total_price() for order in orders)
    return render(request, 'shop/cart.html', {
        'orders': orders,
        'total': total
    })

def checkout(request):
    Order.objects.filter(completed=False).update(completed=True)
    return render(request, 'shop/checkout.html')
```

6. Set Up URLs
They define routes in your app — mapping URLs to views.

### Project URLs (`ecommerce/urls.py`)

```python
from django.contrib import admin
from django.urls import path, include
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('shop.urls')),
] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

### App URLs (`shop/urls.py`)

```python
from django.urls import path
from . import views

urlpatterns = [
    path('', views.product_list, name='product_list'),
    path('add/<int:product_id>/', views.add_to_cart, name='add_to_cart'),
    path('cart/', views.cart, name='cart'),
    path('checkout/', views.checkout, name='checkout'),
]
```

7. Create Templates
Templates are HTML pages rendered by views using Django’s templating engine.

base.html: Common layout for all pages.
product_list.html: Shows all products with "Add to Cart" buttons.
cart.html: Shows your cart and total.
checkout.html: Confirmation after "purchase".

### Base Template (`templates/shop/base.html`)

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Django E-Commerce{% endblock %}</title>
    
    <!-- Bootstrap 5 CSS -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    
    <!-- Font Awesome -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
    
    <!-- Custom CSS -->
    <style>
        :root {
            --primary-color: #4e73df;
            --secondary-color: #f8f9fc;
            --accent-color: #ff6b6b;
        }
        
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background-color: var(--secondary-color);
            padding-top: 56px;
        }
        
        .navbar {
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
        }
        
        .product-card {
            transition: all 0.3s ease;
            border-radius: 10px;
            overflow: hidden;
            border: none;
            box-shadow: 0 2px 15px rgba(0, 0, 0, 0.1);
        }
        
        .product-card:hover {
            transform: translateY(-5px);
            box-shadow: 0 5px 20px rgba(0, 0, 0, 0.15);
        }
        
        .product-img-container {
            height: 200px;
            overflow: hidden;
            display: flex;
            align-items: center;
            justify-content: center;
            background-color: #f8f9fa;
        }
        
        .product-img {
            max-height: 100%;
            width: auto;
            object-fit: contain;
            transition: transform 0.3s ease;
        }
        
        .product-card:hover .product-img {
            transform: scale(1.05);
        }
        
        .cart-icon {
            position: relative;
        }
        
        .cart-badge {
            position: absolute;
            top: -5px;
            right: -5px;
        }
        
        footer {
            background-color: #343a40;
            color: white;
        }
    </style>
    
    {% block extra_css %}{% endblock %}
</head>
<body>
    <!-- Navigation Bar -->
    <nav class="navbar navbar-expand-lg navbar-dark bg-primary fixed-top">
        <div class="container">
            <a class="navbar-brand" href="{% url 'product_list' %}">
                <i class="fas fa-store me-2"></i>Django Shop
            </a>
            
            <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#navbarCollapse">
                <span class="navbar-toggler-icon"></span>
            </button>
            
            <div class="collapse navbar-collapse" id="navbarCollapse">
                <ul class="navbar-nav me-auto">
                    <li class="nav-item">
                        <a class="nav-link" href="{% url 'product_list' %}">Products</a>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link" href="#">Categories</a>
                    </li>
                </ul>
                
                <div class="d-flex">
                    <a href="{% url 'cart' %}" class="btn btn-outline-light cart-icon">
                        <i class="fas fa-shopping-cart fa-lg"></i>
                        <span class="badge bg-danger cart-badge">
                            {% with total_items=request.session.cart_items|default:0 %}
                                {{ total_items }}
                            {% endwith %}
                        </span>
                    </a>
                </div>
            </div>
        </div>
    </nav>

    <!-- Main Content -->
    <main class="container my-5">
        {% if messages %}
            <div class="row">
                <div class="col-12">
                    {% for message in messages %}
                        <div class="alert alert-{{ message.tags }} alert-dismissible fade show">
                            {{ message }}
                            <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
                        </div>
                    {% endfor %}
                </div>
            </div>
        {% endif %}
        
        {% block content %}
        <!-- Content will be injected here from child templates -->
        {% endblock %}
    </main>

    <!-- Footer -->
    <footer class="py-4 mt-5">
        <div class="container">
            <div class="row">
                <div class="col-md-4">
                    <h5>About Us</h5>
                    <p>Your one-stop shop for all your needs. Quality products at affordable prices.</p>
                </div>
                <div class="col-md-4">
                    <h5>Quick Links</h5>
                    <ul class="list-unstyled">
                        <li><a href="#" class="text-white">Home</a></li>
                        <li><a href="#" class="text-white">Products</a></li>
                        <li><a href="#" class="text-white">Contact</a></li>
                    </ul>
                </div>
                <div class="col-md-4">
                    <h5>Connect With Us</h5>
                    <a href="#" class="text-white me-2"><i class="fab fa-facebook-f fa-lg"></i></a>
                    <a href="#" class="text-white me-2"><i class="fab fa-twitter fa-lg"></i></a>
                    <a href="#" class="text-white me-2"><i class="fab fa-instagram fa-lg"></i></a>
                </div>
            </div>
            <hr class="my-4 bg-light">
            <div class="text-center">
                <p class="mb-0">&copy; 2023 Django Shop. All rights reserved.</p>
            </div>
        </div>
    </footer>

    <!-- Bootstrap 5 JS Bundle with Popper -->
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
    
    {% block extra_js %}{% endblock %}
</body>
</html>
```

### Product List (`templates/shop/product_list.html`)

```html
{% extends 'shop/base.html' %}

{% block title %}Our Products{% endblock %}

{% block content %}
<div class="row">
    <div class="col-12 mb-4">
        <h1 class="display-4">Our Products</h1>
        <hr>
    </div>
    
    {% for product in products %}
    <div class="col-lg-4 col-md-6 mb-4">
        <div class="card product-card h-100">
            <div class="product-img-container p-3">
                <img src="{{ product.image.url }}" 
                     alt="{{ product.name }}" 
                     class="product-img img-fluid">
            </div>
            <div class="card-body">
                <h5 class="card-title">{{ product.name }}</h5>
                <p class="text-muted">{{ product.category.name }}</p>
                <h5 class="text-primary">${{ product.price }}</h5>
                <p class="card-text">{{ product.description|truncatewords:20 }}</p>
            </div>
            <div class="card-footer bg-white border-0">
                <a href="{% url 'add_to_cart' product.id %}" 
                   class="btn btn-primary w-100">
                    <i class="fas fa-cart-plus me-2"></i>Add to Cart
                </a>
            </div>
        </div>
    </div>
    {% endfor %}
</div>
{% endblock %}
```

### Cart (`templates/shop/cart.html`)

```html
{% extends 'shop/base.html' %}

{% block content %}
<h1>Your Cart</h1>
<table class="table">
    <thead>
        <tr>
            <th>Product</th>
            <th>Price</th>
            <th>Quantity</th>
            <th>Total</th>
        </tr>
    </thead>
    <tbody>
        {% for order in orders %}
        <tr>
            <td>{{ order.product.name }}</td>
            <td>${{ order.product.price }}</td>
            <td>{{ order.quantity }}</td>
            <td>${{ order.total_price }}</td>
        </tr>
        {% endfor %}
    </tbody>
</table>
<h3>Total: ${{ total }}</h3>
<a href="{% url 'checkout' %}" class="btn btn-success">Checkout</a>
{% endblock %}
```

### Checkout (`templates/shop/checkout.html`)

```html
{% extends 'shop/base.html' %}

{% block content %}
<div class="alert alert-success">
    <h4>Thank you for your order!</h4>
    <p>Your items will be shipped soon.</p>
</div>
<a href="{% url 'product_list' %}" class="btn btn-primary">Continue Shopping</a>
{% endblock %}
```

## 8. Run the Application

```bash
python manage.py runserver
```
