+++
Categories = ["Django", "Python", "Inline", "Forms"]
Description = ""
Tags = ["django", "python", "inline", "forms", 'dynamic']
image = "/img/home-bg.jpg"
author = "Felipe Frizzo"
date = "2016-02-14T15:44:18-02:00"
title = "Adicionar Forms dinâmicos com inlineforset_factory em uma aplicação Django"

+++

Esta é apenas uma explicação rápida de como usar inlineforset_factory no Django.

## Models e Forms

Digamos que nosso site lista pedidos, onde temos que adicionar vários produtos em um pedido. Assim, no mais básico nosso *modelo* e *formulário* pode ser algo como isto.

### models.py
```python
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

### forms.py
```python
from django import forms


class OrderForms(forms.ModelForm):
    class Meta:
        model = Order
        exclude = ['data']


class ItemOrderForms(forms.ModelForm):
    class Meta:
        model = ItemOrder
        exclude = ['order']
```

Agora vamos criar a **views.py**, para exibir e renderizar o formulário para adicionar um Pedido com um ou vários Produtos.

### Views
```python
def order(request):
    order_forms = Order()
    item_order_formset = inlineformset_factory(Order, ItemOrder, form=ItemOrderForms, extra=1, can_delete=False,
                                               min_num=1, validate_min=True)

    if request.method == 'POST':
        forms = OrderForms(request.POST, request.FILES, instance=order_forms, prefix='main')
        formset = item_order_formset(request.POST, request.FILES, instance=order_forms, prefix='product')

        if forms.is_valid() and formset.is_valid():
            forms = forms.save(commit=False)
            forms.save()
            formset.save()
            return HttpResponseRedirect('/pedido/')

    else:
        forms = OrderForms(instance=order_forms, prefix='main')
        formset = item_order_formset(instance=order_forms, prefix='product')

    context = {
        'forms': forms,
        'formset': formset,
    }

    return render(request, 'order.html', context)
```

* Instanciamos a classe Order
* Instanciamos o inlineformset_factory
    * *Order*: classe **PAI**.
    * *ItemOrder*: class **FILHA**.
    * *form=ItemOrderForm*: indica qual formulário a ser usado.
    * *extra=1*: quantidade de *forms* a ser apresentado no *html* inicialmente.
    * *can_delete=False*: ignora a opção de deletar o *form*.
    * *min_num=1* e *validate_min=True*: o primeiro parametro seta uma quantidade mínima de *formlário*, o segundo habilita a validação do *formulário*.

## Template

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

No nosso template precisamos de uma função em **JavaScript** para adicionar novos *forms* de produtos a cada vez que for clicado no botão **Add Item**. Essa função irá primeiro descobrir quantos *forms* foram renderizados. Em seguida, vai pegar um novo modelo, processar ele com os dados de contexto e depois acrescentar no **HTML**. E a próxima parte é umas das mais importantes por que está atualizando os detalhes do **management_form** para aumentar o numero de *forms* incluídos a mais.

Documentação
https://docs.djangoproject.com/en/1.9/ref/forms/models/#inlineformset-factory
