+++
Categories = ["Django", "Python", "Forms", "Inline", "CBV"]
Tags = ["django", "python", "inline", "forms", 'dynamic', 'cbv']
author = "Felipe Frizzo"
date = "2016-06-11T23:43:54-03:00"
title = "Formulários dinâmicos com inlineformset_factory em uma aplicação Django utilizando Class-Based View"

+++

Neste post iremos abordar de forma simples como utilizar o `inlineformset_factory` no Django, utilizando Class-Based View.

## Modelo e Formulário

De maneira mais basica faremos uma lista de pedidos, onde poderemos adicionar um ou mais produtos por pedido. Teriamos um `modelo` e um `formulário` como o exemplo abaixo.

```python
# models.py
from django.db import models


class Order(models.Model):
    client = models.CharField()
    date = models.DateField(auto_now_add=True)


class ItemOrder(models.Model):
    order = models.ForeignKey('Order')
    product = models.CharField()
    quantity = models.PositiveIntegerField()
    price = models.DecimalField(max_digits=20, decimal_places=2)
```

Diferente do exemplo anterior, o inlineformset_factory deve ser declarado dentro do formulário.

```python
# forms.py
from django import forms
from cadastro.models import Order, ItemOrder


class OrderForms(forms.ModelForm):
    class Meta:
        model = Order
        exclude = ['date']


class ItemOrderForms(forms.ModelForm):
    class Meta:
        model = ItemOrder
        exclude = ['order']

ItemOrderFormSet = inlineformset_factory(Order, ItemOrder, form=ItemOrderForms)
```

Agora iremos criar a nossa `view` para podermos exibir e renderizar o formulário para que conseguimos criar um pedido com um ou vário items, mas dessa vez utilizaremos Class-Based View.

```python
# views.py
from django.views.generic import CreateView
from django.shortcuts import redirect

from cadastro.forms import OrderForms, ItemOrderFormSet

class OrderView(CreateView):
    template_name = 'order_view.html'
    form_class = OrderForms

    def get_context_data(self, **kwargs):
        context = super(OrderView, self).get_context_data(**kwargs)
        if self.request.POST:
            context['forms'] = OrderForms(self.request.POST)
            context['formset'] = ItemOrderFormSet(self.request.POST)
        else:
            context['forms'] = OrderForms()
            context['formset'] = ItemOrderFormSet()
        return context

    def form_valid(self, form):
        context = self.get_context_data()
        forms = context['forms']
        formset = context['formset']
        if forms.is_valid() and formset.is_valid():
            self.object = form.save()
            forms.instance = self.object
            formset.instance = self.object
            forms.save()
            formset.save()
            return redirect('pedido')
        else:
            return self.render_to_response(self.get_context_data(form=form))
```

## Template

O template continua o mesmo do exemplo anterior.

```html
{% extends 'base.html' %}
{% load crispy_forms_tags %}
{% block head_title %}{{ template_title }}{% endblock %}

{% block content %}
    <script>
        $(document).ready(function(){
            $("#add-item").click(function(ev) {
                ev.preventDefault();
                var count = $('#order').children().length;
                var tmplMarkup = $("#item-order").html();
                var compiledTmpl = tmplMarkup.replace(/__prefix__/g, count);
                $("div#order").append(compiledTmpl);

                // update form count
                $('#id_product-TOTAL_FORMS').attr('value', count + 1);

                // some animate to scroll to view our new form
                $('html, body').animate({
                    scrollTop: $("#add-item").position().top-200
                }, 800);
            });
        });
    </script>

    <div class="row">
        <div class="col-md-6 col-md-offset-3">
            <h1 class="page-header text-center lead">Cadastro Pedido</h1>
        </div>
    </div>
    <div class="row">
        <div class="col-md-8 col-md-offset-2">
            <form action="" method="POST">
                {% csrf_token %}
                {{ forms|crispy }}
                {{ formset.management_form }}

                <legend class="lead">PRODUTOS</legend>

                <div id="order" class="form-inline form-group">
                    {% for item_order_form in formset %}
                        <div id="item-{{ forloop.counter0 }}">
                            {{ item_order_form|crispy }}
                        </div>
                    {% endfor %}
                </div>

                <a class="btn btn-info" id="add-item"><i class="fa fa-plus"></i> Add Item</a>

                <div class="form-inline buttons">
                    <a href="{% url 'order' %}" class="btn btn-warning pull-right"><i class="fa fa-times"></i> Cancelar</a>
                    <button class="btn btn-primary pull-right" value="Save"><i class="fa fa-floppy-o"></i> Salvar</button>
                </div>
            </form>
    </div>    

    <script type="text/html" id="item-order">
        <div id="item-__prefix__" style="margin-top: 10px">
                {{ formset.empty_form|crispy }}
        </div>
    </script>
{% endblock %}
```

No nosso template precisamos de uma função em `JavaScript` para adicionar novos formulários de items a cada vez que for clicado no botão `Add Item`. Essa função irá primeiro descobrir quantos formulários foram renderizados e em seguida vai adicionar um novo item ao formulário do `Pedido`. E por ultimo irá atualizar o total de formulários de items que foram incluídos.

Documentação
<https://docs.djangoproject.com/en/1.9/ref/forms/models/#inlineformset-factory>
