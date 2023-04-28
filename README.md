django-pursed
===

A simple wallet django app.

### Creating a New Wallet

A wallet is owned by a user. Should you be using a custom
user model, the wallet should still work properly as it
the wallet points to `settings.AUTH_USER_MODEL`.

```python
from wallet.models import Wallet

# wallets are owned by users.
wallet = user.wallet_set.create()
```
### Implement the user registration and login functionality

  from django.shortcuts import render, redirect
from django.contrib.auth.forms import UserCreationForm
from django.contrib.auth import login, authenticate

def register(request):
    if request.method == 'POST':
        form = UserCreationForm(request.POST)
        if form.is_valid():
            user = form.save()
            is_premium = request.POST.get('is_premium')
            if is_premium:
                wallet = Wallet(user=user, balance=2500.00, is_premium=True)
            else:
                wallet = Wallet(user=user, balance=1000.00)
            wallet.save()
            login(request, user)
            return redirect('dashboard')
    else:
        form = UserCreationForm()
    return render(request, 'registration/register.html', {'form': form})
    
### For sending and receiving money, We'll need to implement views for handling transactions. 

from django.shortcuts import render, redirect
from django.contrib.auth.decorators import login_required
from django.contrib import messages
from .models import Wallet

@login_required
def send_money(request):
    sender_wallet = Wallet.objects.get(user=request.user)
    receiver_id = request.POST.get('receiver_id')
    amount = request.POST.get('amount')
    receiver_wallet = Wallet.objects.get(id=receiver_id)
    if sender_wallet.balance < amount:
        messages.error(request, 'Insufficient funds')
        return redirect('dashboard')
    transaction_charge = amount * 0.02
    sender_wallet.balance -= amount + transaction_charge
    receiver_wallet.balance += amount
    superuser_wallet = Wallet.objects.get(user__is_superuser=True)
    superuser_wallet.balance += transaction_charge
    sender_wallet.save()
    receiver_wallet.save()
    superuser_wallet.save()
    messages.success(request, f'{amount} INR sent to {receiver_wallet.user.username}')
    return redirect('dashboard')


### Despositing a balance to a wallet

```python
from django.db import transaction

with transaction.atomic():
    # We need to lock the wallet first so that we're sure
    # that nobody modifies the wallet at the same time 
    # we're modifying it.
    wallet = Wallet.select_for_update().get(pk=wallet.id)
    wallet.deposit(100)  # amount
```

### Withdrawing a balance from a wallet

```python
from django.db import transaction

with transaction.atomic():
    # We need to lock the wallet first so that we're sure
    # that nobody modifies the wallet at the same time 
    # we're modifying it.
    wallet = Wallet.select_for_update().get(pk=wallet.id)
    wallet.withdraw(100)  # amount
```

### Withdrawing with an insufficient balance

When a user tries to withdraw from a wallet with an amount
greater than its balance, the transaction raises a
`wallet.errors.InsufficientBalance` error.

```python
# wallet.current_balance  # 50

# This raises an wallet.errors.InsufficentBalance.
wallet.withdraw(100)
```

This error inherits from `django.db.IntegrityError` so that
when it is raised, the whole transaction is automatically
rolled-back.

### Transferring between wallets.

One can transfer a values between wallets. It uses
`withdraw` and `deposit` internally. Should the sending
wallet have an insufficient balance,
`wallet.errors.InsufficientBalance` is raised.

```python
with transaction.atomic():
    wallet = Wallet.select_for_update().get(pk=wallet_id)
    transfer_to_wallet = Wallet.select_for_update().get(pk=transfer_to_wallet_id)
    wallet.transfer(transfer_to_wallet, 100)
```

CURRENCY_STORE_FIELD
---

The `CURRENCY_STORE_FIELD` is a django field class that
contains how the fields should be stored. By default,
it uses `django.models.BigIntegerField`. It was chosen that
way for simplicity - just make cents into your smallest 
unit (0.01 -> 1, 1.00 -> 100).

You can change this to decimal by adding this to your
settings.py:

```python
# settings.py
CURRENCY_STORE_FIELD = models.DecimalField(max_digits=10, decimal_places=2)
```

You need to run `./manage.py makemigrations` after that.
