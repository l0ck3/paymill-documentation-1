---
title: "Credit Card Frame Migration Guide"
menu: "Credit Card Frame Migration Guide"
type: "guide"
status: "published"
order: 2
---

In order to comply with PCI DSS 3.0 and be eligible for SAQ-A, you need to use our [Credit Card Frame](/guides/reference/credit-card-frame.html). If you were directly using our bridge before, this guide explains how you can easily switch to the new solution:

1. Use the DSS3 version of our JavaScript bridge.
2. Replace your own fields with the embedded frame.
3. Provide labels, placeholders and error messages.
4. Request a token using the embedded frame.

## Example integration

For the purpose of this guide, let's assume the integration on your site looks something like this:

```
<!-- Payment form -->
<form id="payment-form" action="/store/orders" method="post">

  <!-- Hidden fields that will be submitted to the server -->
  <input type="hidden" class="amount-int" name="amount_int" value="4200" />
  <input type="hidden" class="currency" name="currency" value="EUR" />
  <input type="hidden" class="token" name="token" value="" />

  <div class="form-row">
    <label>Card number</label>
    <input type="text" class="card-number" size="20" />
  </div>

  <div class="form-row">
    <label>CVC</label>
    <input type="text" class="card-cvc" size="4" />
  </div>

  <div class="form-row">
    <label>Name</label>
    <input type="text" class="card-holdername" size="4" />
  </div>

  <div class="form-row">
    <label>Expiry date (MM/YYYY)</label>
    <input type="text" class="card-expiry-month" size="2" />
    <span>/</span>
    <input type="text" class="card-expiry-year" size="4" />
  </div>

  <input type="submit" value="Submit order" />

</form>

<!-- PAYMILL bridge -->
<script src="https://bridge.paymill.com/"></script>

<!-- Payment handling -->
<script>
  // Public PAYMILL API key.
  var PAYMILL_PUBLIC_KEY = '443368146777979106b2ce2042c289a5';

  // Submit handler for payment form.
  var form = $('#payment-form');
  form.submit(function(event) {

    // Don't submit the form yet.
    event.preventDefault();

    // Create a credit card token.
    paymill.createToken({
      amount_int:     form.find('.amount-int').val(),
      currency:       form.find('.currency').val(),
      number:         form.find('.card-number').val(),
      exp_month:      form.find('.card-expiry-month').val(),
      exp_year:       form.find('.card-expiry-year').val(),
      cvc:            form.find('.card-cvc').val(),
      cardholder:     form.find('.cardholder').val()
    }, function(error, result) {
      if (error) {
        // Token could not be created, handle error.
        console.log(error.apierror, error.message);
      } else {
        // Attach token to form and submit it.
        form.find('.token').val(result.token);
        form.submit();
      }
    });

  });
</script>
```

### 1. Use the DSS3 version of our JavaScript bridge

Please use the JavaScript bridge at https://bridge.paymill.com/dss3 instead of https://bridge.paymill.com to use the credit card frame.

```
<script src="https://bridge.paymill.com/dss3"></script>
```

Before we bring the new solution to the standard version of our bridge, we have it available at a separate URL. Later this year we will bring this functionality to the standard version. Once we have updated our standard version, you can move back to the standard URL.

### 2. Replace your own fields with the embedded frame

Since you no longer want to have credit card data directly on your page, simply remove the corresponding fields and instead add a container to load our embedded frame.

```
<form id="payment-form" action="/store/orders" method="POST">

  <!-- Hidden fields that will be submitted to the server -->
  <input type="hidden" class="amount-int" name="amount_int" value="4200" />
  <input type="hidden" class="currency" name="currency" value="EUR" />
  <input type="hidden" class="token" name="token" value="" />

  <div id="credit-card-fields"></div>

  <input type="submit" value="Submit order" />

</form>
```

In order to actually load the credit card frame, simply call `embedFrame` when your page has loaded. Provide the container element, either by specifying its ID (without the `#`) or directly providing the DOM element. jQuery objects are also supported, as shown is this example:

```
paymill.embedFrame($('#credit-card-fields'), options, callback);
```

The `options` parameter will be explained next, so just a quick word about `callback`: You can provide a handler for the outcome of loading the frame. It will be called without a parameter if load was successful. If the frame failed to load, it will receive an `error` parameter.

```
paymill.embedFrame(container, options, function(error) {
  if (error) {
    // Frame could not be loaded, check error object for reason.
    console.log(error.apierror, error.message);
    // Example: "container_not_found"
  } else {
    // Frame was loaded successfully and is ready to be used.
  }
});
```

### 3. Provide labels, placeholders and error messages

By default, the credit card form uses a **default design** but contains **no labels, placeholders or error messages**. This means you have to provide text content before using it on your live site.

<div class="information">
  You can provide labels and placeholders for each field group (note that there are two fields for expiry date) in the `options` parameter of `embedFrame`.
</div>

```
paymill.embedFrame(container, {
  labels: {
    number:     'Credit card',
    cvc:        'CVC',
    cardholder: 'Card holder',
    exp:        'Valid until'
  },
  placeholders: {
    number:     'XXXX XXXX XXXX XXXX',
    cvc:        'XXX',
    cardholder: 'John Doe',
    exp_month:  'MM',
    exp_year:   'YYYY'
  }
}, callback);
```

All text content is optional so you can add or leave out whatever suits your site.

Data is validated automatically when you request a token and you can provide error messages that will automatically appear inside the iframe.

<div class="info">
  Since card holder is optional, it will not be validated.
</div>

You can specify error messages within the `options` parameter of `embedFrame`. Like with labels, you specify one error message per field group.

```
paymill.embedFrame(container, {
  // other options ...
  errors: {
    number:     'Please provide a valid credit card number',
    cvc:        'Please enter the card validation code',
    exp:        'Please provide a valid expiry date'
  }
}, callback);
```

<div class="important">
  To keep things simple, there is only one error message per field group. Make sure you phrase it in a way that covers both missing and invalid data.
</div>

After calling `createTokenViaFrame`, these error messages will be appended to the respective invalid fields.

### 4. Request a token using the embedded frame

A token can be obtained using the method `createTokenViaFrame(options, callback)`. It works just like `createToken` but you don't pass credit card data to it. Like `createToken`, it takes a callback that receives either the result of the operation or a potential error.

```
paymill.createTokenViaFrame({
  amount_int: form.find('.amount-int').val(),
  currency:   form.find('.currency').val()
}, function(error, result) {
  if (error) {
    // Token could not be created, handle error.
    console.log(error.apierror);
  } else {
    // Attach token to form and submit it.
    form.find('.token').val(result.token);
    form.submit();
  }
});
```

Your response handler receives the same `error` and `result` object as before and doesn't have to change.

<div class="info">
  The regular `createToken` still exists and continues to be the appropriate method for creating direct debit tokens. 3-D Secure handling also remains the same, you can still pass the optional `tdsInit` and `tdsCleanup` handlers.
</div>

```
paymill.createTokenViaFrame(transactionData, responseHandler, tdsInit, tdsCleanup);
```

## Updated integration

After having made the changes described above, your integration should now look like this:

```
<!-- Payment form -->
<form id="payment-form" action="/store/orders" method="post">

  <!-- Hidden fields that will be submitted to the server -->
  <input type="hidden" class="amount-int" name="amount_int" value="4200" />
  <input type="hidden" class="currency" name="currency" value="EUR" />
  <input type="hidden" class="token" name="token" value="" />

  <div id="credit-card-fields"></div>

  <input type="submit" value="Submit order" />

</form>

<!-- PAYMILL DSS3 bridge -->
<script src="https://bridge.paymill.com/dss3"></script>

<!-- Payment handling -->
<script>
  // Public PAYMILL API key.
  var PAYMILL_PUBLIC_KEY = '443368146777979106b2ce2042c289a5';

  // Embed the credit card frame.
  paymill.embedFrame($('#credit-card-fields'), {
    labels: {
      number:     'Credit card',
      cvc:        'CVC',
      cardholder: 'Card holder',
      exp:        'Valid until'
    },
    placeholders: {
      number:     'XXXX XXXX XXXX XXXX',
      cvc:        'XXX',
      cardholder: 'John Doe',
      exp_month:  'MM',
      exp_year:   'YYYY'
    },
    errors: {
      number:     'Please provide a valid credit card number',
      cvc:        'Please enter the card validation code',
      exp:        'Please provide a valid expiry date'
    }
  }, function(error) {
    if (error) {
      // Frame could not be loaded, check error object for reason.
      console.log(error.apierror, error.message);
      // Example: "container_not_found"
    } else {
      // Frame was loaded successfully and is ready to be used.
    }
  });

  // Submit handler for payment form.
  var form = $('#payment-form');
  form.submit(function(event) {

    // Don't submit the form yet.
    event.preventDefault();

    paymill.createTokenViaFrame({
      amount_int: form.find('.amount-int').val(),
      currency:   form.find('.currency').val()
    }, function(error, result) {
      if (error) {
        // Token could not be created, handle error.
        console.log(error.apierror, error.message);
      } else {
        // Attach token to form and submit it.
        form.find('.token').val(result.token);
        form.submit();
      }
    });

  });
</script>
```
